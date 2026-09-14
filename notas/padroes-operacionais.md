# Padrões operacionais do homelab

## Novas VMs

Toda VM nova criada no homelab: **Debian 13 (trixie)** com **Btrfs** na raiz (subvolumes `@`/`@home`, habilita snapshots/rollback via subvolumes ou snapper, além de snapshot no nível do hipervisor). Regra geral e permanente do usuário (vale além do homelab também). Discos de VM sempre em `/data/images` (pool `default` do libvirt) — nunca em `/var/lib/libvirt/images`.

Exceções documentadas conscientemente: `syslog.particula.local` (VM antiga) usava ext4 a pedido do usuário; `bitwarden.particula.local` usa ext4 e não foi convertido sem pedido explícito.

## Preferir correção pontual a restaurar VM inteira

Quando um serviço quebra por efeito colateral de uma mudança de config **recente e intencional** (ex. rotação de certificado), e o diagnóstico já identificou a causa raiz perto de uma correção pontual, **preferir a correção pontual** a restaurar a VM inteira a partir de backup — mesmo que o backup pareça o caminho "mais seguro".

**Caso confirmado (incidente de GUI do pfSense, 2026-08-04):** restaurar reverteria uma troca de certificado intencional (perdendo esse trabalho) e causaria outage real do firewall de produção (estados de conexão, DHCP, VPN). A correção pontual resolveu sem nenhuma dessas perdas.

**Como aplicar:** diante de "não seria o caso de restaurar/reverter X?", avaliar: (a) o backup é anterior a uma mudança intencional recente que seria perdida? (b) o diagnóstico já está perto o suficiente de uma correção pontual de baixo risco? Se sim aos dois, recomendar continuar o fix pontual e guardar o restore como plano B caso o pontual falhe ou revele problema mais profundo.

## DNS: servidores usam proton/neutron

Ver [../rede/dns-particula-local.md](../rede/dns-particula-local.md) — hosts do tipo servidor usam `203.0.113.10`/`.11`, não Pi-hole/AdGuard.

## Threat model: sem hardening de exposição à internet

Ver [../seguranca/threat-model.md](../seguranca/threat-model.md) — não propor fail2ban/proxy reverso para serviços restritos ao LAN.

## Host key SSH mudou sem motivo aparente: verificar via `pct exec`/`pct enter` antes de suspeitar de MITM

Um host (`phpipam`, 2026-09-06) apresentou aviso de "REMOTE HOST IDENTIFICATION HAS CHANGED" no `known_hosts` sem nenhum registro de reinstalação/restore no repositório. Em vez de aceitar cegamente (`StrictHostKeyChecking=accept-new`) ou presumir ataque, comparar a host key servida agora com a que está em `/etc/ssh/ssh_host_ed25519_key.pub` **dentro do LXC**, acessado via `pct exec <vmid> -- cat ...` a partir do Proxmox (canal confiável, não passa pela rede) — não pela própria conexão SSH que está sob suspeita. Se bater, é só `known_hosts` desatualizado (`ssh-keygen -R <host>` remove a entrada antiga). Ver também `uptime -s` do container para descartar reboot/restore recente coincidente.

**Como aplicar:** só remover a entrada do `known_hosts` depois dessa verificação — nunca por conveniência.
