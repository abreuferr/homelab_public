# quark.particula.local — host Proxmox

Nó Proxmox VE standalone (sem cluster/corosync), 4 vCPUs, 15GiB RAM. É o hospedeiro único de todo o homelab (~15 VMs/CTs, incluindo DNS proton/neutron). Falha da fonte de alimentação é ponto único de falha que derruba tudo de uma vez.

## Acesso

- SSH: acesso administrativo, porta **59022** (não 22).
- API Proxmox: token `root@pam!root`, endpoint `https://quark.particula.local:8006/api2/json/...`, header `Authorization: PVEAPIToken=root@pam!root=<secret>`. Certificado emitido pela CA interna — `curl` precisa de `-k` fora de contexto que já confie na CA. O token precisa de ACL explícita em Datacenter → Permissions (path `/`); sem isso a API autentica mas retorna listas vazias.

## Inventário (revalidado 2026-09-09)

**VMs (qm):**

| VMID | Nome | Status |
|---|---|---|
| 102 | opnsense | rodando — gateway ativo |
| 111 | pfsense | parada — rollback do teste OPNsense; mantida parada, ainda não apagada |
| 9000/9001 | templates debian12/13 | parados |

VM 101 (`ca`) e VM 113 (`bitwarden`) já foram apagadas — não existem mais no `qm list`.

**Containers LXC (pct):**

| VMID | Nome | IP | Registro DNS |
|---|---|---|---|
| 100 | netbird | 203.0.113.13 | netbird.particula.local |
| 103 | adguard | 203.0.113.22 | adguard.particula.local |
| 104 | pihole | 203.0.113.12 | pihole.particula.local |
| 105 | openbao (CA de produção) | 203.0.113.15 | openbao.particula.local |
| 106 | qbittorrent | 203.0.113.17 | qbittorrent.particula.local |
| 107 | freshrss | 203.0.113.18 | freshrss.particula.local |
| 110 | vaultwarden | 203.0.113.19 | vaultwarden.particula.local |
| 114 | immich | 203.0.113.25 | immich.particula.local |
| 116 | proton (DNS primário) | 203.0.113.10 | proton.particula.local |
| 117 | neutron (DNS secundário) | 203.0.113.11 | neutron.particula.local |
| 118 | syslog | 203.0.113.16 | syslog.particula.local |
| 119 | phpipam | 203.0.113.23 | phpipam.particula.local |

IPs revalidados em 2026-09-22. Todos os 12 LXCs batem 1:1 com um registro DNS — nenhum host órfão. `vaultwarden` (Alpine) é o único que não aceita SSH direto na porta 22; acesso só via `pct exec`/`pct enter`.

Inventário decai rápido — sempre revalidar `qm list`/`pct list` ao vivo antes de agir, em vez de confiar num levantamento antigo.

**Storage:** `local` (dir, 94GB) e `local-lvm` (thin LVM, 338GB, ~69% usado — onde ficam os discos de VM/CT, monitorar).

`pct push <vmid> <origem> <destino> --perms 600 --user 0 --group 0` funciona bem para colocar arquivos dentro de um LXC sem SSH direto. `pct exec <vmid> -- <comando>` também funciona sem precisar de chave/SSH, roda como root via Proxmox — útil quando um CT ainda não tem acesso SSH configurado.

## Backup diário (vzdump)

`/usr/local/sbin/proxmox-backup.sh` roda via crontab do root, `0 2 * * *`. Faz `vzdump --all 1 --mode snapshot --compress zstd` com prioridade baixa (`nice -n 19 ionice -c3`) de **todas** as VMs/LXCs do nó. Destino: `/mnt/backup` (disco dedicado `/dev/sda1`, ext4, 220G), separado do `local-lvm` mas **no mesmo host físico** — sem cópia off-site. Retenção: só os **2 backups mais recentes** (`RETENTION_KEEP=2`), script apaga os mais antigos. Log em `/var/log/proxmox-backup.log`.

Guardas de segurança: aborta se `/mnt/backup` não estiver montado ou tiver menos de 20GB livres (`MIN_FREE_GB=20`); usa `flock -n` em `/var/lock/proxmox-backup.lock` para evitar execuções concorrentes. Cada backup gera `sha256sum` por arquivo, verificado pelo `/usr/local/sbin/proxmox-restore.sh` antes de restaurar.

Antes de qualquer operação arriscada numa VM/CT, checar a data do backup mais recente em `/var/log/proxmox-backup.log` — só há ~2 dias de retenção, não presumir que "existe backup" sem checar a idade.

**Bug encontrado e investigado 2026-09-05: causa raiz não é o checksum travando, é o `local-lvm` cheio.** `/mnt/backup` estava 97% cheio (202G/220G) com 4 dias acumulados (`09-02` a `09-05`) em vez de 2 (`RETENTION_KEEP=2`). Cadeia completa:

1. Thin pool `pve/data` (storage `local-lvm`) estava em 81.58% de uso, inflado por dezenas de snapshots LVM avulsos nunca removidos (`snap_vm-*_Update_*`, o mais antigo de 2026-02-19 — provavelmente de algum script de pré-update que não limpa depois). Vários discos individuais quase cheios: vm-100 (99.47%), vm-106 (99.59%), vm-116 (99.64%), vm-117 (99.62%), vm-103 (96.22%), vm-111 (96.31%), vm-113 (96.24%).
2. Nas execuções de `09-04` e `09-05`, o pool cruzou o limiar de segurança do LVM e os 12 backups de LXC (100,103,104,105,106,107,110,114,116,117,118,119) falharam com `ERROR: ... lvcreate snapshot ... Cannot create new thin volume, free space in thin pool pve/data reached threshold` (confirmado nos `.log` individuais de cada CT). VMs QEMU não usam esse mecanismo e continuaram OK.
3. `vzdump` retornou exit code não-zero (12 jobs falhos). `proxmox-backup.sh` roda `vzdump ... | tee -a "$LOG_FILE"` sob `set -euo pipefail` — o erro propagou pelo pipe e matou o script logo após a linha do vzdump, **antes** da etapa de checksum (4b) e da limpeza de retenção (5). Não é o checksum travando; ele nunca chega a rodar.

Ação tomada: removidos manualmente `proxmox-2026-09-02` e `proxmox-2026-09-03` de `/mnt/backup` (liberou ~129G, disco caiu de 97% para 36%). **Não resolvido:** o `local-lvm` continua no limite — enquanto isso não for tratado (limpar snapshots antigos e/ou expandir o pool), qualquer backup que rode com o pool acima do limiar volta a falhar do mesmo jeito e o disco de backup enche de novo em ~2 dias. Isso é mais urgente que o disco de backup: é o storage de produção das VMs/CTs.

**Bug do script corrigido em 2026-09-05:** `set -e`+`pipefail` fazia qualquer falha parcial do vzdump abortar o script antes do checksum e da retenção, deixando lixo acumular sem alertar. Corrigido em `/usr/local/sbin/proxmox-backup.sh` (backup em `proxmox-backup.sh.bak-20260905`): o exit code do vzdump agora é capturado à parte via `PIPESTATUS` (`... | tee -a "$LOG_FILE" || VZDUMP_EXIT=${PIPESTATUS[0]}`), o script loga um aviso e **sempre** prossegue para gerar checksums e aplicar a retenção, e só ao final propaga o exit code real do vzdump (`exit "$VZDUMP_EXIT"`) para que cron/e-mail ainda sinalizem a falha. Validado com `bash -n` (sintaxe) e um teste isolado do padrão `set -e`+`PIPESTATUS` fora do host; não foi rodado um vzdump real de teste para não gerar snapshots desnecessários nas VMs/CTs em produção — a validação de ponta a ponta acontece na próxima execução agendada (`0 2 * * *`).

**Origem dos snapshots `snap_vm-*_Update_*`:** vêm da rotina de atualização de VMs/LXCs que o usuário usa, [BassT23/Proxmox](https://github.com/BassT23/Proxmox), instalada em `/etc/ultimate-updater/` (config em `update.conf`, logs em `/var/log/ultimate-updater*.log`) — cria um snapshot LVM antes de cada update. O README não documenta isso, mas a ferramenta **tem retenção própria**: `KEEP_SNAPSHOTS` no `update.conf` (estava em `"3"`). Não era ausência de limpeza — 3 cópias por VM/CT é que era generoso demais para a capacidade do `local-lvm` deste host, especialmente nos discos grandes (106, 111, 113: 20-40G cada). Isso, mais o fato de a VM 111 (pfsense, parada) ter ficado travada em 3 snapshots antigos de fevereiro sem nunca receber updates novos que os reciclassem, inflou o pool até 81.58%.

Ações tomadas em 2026-09-05:
1. Removidos manualmente 21 snapshots antigos (mantido só o mais recente por VM/CT; os 3 da VM 111 removidos por completo, já que a VM está para ser apagada — ver inventário). `local-lvm` caiu de 81.58% para 56.77%. `snap_vm-101-disk-0_clean` (fora do padrão `_Update_`, provavelmente manual) não foi tocado.
2. `KEEP_SNAPSHOTS` reduzido de `"3"` para `"1"` em `/etc/ultimate-updater/update.conf` (backup em `update.conf.bak-20260905`) — a partir da próxima rodada de update, a própria ferramenta mantém só o snapshot mais recente por VM/CT. Optou-se por usar o mecanismo nativo em vez de criar cron/script próprio de poda, para não duplicar lógica.

**Efeito colateral do passo 1 (corrigido no mesmo dia):** apagar os 21 snapshots com `lvremove` bruto (em vez de `pct/qm delsnapshot`) removeu o volume LVM mas não atualizou a metadata de snapshot do Proxmox (seções `[Update_*]` nos arquivos `/etc/pve/lxc/<id>.conf` e `/etc/pve/qemu-server/<id>.conf`, mais o ponteiro `parent:` de quem vinha depois na cadeia). Descoberto quando uma atualização real no LXC 100 (via ultimate-updater) tentou podar o snapshot antigo, esbarrou numa seção fantasma, e travou o CT (`lock: snapshot-delete`) — visível em `UPID:...vzdelsnapshot:100:... lvremove snapshot ... error: Failed to find logical volume`. Um CT/VM travado bloqueia inclusive o `vzdump`, então o mesmo aconteceria no backup das 02h se não corrigido.

Correção aplicada nos 11 VMs/CTs afetados (100,103,104,106,107,110,111,113,116,117,119), com backup de cada `.conf` antes:
- Removidas as seções `[Update_*]` cujo volume LVM já não existia (metadata fantasma).
- Removida a linha `parent:` pendurada no snapshot real remanescente (ou na config principal, no caso da VM 111, que ficou sem nenhum snapshot).
- CT 100 especificamente: `pct unlock 100` + `pct delsnapshot 100 Update_20260830_120341` (esse tinha volume real, apagado pela via oficial) — e uma segunda passada pra limpar um `parent:` que o próprio Proxmox deixou pendurado ao reencadear a árvore durante esse delsnapshot.
- Verificado ao final: `pct/qm listsnapshot` limpo (só o snapshot real de cada um) e `pct/qm status` normal em todos os 11.

**Lição:** nunca usar `lvremove` direto para apagar snapshot de VM/CT do Proxmox — sempre `pct delsnapshot`/`qm delsnapshot`, que cuida da metadata e do reencadeamento da árvore junto com a remoção do volume.

**Backup off-site:** ainda não implementado. Avaliado `rclone copy` (não `sync`) para Backblaze B2 (~US$6/TB/mês) com `rclone crypt` por cima (dados sensíveis: CA interna, OpenBao, Bitwarden, Immich). Como o vzdump é full-backup diário (sem incremental), reter mais que poucos dias off-site fica caro — migrar para Proxmox Backup Server (incremental + dedupe) resolveria isso, mas é projeto maior (exige datastore próprio, de preferência fora do `quark`). Tamanho de referência: ~112GB nos 2 ciclos de retenção atuais.

## Segurança e hardening

- SSH hardened: `PermitRootLogin no`, `PasswordAuthentication no`, só pubkey, porta 59022, `MaxAuthTries 3`, `MaxSessions 2`, `ClientAliveCountMax 2`, `TCPKeepAlive no`, `LogLevel VERBOSE` — mesmo padrão aplicado em todo o homelab.
- Firewall do Proxmox habilitado (`/etc/pve/nodes/quark/host.fw`), com regras `ACCEPT` de origem `203.0.113.0/24` para 8006/3128/8008 e `ACCEPT` para 59022/icmp.
- Certificado do painel web (8006) emitido pela CA interna (`Particula Intermediate CA`), não self-signed genérico.
- `/root/.ssh/authorized_keys` é um symlink para `/etc/pve/priv/authorized_keys` (mecanismo interno do Proxmox para SSH entre nós de cluster, presente mesmo em standalone) — a chave RSA ali é do próprio nó, não remover.
- Sem `fail2ban` (SSH já mitiga por ser só pubkey), sem `unattended-upgrades`, AppArmor com só 30/129 perfis em enforce (majoritariamente LXC).
- Processo `ProxMenux` (dashboard comunitário, porta 8008, roda como root) — legítimo, mas por ser projeto de terceiros e correr como root, vale manter atualizado e considerar restringir a porta via firewall.

**Achado 2026-09-22 (varredura de segurança):** `policy_in` do `host.fw` está como `ACCEPT`, não `DROP`. Isso torna as regras de origem `203.0.113.0/24` acima **cosméticas** — qualquer pacote não coberto por elas é aceito de qualquer origem. O bloqueio real hoje vem só do que está explicitamente em `DROP` no `cluster.fw` (SMB/NetBIOS/UPnP) e, presumivelmente, do isolamento de rede a montante. **Ainda não corrigido** — mudar `policy_in` para `DROP` exige confirmação antes de aplicar (risco de lockout do próprio SSH se a regra ficar incompleta).

Outros achados da mesma varredura:
- `rpcbind`/`nfs-common` instalados e escutando publicamente sem uso de NFS identificado — candidato a desabilitar.
- Serviço de auto-update de terceiros (bind `0.0.0.0`, roda como root) exige autenticação — não é acesso aberto, apesar do bind amplo.
- `iperf3` foi removido do host (decisão do usuário) — não escuta mais.
- 2 reboots seguidos sem causa identificável — journal não reteve a janela (poucos boots armazenados, com lacunas de semanas).

## Memória e swap (zram)

Configurado em 2026-08-23: `/dev/zram0` (prioridade 100) na frente do swap em disco `/dev/dm-0` (prioridade -1), para reduzir escrita em disco sob pressão de memória.

**Checado em 2026-08-26** via `/proc/swaps` e `free -h`:

```
/dev/dm-0   (disco)  8388604 KB total   Used: 0          Priority: -1
/dev/zram0  (zram)   4876264 KB total   Used: 723576 KB  Priority: 100

Swap: 12Gi total, 706Mi usado, 11Gi livre
Mem:  15Gi total, 8.3Gi usado, 476Mi livre, 7.2Gi buff/cache
```

zram está absorvendo 100% do swap ativo; disco em uso zero (não só "diminuindo", já não tem nenhuma página lá). Prioridades corretas — kernel só vai ao disco se o zram encher. `zramctl` não está instalado no host (comando não encontrado); os dados vieram só de `/proc/swaps`.

## Notas operacionais

- **Achado 2026-08-23:** ventoinha da fonte de alimentação com ruído alto — investigado remotamente (temperaturas e carga normais, nada no dmesg indicando throttling); confirmado por inspeção física do usuário como **problema mecânico** (desgaste de rolamento), não térmico/elétrico. Prioridade: trocar a fonte (ou só a ventoinha) antes que falhe de vez.

## Monitor de temperatura (criado 2026-09-05)

Não havia telemetria de software para temperatura antes disso (só inspeção física). Como o homelab não tem Zabbix/Prometheus/Grafana, o postfix do host é `loopback-only` (sem `relayhost`, não manda e-mail pra fora) e o CT syslog central só aceita origem do OPNsense (ver [../servicos/syslog.md](../servicos/syslog.md)), a notificação escolhida foi push via [ntfy.sh](https://ntfy.sh) — sem novo serviço rodando no host, só `curl` de saída.

- `/usr/local/sbin/temp-monitor.sh` (root:root, 750), agendado no crontab do root a cada 5 min (`*/5 * * * *`; crontab antigo salvo em `/root/crontab.root.bak-20260905`).
- Lê `sensors -j` (lm-sensors) via `jq`, e só avalia sensores que o próprio driver expõe com um limiar `*_max` (hoje: `Package id 0` + `Core 0-3` do `coretemp`, limiar 80°C; `Composite` do NVMe, limiar 79.85°C) — não inventa limiar fixo por fora do hardware. Sensores sem `*_max` (ex.: `acpitz`, `Sensor 1-6` do NVMe) não são checados.
- Histerese de 5°C: dispara alerta ao cruzar o limiar (`>=`), só dispara "normalizado" ao cair 5°C abaixo dele — evita flapping/spam a cada 5 min enquanto a temperatura oscila perto do limiar. Estado por sensor em `/var/local/temp-monitor/<chip>_<feature>` (arquivo presente = alerta ativo).
- Tópico ntfy é segredo de baixo risco (quem souber o nome pode publicar/ler no tópico) — valor real só no script no host, não neste repo. Testado ponta a ponta em 2026-09-05 (alerta e recuperação, via cópia temporária do script com limiar forçado baixo).
- Ao investigar "VM não iniciou automaticamente", sempre comparar uptime da VM com uptime do node e cruzar com o log de tasks da API antes de concluir falha de auto-start — pode ser desligamento/religamento manual posterior ao boot real.
- Uso de ntfy generalizado em 2026-09-05 pra todo alerta do homelab (não só temperatura) — tópico dedicado próprio, script reutilizável `/usr/local/sbin/ntfy-alert.sh`, documentado em [../servicos/ntfy.md](../servicos/ntfy.md).

## Alertas de falha no backup via ntfy (adicionado em 2026-09-05)

`/usr/local/sbin/proxmox-backup.sh` passou a chamar `/usr/local/sbin/ntfy-alert.sh` (prioridade `high`) em todo ponto de falha: lock já em uso, `/mnt/backup` não montado, espaço insuficiente, `vzdump` com exit code != 0. Substitui a expectativa antiga (comentário do próprio script) de que "cron/e-mail" sinalizaria a falha — nunca sinalizava, por causa do postfix loopback-only (mesma razão do monitor de temperatura acima). Backup do script anterior em `/usr/local/sbin/proxmox-backup.sh.bak-20260905113716` no host. Validado com `bash -n` nos dois lados (local e no host) e teste direto do `ntfy-alert.sh` (HTTP 200, notificação recebida); não foi rodado um `vzdump` real de teste, mesma razão do bugfix anterior — validação de ponta a ponta acontece na próxima execução agendada (`0 2 * * *`). Detalhes do script e dos tópicos em [../servicos/ntfy.md](../servicos/ntfy.md).

## Desligamento em massa para manutenção (criado 2026-09-23)

`/usr/local/sbin/shutdown-all-guests.sh` (root:root, 750). Desliga todas as VMs e LXCs rodando no host, para manutenção do host físico (`sudo /usr/local/sbin/shutdown-all-guests.sh`).

Fica em `sbin`, não `bin`: é script de administração (exige root), mesma convenção dos scripts de backup/restore e do `ntfy-alert.sh`.

- Lista VMs/LXCs em execução via `qm list`/`pct list`, dispara `qm shutdown`/`pct shutdown` em paralelo para todos com timeout de 180s e força o desligamento (`--forceStop`/`--force`) se algum não parar a tempo. Ao final, mostra `qm list`/`pct list` para confirmar.
- Exige root (checa `id -u`).
- Inclui o gateway (OPNsense) e os LXCs de DNS — rodar derruba internet e DNS da LAN até a manutenção terminar e os guests voltarem a subir manualmente.
- Validado só com `bash -n` (sintaxe); não foi rodado de verdade, pra não derrubar produção sem necessidade real de manutenção.
