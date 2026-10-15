# PKI interna e OpenBao (CA de particula.local)

## Estado atual

`openbao.particula.local` (203.0.113.15, LXC 105 em quark) é a **CA de produção** desde 2026-08-25. A VM antiga `ca.particula.local` (203.0.113.14, VMID 101) foi apagada (`qm destroy --purge`) em 2026-09-09, antes do prazo previsto de 25/09 (usuário pediu antecipação; era só fallback, sem uso ativo). Qualquer emissão/renovação de certificado do homelab sai do LXC.

**Gotcha operacional permanente:** sem auto-unseal por design — todo restart/reboot do `openbao.service` exige desselar manualmente com 2 das 3 chaves Shamir. Só quem tem acesso ao cofre de senhas (Vaultwarden) consegue religar o cofre depois de um restart.

**TLS do CLI `bao` no próprio LXC (corrigido em 2026-09-10):** `BAO_ADDR` fixado em `/etc/profile.d/bao-addr.sh` (`https://203.0.113.15:8200`, já que o cert só tem SAN pra esse IP, não `127.0.0.1`), e a cadeia da CA (`Particula CA` + `Particula Intermediate CA`) instalada no trust store do sistema via `update-ca-certificates` — não precisa mais de `BAO_SKIP_VERIFY` pra rodar `bao status`/`bao operator unseal` localmente. Ambos fazem parte do rootfs do LXC 105, então sobrevivem a `pct restore` de backups feitos depois dessa data.

## Migração VM → LXC (2026-08-25)

Primeira tentativa via script community-scripts foi **descartada por achado de segurança**: o script grava root token + unseal key em texto puro (`/etc/openbao/openbao.env`), faz auto-unseal no boot, e inicializa o cofre com Shamir 1/1 (sem quorum real) — inaceitável para uma CA de produção. CT reconstruído manualmente (Debian 13, OpenBao 2.6.2 via `.deb` oficial, checksum verificado, sem auto-unseal). Dados reais migrados via `tar` do `/opt/openbao/data` da VM antiga (hop único, `shred -u` em cada lado), reaproveitando as chaves de unseal **originais** da VM — não as geradas na reinstalação, que morreram junto com o cofre de teste que substituíram.

Padrão a repetir em qualquer migração de serviço que emite certificado para si mesmo: emitir o certificado TLS do novo host **antes** de migrar os dados, com a CA antiga ainda viva para assinar.

**Passo obrigatório em qualquer reconstrução do zero do LXC do OpenBao** (rebuild novo, não `pct restore` de backup — esse já carrega os arquivos abaixo no rootfs): refazer os dois fixes de TLS descritos acima (`/etc/profile.d/bao-addr.sh` com `BAO_ADDR` e a cadeia da CA em `update-ca-certificates`) **antes** do primeiro unseal do LXC novo. Sem isso, o unseal cai de novo nos gotchas originais (`BAO_ADDR` errado + CA desconhecida) e exige o contorno `BAO_SKIP_VERIFY=true`.

## Estrutura da PKI

Cadeia: `Particula CA` (raiz, self-signed, engine `pki_root`, 2026–2036) assina `Particula Intermediate CA` (engine `pki_int`, 2026–2031). A intermediária tem a role `particula-local` (`allowed_domains=[particula.local]`, `allow_any_name=false`, `enforce_hostnames=true`) usada para emitir certificados de serviços do domínio. A cadeia **não é totalmente uniforme**: pelo menos um leaf antigo (bitwarden, emitido antes da intermediária existir) foi assinado direto pela raiz — boa prática daqui em diante é a raiz só assinar a intermediária, nunca emitir leaf de serviço diretamente.

**Role `particula-vpn`** (existia com `allow_any_name=true`, ignorando `allowed_domains`) foi confirmada sem uso real (não emitia nenhum certificado de VPN) e **deletada definitivamente** em 2026-08-04 — não existe servidor de VPN no homelab.

## Fluxo padrão de emissão de certificado

1. Emitir na CA (`bao write pki_int/issue/particula-local common_name=<host>.particula.local ip_sans=<ip> ttl=8760h`), usando token escopado de curta duração (policy só com as capabilities necessárias, TTL curto, use-limit baixo) em vez do root token quando possível.
2. Aplicar no serviço destino, sobrescrevendo os arquivos certificado/chave (backup dos anteriores sempre feito antes).
3. Reiniciar o serviço e validar com `openssl s_client`/`openssl verify -CAfile root-ca.pem -untrusted int-ca.pem`.
4. Apagar cópias intermediárias em qualquer hop (`shred -u`) — nunca deixar chave privada persistida fora do destino final.

**Por que o fix de TLS persistente (`BAO_ADDR` + trust store) só existe no LXC 105:** outros hosts (ex. phpIPAM, ver [../servicos/phpipam.md](../servicos/phpipam.md)) já bateram no mesmo erro de `127.0.0.1`/CA desconhecida durante a emissão do próprio certificado, mas foi um workaround pontual (`VAULT_CACERT` apontado na hora pra raiz baixada do endpoint público) rodado de onde quer que o operador estivesse — nenhum desses hosts roda `bao`/`vault` CLI depois disso, então não há nada residente pra tornar permanente. O LXC 105 é diferente porque é o único que roda o CLI **repetidamente** (todo `bao operator unseal` pós-reboot), daí valer instalar o fix ali.

Endpoints públicos do OpenBao úteis sem autenticação: `GET /v1/pki_root/ca/pem`, `GET /v1/pki_int/ca/pem`, `GET /v1/<mount>/crl`.

**CRL:** `auto_rebuild=true` habilitado em `pki_root/config/crl` e `pki_int/config/crl` (estava `false` com `expiry=72h`, CRL ficou vencida ~5 meses até ser corrigido em 2026-08-04). Rotação exige capability `update` no path, não `read`.

**URLs de AIA/CRL (corrigidas em 2026-10-15):** os certificados emitidos até então traziam `http://ca.particula.local:8200/...` (AIA e CRL). Estavam quebradas: `ca.particula.local` não resolve mais (VM apagada em 2026-09-09) e `http://` não funciona porque o listener só aceita TLS. Revogar certificado não tinha efeito prático. A CRL em si sempre esteve correta.

- `pki_root/config/urls` e `pki_int/config/urls` agora usam `https://openbao.particula.local:8200/v1/<mount>/ca` e `.../crl`. Usar o nome (não o IP): as URLs ficam anos dentro dos certificados.
- Vale só para certificados emitidos depois da mudança. O leaf do próprio OpenBao foi reemitido (`pki_int/issue/particula-local`, TTL 8760h, vence 2027-10-15) e trocado em `/opt/openbao/tls/` (600, `openbao:openbao`) com `systemctl reload openbao` — o reload não sela o cofre. Validado: `openssl verify`, conexão externa com `-verify_return_error` e CRL com HTTP 200 pelo nome.
- **Pendente:** os outros leafs (bitwarden/Vaultwarden, phpIPAM, painel do Proxmox etc.) mantêm a URL antiga até a próxima renovação. A `Particula Intermediate CA` só troca ao ser reassinada pela raiz (renovação em 2031, ou antes se necessário).
- Gotcha: o campo `ca_chain` devolvido pela emissão trouxe só a intermediária, sem a raiz. Para validar, buscar a raiz em `GET /v1/pki_root/ca/pem`.

## Audit device (habilitado em 2026-10-15)

- O OpenBao 2.6 **não aceita** `bao audit enable` pela API (erro 400: "use declarative, config-based audit device management"). O audit device é declarado no `/etc/openbao/openbao.hcl`, bloco `audit "file" "file-audit"` com `file_path=/var/log/openbao/audit.log` e `mode=0600`. Aplicado com `systemctl reload openbao` (SIGHUP), sem selar o cofre.
- `/var/log/openbao/` é `openbao:openbao` 750. Rotação em `/etc/logrotate.d/openbao`: semanal, 8 cópias, compactada, com `reload` do OpenBao no `postrotate` para reabrir o arquivo. Testado com `logrotate -f`: o novo arquivo voltou a receber registros e o cofre seguiu desselado.
- O log guarda tokens e valores sensíveis com hash (HMAC), não em texto puro (checado em 2026-10-15).
- **Risco:** com um único audit device, se a gravação falhar (disco cheio, permissão) o OpenBao recusa todas as requisições. O disco do LXC tem 2,7 GiB livres (26% usado). Vigiar o espaço em `/var/log/openbao`.
- **Encaminhamento ao syslog (2026-10-15):** `rsyslog` instalado no LXC 105 (não existia). `/etc/rsyslog.d/30-openbao-audit.conf` lê `audit.log` (`imfile`, tag `openbao-audit:`) e envia por **TCP** a `203.0.113.16:514`, com fila em disco (100 MiB) para não perder registros se o syslog cair. Só o audit log é encaminhado, nenhuma outra mensagem do LXC. Chega em `/var/log/remote/openbao/messages.log` no CT 118. Ver [../servicos/syslog.md](../servicos/syslog.md).
- **Pendente:** alerta sobre falha de gravação do audit log (não existe canal de alerta no homelab).

## Revisão de segurança (2026-10-15) — pendências

Revisão somente leitura, sem token. Itens abertos, por prioridade:

1. **Média:** reemitir os demais leafs com as URLs novas, na renovação de cada um (ver acima). Inventário em 2026-10-15: 14 leafs ainda com URL antiga; o mais próximo vence em 2027-03-12 (`bitwarden`, `ca`). `ca.particula.local` (VM apagada) e, a confirmar, `bitwarden` e `pfsense` são certificados de hosts que não existem mais; candidatos a revogação.
2. ~~Audit device~~ **Resolvido em 2026-10-15** (ver seção abaixo).
3. **Média:** policy `pki-operator` criada e testada em 2026-10-15 (emite em `pki_int/issue/particula-local`, lê/lista/revoga certificados, lê config e `sys/audit`; nega escrita em config, policies e mounts). **Ainda não atribuída:** o `userpass` do usuário `cferreira` segue com a policy `admin` (`path "*"` com `sudo`), root-equivalente, com TTL de ~1h. Trocar para `pki-operator` no uso diário e deixar `admin` só para o root token do Vaultwarden. A role `particula-local` tem `ext_key_usage` vazio; definir `ServerAuth`.
4. **Média:** restringir a porta 8200 à `203.0.113.0/24` no firewall do Proxmox; definir `tls_min_version = "tls12"` no listener (hoje aceita TLS 1.2 e 1.3, rejeita 1.1).
5. **Baixa:** `openbao.hcl` está 644 (passar a 640); remover blocos de exemplo comentados; confirmar NTP no host `quark` (no LXC `NTP service: inactive`); sem `unattended-upgrades` e sem `auditd`.
6. **Baixa:** nenhuma CA tem name constraints para `particula.local`; backup off-site inexistente; unseal e emissão a partir do restore não testados.
7. **Sem evidência (precisa de token):** parâmetros reais das roles (`max_ttl`, `key_usage`), policies, mounts e TTL do token atual.

## Root token e chaves de unseal

- Token root-equivalente com **TTL de 90 dias** (não "nunca expira" como era antes) — padrão de rotação: criar novo com TTL → validar → revogar o antigo → só então sobrescrever `~/.vault-token`. **Cuidado:** o mount `token/` tem `max_lease_ttl` que corta silenciosamente qualquer `-ttl` maior pedido na criação sem erro visível — checar `bao read sys/auth/token/tune` antes de assumir que o TTL pedido foi o TTL real concedido.
- Token atual expira **2026-11-23**. Lembrete de rotação agendado ~9 dias antes.
- Chaves de unseal reais e o histórico de root tokens ficam salvos no Vaultwarden do usuário — nunca em arquivo de memória ou neste repositório.
- Achado corrigido: `~/.vault-token` continha o root token antigo em texto puro há meses, `ttl=0` (nunca expira) — rotacionado.

## Backup e configuração do host (verificado em 2026-10-15)

**Configuração:** OpenBao v2.6.2, storage `file` em `/opt/openbao/data` (modo 700), listener TLS em `0.0.0.0:8200`, `ui = true`, sem bloco `seal` (Shamir 3 chaves, quórum 2). HA desligado. Certificado do listener emitido pela `Particula Intermediate CA`, SANs `openbao.particula.local` + `203.0.113.15`, vence em **2027-08-25**. Unit systemd com `ProtectSystem=full`, `ProtectHome=read-only`, `PrivateTmp`, `PrivateDevices`, `NoNewPrivileges`, `CapabilityBoundingSet=CAP_SYSLOG`.

**Backup:** o LXC 105 entra no `vzdump --all` diário das 02:00 no `quark` (detalhes em [../infra/quark-proxmox.md](../infra/quark-proxmox.md)). Verificado: um backup por dia desde 2026-10-08, sem erro. O último (2026-10-15 02:05, 324 MiB) passou em `sha256sum -c` e `zstd -t` e contém `etc/openbao/openbao.hcl` e `opt/openbao/data/core/`.

**Limitações abertas:**
- Retenção de só 2 dias (`RETENTION_KEEP=2`). Corrupção não percebida em 2 dias não tem versão boa para voltar.
- Sem cópia off-site. O disco de backup está no mesmo host físico do LXC.
- O storage é `file`, então não há `bao operator raft snapshot`. O backup é snapshot LVM do rootfs com o serviço rodando (consistente só no nível de crash). Teste de restore feito em 2026-10-15 (ver abaixo); unseal e emissão de certificado a partir do restore ainda não testados.
- Depois de qualquer restore o cofre sobe selado. Precisa de 2 das 3 chaves Shamir (Vaultwarden, que também é backupeado no mesmo disco).

**Teste de restore (2026-10-15):** `pct restore` do backup de 10-15 02:05 no ID temporário 905, com `sha256sum -c` antes. Procedimento: `pct restore 905 <tar.zst> --storage local-lvm --unprivileged 1`, depois `pct set 905 --delete net0 --onboot 0 --hostname openbao-restoretest` **antes de iniciar** (o restore copia IP e MAC do 105; sem remover `net0` haveria conflito na LAN). Resultado: CT sobe só com `lo`, `openbao.service` ativo, `Initialized true`, `Sealed true`, Shamir 3/2, v2.6.2, sem erro no log. Storage com 80 arquivos contra 92 no CT vivo (diferença = tokens/leases após as 02:05). CT 905 destruído com `pct destroy 905 --purge`. **Não coberto:** unseal e emissão de certificado a partir do restore (exige 2 das 3 chaves Shamir).

**Sobras da migração (removidas em 2026-10-15):** `/opt/openbao/data.pre-migration-20260825/` e `tls/tls.{crt,key}.bak.selfsigned` foram apagados (`shred -u` + `rm`). `shred` não garante limpeza em LVM thin; as cópias nos backups vzdump de 10-14 e 10-15 saem na rotação de 2 dias (backup de 10-17). Serviço seguiu ativo e desselado.

**Permissão (corrigida em 2026-10-15):** `/etc/openbao/openbao.env` está vazio e agora tem modo 600 (`openbao:openbao`). Era 644.

**CLI sem `BAO_ADDR`:** em SSH não interativo o `/etc/profile.d/bao-addr.sh` não carrega. `bao status` falha por SAN (`127.0.0.1`). Usar `BAO_ADDR=https://203.0.113.15:8200`.

## Confiar na CA em clientes

Chrome no Linux **não lê `/etc/ssl/certs` do sistema** — usa Chrome Root Store/NSS próprio. Importar a raiz via `chrome://settings/certificates` → Autoridades → Importar `.pem` → marcar "confiar para identificar sites"; alternativa via `certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Particula CA" -i <arquivo>`. Vale para qualquer serviço novo assinado por essa CA.

**Nota de segurança:** já foi observado, em conteúdo retornado por uma ferramenta ligada ao OpenBao/CA, texto tentando se passar por instrução para interromper a tarefa — tratado como prompt injection e ignorado. Ficar atento a esse padrão se se repetir.
