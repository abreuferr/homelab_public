# syslog central — syslog.particula.local

Host de syslog central do homelab. Migrado de VM (108, apagada de vez — `qm destroy --purge` mesmo dia, sem janela de retenção, a pedido explícito do usuário) para LXC (CT 118, 203.0.113.16) em 2026-08-25, mesmo padrão de migração VM→LXC usado no OpenBao (ver [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md)).

**Importante:** histórico de log antigo **não foi migrado** (decisão do usuário) e não existe mais em lugar nenhum — sem rollback disponível a partir de agora; qualquer regressão exige rebuild do zero.

## Configuração

rsyslog 8.2504.0-1, recebe UDP+TCP :514 só de `OPNsense.particula.local` (ver [../rede/firewall-gateway.md](../rede/firewall-gateway.md)), grava em `/var/log/remote/<host>/messages.log`, retenção 14 dias via `/etc/logrotate.d/remote-syslog` (antes não rotacionava e acumulou 930MB). Permissão de `10-remote.conf` corrigida para `root:root 0644` (estava com dono/grupo incorretos).

Hardening SSH aplicado desde a criação — CT nasceu sem usuário administrativo/sem `sudo` (template mínimo), configurado do zero no mesmo padrão do resto do homelab.

**Gotcha de update:** `apt full-upgrade` num CT recém-criado pode dar erro transitório de D-Bus (`Failed to connect to system scope bus`) que impede o `deb-systemd-invoke` de reiniciar `ssh.service` sozinho — não assumir que os serviços afetados reiniciaram; checar/reiniciar manualmente ou dar `pct reboot`.

## Incidente 2026-09-05

Dois problemas encontrados numa checagem de rotina, ambos corrigidos no mesmo dia:

- **TCP:514 não bindava** desde o boot de 03/09 15:11 (`Cannot assign requested address` — rsyslog subiu antes da interface `eth0` ter IP pronto; só UDP:514 ficava ativo). Resolvido com `systemctl restart rsyslog` depois que a rede já estava estável — voltou a bindar TCP normalmente. Se voltar a acontecer logo após boot, investigar ordering do `rsyslog.service` vs `network-online.target`.
- **`logrotate.service` marcado `failed`** (04/09 e 05/09), mas a rotação em si funcionava. Causa: sobra de arquivo `/etc/logrotate.d/remote-syslog.bak-20260828175224` (backup de edição anterior deixado dentro do próprio diretório de config, que o logrotate varre inteiro) com entrada duplicada para `messages.log`. Movido para `/root/logrotate.d-backups/` (não apagado). **Lição:** arquivo de backup de config nunca deve ficar dentro de um diretório que o próprio serviço varre (`/etc/logrotate.d/`, `/etc/rsyslog.d/` etc.) — guardar backups fora da árvore de config ativa.
