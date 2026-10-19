# neutrino.particula.local — bridge `br0` para KVM/libvirt

Estação de trabalho openSUSE (`neutrino`) que também roda VMs locais com KVM/QEMU e libvirt. As VMs entram direto na LAN `203.0.113.0/24` por uma bridge Linux. Não tem relação com o `quark` (Proxmox) nem com a VM DNS `neutron`.

## Rede do host

Gerenciada pelo NetworkManager (`nmcli`). A interface física `em1` é porta da bridge e não tem IP próprio.

| Item | Valor |
|---|---|
| Bridge | `br0` (STP desligado) |
| Porta | `em1` (perfil `br0-em1`) |
| IP | `203.0.113.171/24`, estático, fora do range do DHCP |
| Gateway | `203.0.113.1` |
| DNS | `203.0.113.12`, `203.0.113.22` (busca `particula.local`) |
| MAC da bridge | clonado de `em1` (`ethernet.cloned-mac-address`) |

O perfil `em1` original ficou com `autoconnect no`, senão disputa a interface com a bridge. O `br0` tem `autoconnect-ports 1` para subir sempre junto com `em1`.

Perfis em `/etc/NetworkManager/system-connections/` (`br0.nmconnection`, `br0-em1.nmconnection`).

### Como foi criado

```
sudo nmcli con add type bridge ifname br0 con-name br0 bridge.stp no \
  ethernet.cloned-mac-address <MAC de em1>
sudo nmcli con add type bridge-slave ifname em1 master br0 con-name br0-em1
sudo nmcli con modify br0 ipv4.method manual ipv4.addresses 203.0.113.171/24 \
  ipv4.gateway 203.0.113.1 ipv4.dns "203.0.113.12 203.0.113.22" \
  ipv4.dns-search particula.local ipv4.ignore-auto-dns yes ipv4.ignore-auto-routes yes \
  connection.autoconnect-ports 1
sudo nmcli con modify em1 connection.autoconnect no
sudo nmcli con down em1; sudo nmcli con up br0; sudo nmcli con up br0-em1
```

Antes de fixar o IP, `arping -D -I br0 203.0.113.171` não recebeu resposta (sem conflito). Recomendado reservar o `.171` no roteador.

### Reverter

```
sudo nmcli con delete br0 br0-em1
sudo nmcli con modify em1 connection.autoconnect yes
sudo nmcli con up em1
```

## Rede libvirt `host-bridge`

Rede em modo bridge sobre o `br0`, com autostart e persistente:

```xml
<network>
  <name>host-bridge</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

Criada com `virsh net-define`, `net-autostart` e `net-start`. A rede `default` (NAT) continua existindo.

Uso em VM: `virt-install --network network=host-bridge,model=virtio`, ou no virt-manager "Virtual network 'host-bridge'". A VM precisa de IP próprio na `203.0.113.0/24` (estático ou DHCP). O DHCP do roteador não atende o range do servidor.

## Acesso ao libvirt sem senha

No openSUSE o `virt-manager` usa `qemu:///system` via polkit. A regra `/usr/share/polkit-1/rules.d/50-libvirt.rules` libera `org.libvirt.unix.manage` para o grupo `libvirt`. O usuário `cferreira` foi adicionado a esse grupo (`/usr/sbin/usermod -aG libvirt cferreira`). Vale só em sessão nova: logout/login, ou `sg libvirt -c "virsh -c qemu:///system ..."`.

Sem o grupo, o virt-manager falha com `no polkit agent available to authenticate action 'org.libvirt.unix.manage'`.

## Gotcha: chave SSH mudou numa VM nova

VM nova que herda o IP de outra máquina gera `REMOTE HOST IDENTIFICATION HAS CHANGED`. Confirmar antes de remover a chave: `ip neigh show <ip>` no host mostra o MAC, e `virsh domiflist <vm>` mostra o MAC da VM. Se baterem, `ssh-keygen -R "[<ip>]:<porta>"` e conferir o fingerprint na primeira conexão.

## Primeira VM

`segura-42`, na `br0`, IP `203.0.113.201` (validado em 2026-10-19).
