# Gateway/firewall — pfSense → OPNsense (203.0.113.1)

## Estado atual

O gateway real (WAN é PPPoE com IP público — não é ambiente isolado) está em **teste de migração do pfSense para o OPNsense**, ambos no mesmo IP `203.0.113.1`, no host `quark`:

- **pfSense** (VMID 111): parado (não apagado), mantido só como rollback de emergência. Será apagado (~set/2026) junto com as VMs `ca` e `bitwarden`.
- **OPNsense** (VMID 102): ativo/em teste desde 2026-08-21. Acesso SSH habilitado, login root desabilitado, DNS do sistema apontando para proton/neutron (.10/.11).

Ao ajudar com rede/firewall/DNS no homelab, considerar que o gateway pode estar num ou noutro conforme a fase da migração — confirmar qual VM está ativa antes de dar comandos de acesso/config, já que o IP é o mesmo nas duas.

O projeto próprio (nunca foi um repositório git) foi removido por redundância — este é o histórico consolidado. Credenciais reais (`.api-credentials`, chave privada do cert da GUI) são mantidas fora deste repositório, nunca commitadas.

## pfSense — gotchas técnicos (referência histórica)

Acesso root: chave SSH de `root` removida (hardening 2026-08-04) — vira root via SSH (porta 22) + `su -` (senha própria, usuário administrativo no grupo `wheel`). **`PermitRootLogin no` não sobrevive a reboot** — o script gerador `/etc/sshd` recalcula `PermitRootLogin yes` sempre que a conta `admin` da GUI está habilitada; tornar isso permanente exigiria editar o script fora do fluxo suportado pelo Netgate (se perderia num upgrade) — decidido não fazer.

Causa de um outage de GUI (2026-08-04, após edição direta de `config.xml`):
1. **Cache de config obsoleto:** `config_read_file(true, true)` no bootstrap nunca compara mtime com cache ativo — editar `config.xml` fora de `write_config()` deixa `/tmp/config.cache` travado no valor antigo. Fix: apagar `/tmp/config.cache` força reparse.
2. **Dois `nginx.conf` distintos:** `/usr/local/etc/nginx/nginx.conf` é o stock do pacote (não usado pela GUI); o real é `/var/etc/nginx-webConfigurator.conf`, gerado por `system_webgui_start()`.
3. **Nunca usar `service nginx restart`** para a GUI — sobe a instância errada (porta 80), conflita com o webConfigurator real. Restart correto: `system_webgui_start()` via PHP, ou `/usr/local/sbin/nginx -c /var/etc/nginx-webConfigurator.conf` (garantindo que nada errado esteja preso na porta 80 antes).
4. **Shell de login é tcsh**, não bash/sh — heredocs/aspas complexas em `ssh host '...'` quebram de forma confusa. Método confiável: gravar script localmente, enviar via stdin (`ssh host sh < script.sh`).

Nesse incidente, a decisão foi corrigir pontualmente (matar o nginx errado, subir o certo) em vez de restaurar a VM inteira do backup Proxmox — o backup era anterior a uma troca de certificado intencional (ligada ao trabalho com a CA interna) que seria perdida, e o restore causaria outage real do firewall de produção. Ver [notas/padroes-operacionais.md](../notas/padroes-operacionais.md) para a regra geral por trás dessa decisão.

## OPNsense — infraestrutura e acesso

- Versão `26.7.2_2` (FreeBSD 15.1-RELEASE-p2), sem atualizações pendentes, sem plugins de terceiros.
- Interfaces: `vtnet0` = LAN, `203.0.113.1/24` + IPv6 `2001:db8:a515::/64` (delegado via PD). `vtnet1` = WAN físico sem IP direto — WAN real é `pppoe1` (PPPoE), IP público dinâmico (sem IP fixo do ISP — não registrar o valor aqui) + IPv6 delegado via PD (prefixo também dinâmico).
- SSH: habilitado (System > Settings > Administration, porta 22). Usuário administrativo (uid 2001, grupo `admins`/`wheel`) criado **pela GUI** com uma chave em Authorized keys — tentativa inicial via PHP nativo (`local_user_set`/`local_group_set`) rodando como root por SSH travou (sem corromper nada, `write_config()` nunca foi alcançado); GUI funcionou de primeira. Login SSH de `root` desabilitado, confirmado "Permission denied". Hostname: `OPNsense.particula.local`.
- API: key/secret mantidos fora deste repositório. O usuário `api` tem privilégio nativo `user-config-readonly` (`ApiControllerBase::throwReadOnly()`) — bloqueia qualquer escrita via API independente de outros privilégios; já é "least-privilege" para escrita, confirmado na prática (tentativa de criar usuário via API foi recusada). Páginas legadas (`.php`, ex. `system_general.php`) não aceitam API key, só sessão de login (usuário/senha + CSRF).
- **Usuário de API com escrita restrita** (`api-fw-write`, 2026-09-12): criado à parte do `api` (que permanece read-only de propósito) pra permitir automação pontual de regras/aliases de firewall. Grupo `api-firewall-write`, privilégios necessários (nomes de GUI, mapeados em `ACL.xml` — não é o mesmo privilégio de edição de página comum):
  - **Firewall: Rules [new]** (`page-filter-api`) — cobre `/api/firewall/filter/*`.
  - **Firewall: Alias: Edit** (`page-firewall-alias-edit`) e **Firewall: Aliases** (`page-firewall-aliases`) — cobrem `/api/firewall/alias/*`.
  - `page-firewall-rules-edit` (privilégio de GUI) não é suficiente sozinho pra API — dá 403 mesmo em leitura se faltar o `page-filter-api`.
  - Usuário/grupo criados pela GUI (criação via PHP nativo já travou antes nesse OPNsense). Key/secret gerados na própria página do usuário (seção "API keys"), usados uma vez e descartados — não ficam em arquivo/memória.
  - Fluxo de mudança testado: criar/editar alias (`addItem`/`delItem`) → `alias/reconfigure` → criar/editar regra (`addRule`) → **`filter/apply`** (só esse último efetivamente recarrega o pf; sem ele a regra fica salva mas inativa). `filter/apply` é tratado como ação de alto impacto (gate do Claude Code auto mode pede confirmação explícita antes de rodar).
- Senha de root existe e foi usada uma vez (via `su` a partir do usuário administrativo, pra remover um backup de config pós-alteração) — não fica salva em nenhum arquivo/memória.

## OPNsense — achados de auditoria inicial (2026-08-21/23, somente leitura)

- Firmware/pacotes base atualizados, sem plugins de terceiros. Sem NAT 1:1/port-forward — só NAT de saída automático. GUI (80/443) só liberada para LAN via regra anti-lockout. Regras de firewall são as padrão de instalação + "allow LAN to any" (v4/v6), nada de custom. syslog local ativo (retenção 31 dias), NTP rodando.
- Certificado da GUI: leaf `CN=opnsense.particula.local` emitido pela CA interna (`pki_int`/role `particula-local`), SAN `DNS:opnsense.particula.local`+`IP:203.0.113.1`, válido 22/ago/2026→22/ago/2027. Authorities `Particula CA`/`Particula Intermediate CA` importadas em System > Trust > Authorities; cert+chave em System > Trust > Certificates e selecionado em Administration. `openssl s_client` confirma cadeia servida e `Verification: OK`. Nota: OpenBao estava sealed no momento (provável reboot da VM) — unseal com 2 das 3 chaves Shamir antes de emitir o cert. Cadeia serve 4 certs (leaf+intermediária duplicada+raiz) — redundante mas inofensivo, cosmético reeditar pra só o leaf.
- Resolver DNS (Unbound): habilitado via GUI (System > Settings > General), `resolv.conf` confirmado com `nameserver 203.0.113.10`/`203.0.113.11` (proton/neutron), domínio `particula.local`.
- Log de firewall (API `/api/diagnostics/firewall/log`): tráfego bloqueado no WAN é majoritariamente ruído normal de scanners de internet. **Atualizado 2026-09-12:** análise do syslog identificou exceção — `187.75.226.110` gerando scan/flood sustentado contra a porta 51820 (WireGuard), constante desde pelo menos 01/09 (12+ dias, 80k-175k blocks/dia, mesma porta origem/destino). Bloqueado explicitamente via alias `blocklist_wireguard_scanner` + regra WAN (`quick block in`, log ativo), aplicada e validada (sem lockout de LAN/GUI/SSH).
- **Atualizado 2026-10-02:** análise de 24h do syslog (~212k eventos; ~150k bloqueios WAN): um único scanner = 55% dos bloqueios, já coberto pelo alias. Adicionados mais 5 scanners UDP 51820 ao `blocklist_wireguard_scanner` via API (alias com 16 IPs). Bloqueios na LAN por pacotes FIN/ACK após expiração de estado são benignos. Tráfego P2P de saída não é atribuível a host pelo log (NAT); usar Firewall → Diagnostics → States.
- Baseline de tráfego esperado: clientes da LAN com NetBird/WireGuard geram tráfego de saída para UDP/51820, TCP/33073, TCP/10000, UDP/3478, TCP/33080 — esperado, não deve gerar alerta. Não confundir com tentativas **bloqueadas** de origem externa contra a porta WAN 51820.
- `PasswordAuthentication yes` no sshd, mas sem regra de liberação na WAN (só DHCPv6 client) — sem exposição à internet, mitigado pelo lockout nativo de força bruta (`sshlockout`). Não corrigido: API é `user-config-readonly`, sem `NOPASSWD` — precisa senha de sudo interativa ou GUI.
- 2FA de admin: item em aberto, sem decisão do usuário ainda.
- pfSense antigo não tinha nenhuma regra customizada — nada a portar, se quiser regras além do padrão seria criar do zero.

## syslog remoto do gateway

O gateway envia log para `syslog.particula.local` — ver [servicos/syslog.md](../servicos/syslog.md). Pegadinhas do lado OPNsense: campo Level vem só com "alert" marcado (precisa habilitar "info"+); há dois "programs" parecidos e só `filterlog` é o log de pacotes real (`firewall` é outra coisa); Save do diálogo não reconfigura o serviço, precisa clicar Restart.
