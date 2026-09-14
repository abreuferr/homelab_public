# Padrões e gotchas — Debian, BIND9, MariaDB/Galera

## Ordem correta ao provisionar um novo servidor

1. **Desabilitar AppArmor proativamente** (`apparmor=0` no GRUB, antes do reboot e antes de instalar o serviço) quando o ambiente não precisa de confinamento. Fazer isso reativamente (só depois de uma queda causada pelo próprio AppArmor) custa uma depuração inteira.
2. Se o objetivo for só parar o confinamento (não remover o pacote), **desabilitar, nunca fazer `apt purge`** de `apparmor`/`libapparmor1` — no Debian isso pode arrastar `sysvinit-core` como dependência e tentar trocar o sistema de init no meio do processo.
3. Decidir os secret paths no gerenciador de segredos (OpenBao) **antes** de criar credenciais de serviço (usuário SST do Galera, certificados SSL) — criar primeiro e migrar depois gera duplicação permanente (original + cofre).

## Gotchas técnicos específicos do Debian

- **BIND9 + DNSSEC:** `dnssec-policy` impõe `max-zone-ttl` padrão de 86400s — uma zona com `$TTL` maior que isso **para de carregar no master** (outage real) até o TTL ser reduzido.
- **BIND9 + AppArmor:** o perfil `usr.sbin.named` do Debian só permite `/etc/bind/** r` (read-only) e reserva `/var/cache/bind` para dados de slave. Zonas master com `inline-signing` (que escrevem journal/chaves) precisam ficar em `/var/lib/bind` (rw). Falhas de escrita nesse cenário não aparecem no log do `named` — só em `journalctl -k | grep apparmor`.
- **Trixie (Debian 13) + BIND9:** o pacote removeu `named.conf.default-zones` em favor de `named.conf.root-hints` — um `named.conf` antigo preservado por confold quebra o `named-checkconf` do postinst e trava o `full-upgrade`.
- **Trixie + MariaDB/Galera:** o pacote habilita `mariadb.socket` (socket activation na 3306), que colide com o `galera_recovery` no bootstrap ("Address already in use"). Fix: `systemctl disable --now mariadb.socket` antes de `galera_new_cluster`.
- **DNSSEC em domínio `.local`:** não dá para validar com a flag AD de um `dig` comum (autoritativo nunca seta AD na própria resposta) — usar `delv -a <trust-anchors> +root=<zona>` para validação real ponta a ponta.

## Runbook de referência

Runbook parametrizado de upgrade Debian 12→13 (canário, snapshot, upgrade destacado via `systemd-run`, auto-reparo do bind9, fix do mariadb.socket, Galera cold vs. rolling, unseal do cofre) — útil como checklist para qualquer upgrade de major version em hosts do homelab que rodem esses mesmos serviços (proton/neutron para BIND-like, openbao).
