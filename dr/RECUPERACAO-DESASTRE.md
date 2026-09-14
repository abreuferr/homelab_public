# Plano de recuperação de desastre — particula.local

Roteiro para reconstruir o homelab depois de perder o host `quark` (Proxmox), assumindo que os backups `vzdump` (off-site, em implementação pelo usuário) estão disponíveis para restauração. Não cobre reconstrução sem nenhum backup — nesse caso, seria refazer cada serviço do zero usando a documentação em `infra/`, `rede/`, `seguranca/` e `servicos/` como referência, sem garantia de recuperar dados.

## Pré-requisitos antes de começar

- Proxmox reinstalado manualmente em `quark`, mesmo IP `203.0.113.5/24`, com bridge/VLAN configurada manualmente (responsabilidade do usuário, fora deste roteiro).
- Backup `vzdump` acessível para restauração (`qmrestore`/`pct restore`).
- Acesso administrativo disponível — obrigatório para qualquer acesso a qualquer host restaurado.
- Senha mestra do Vaultwarden do usuário — não fica documentada em lugar nenhum deste repositório, por design. Sem ela não dá para recuperar as chaves de unseal do OpenBao (ver Fase 1, passo 3).
- Credenciais PPPoE do provedor de internet, para a WAN do OPNsense — não estão neste repositório.
- Um dispositivo com **IP estático manual** na faixa `203.0.113.0/24` para acessar o painel do Proxmox (`https://203.0.113.5:8006`) — o DHCP da LAN é o `proton` (ver Fase 1, passo 2), que só volta depois de restaurado; sem IP estático não dá pra alcançar o Proxmox nesse meio-tempo.

## Fase 0 — Base do Proxmox

1. Rede: bridge/VLAN configurada manualmente pelo usuário (fora deste roteiro, mesma decisão que a reinstalação do Proxmox) — `203.0.113.5/24`, gateway `203.0.113.1` (assim que o OPNsense voltar).
2. Reaplicar hardening do host: hardening de SSH, firewall do Proxmox restringindo 8006/3128 à `203.0.113.0/24`, certificado do painel web emitido pela CA interna assim que o OpenBao estiver de pé (Fase 1, passo 4).
3. Storage: recriar `local` (dir) e `local-lvm` (thin LVM) com tamanhos equivalentes ao inventário anterior (ver [infra/quark-proxmox.md](infra/quark-proxmox.md)); criar `/mnt/backup` apontando para onde o backup off-site foi restaurado/montado.

## Fase 1 — Ordem de restauração das VMs/CTs

A ordem importa por dependência, não é arbitrária nem por VMID.

1. **OPNsense (VMID 102)** — primeiro. É o gateway/firewall da rede; sem ele não há saída WAN (necessária para baixar pacotes durante o resto da restauração).
   - Restaurar (`qmrestore`), validar IP `203.0.113.1`, reconectar WAN PPPoE com as credenciais do provedor.
2. **proton (LXC 116) e neutron (LXC 117)** — DNS **e DHCP** (`proton` é quem serve DHCP pra LAN, não o OPNsense — `neutron` é só DNS secundário, não assume DHCP se `proton` cair). Sem `proton`, nenhum dispositivo novo entrando na LAN recebe IP via DHCP (o dispositivo usado para administrar a restauração precisa de IP estático até aqui, ver Pré-requisitos); sem os dois, nenhum hostname `*.particula.local` resolve.
   - Restaurar os dois (`pct restore`), validar com `dig @203.0.113.10` e `dig @203.0.113.11`, confirmar DHCP respondendo (`proton`) antes de voltar a depender dele em vez do IP estático.
   - Se o `pct restore` falhar ou o backup estiver indisponível, reconstrução completa do zero (instalação, settings, zonas, DHCP) está documentada em [rede/dr-technitium-proton.md](../rede/dr-technitium-proton.md) e [rede/dr-technitium-neutron.md](../rede/dr-technitium-neutron.md).
3. **Vaultwarden (LXC 110)** — antes do OpenBao. É onde estão as chaves de unseal (Shamir) e o root token da CA.
   - Restaurar (`pct restore`), desbloquear com a senha mestra do usuário — isso quebra a dependência circular (Vaultwarden não depende do OpenBao para abrir).
4. **openbao (LXC 105)** — CA.
   - Restaurar (`pct restore`), desselar manualmente com 2 das 3 chaves Shamir recuperadas do Vaultwarden (passo anterior) — ver [seguranca/pki-ca-openbao.md](seguranca/pki-ca-openbao.md).
   - Validar: `GET /v1/pki_int/ca/pem` respondendo, emissão de certificado de teste funcionando.
   - **Gotchas de TLS no unseal (confirmados em 2026-09-10, corrigidos em definitivo no próprio LXC 105 no mesmo dia):**
     - O CLI `bao` aponta para `127.0.0.1` por padrão, mas o certificado do OpenBao só tem SAN para `203.0.113.15` — **corrigido** via `/etc/profile.d/bao-addr.sh` (`export BAO_ADDR=https://203.0.113.15:8200`), carregado automaticamente em qualquer login shell no LXC. Se precisar rodar `bao` fora de um shell de login (ex. script não-interativo), exportar a variável manualmente.
     - A verificação de CA falhava (`certificate signed by unknown authority`) porque o host não confiava na CA interna — e a dependência circular real é que **não dá pra buscar a cadeia via API pra resolver isso**, pois os endpoints `pki_root/ca/pem`/`pki_int/ca/pem` também exigem o cofre desselado. **Corrigido** instalando root + intermediária no trust store do sistema (`/usr/local/share/ca-certificates/particula-{root,int}-ca.crt` + `update-ca-certificates`) — validado com `openssl s_client` (`Verify return code: 0`) e `bao status` sem `BAO_SKIP_VERIFY`. Isso faz parte do rootfs do LXC, então sobrevive a um `pct restore` de backup feito depois de 2026-09-10; backups **anteriores** a essa data ainda vão cair no gotcha original — usar `BAO_SKIP_VERIFY=true` só nesse caso, como contorno pontual de bootstrap.
   - **Gotcha na emissão de certificado logo após desselar:** o campo `ca_chain` da resposta de `pki_int/issue/particula-local` trouxe só a intermediária (`Particula Intermediate CA`), **sem a raiz** (`Particula CA`) — quebra verificação TLS em qualquer serviço que dependa da cadeia completa, até buscar a raiz à parte (`GET /v1/pki_root/ca/pem`, sem auth, já funciona normalmente com o cofre desselado) e completar o `ca-chain.pem` manualmente. Conferir esse detalhe em qualquer emissão de certificado feita logo após uma restauração/unseal.
5. **Pi-hole (CT 104) e AdGuard (CT 103)** — filtragem de clientes. Não bloqueiam nada depois do DNS de servidor (passo 2) já estar de pé.
6. **Demais serviços, em qualquer ordem:** phpIPAM (119), syslog (118), freshrss (107), qbittorrent (106), immich (114), netbird (100).
   - Cada um: restaurar, depois validar hardening SSH (7 parâmetros) e certificado TLS ainda válido — se vencido, reemitir via OpenBao (já deve estar de pé desde o passo 4).

## Fase 2 — Validação final

- Checklist de hardening SSH aplicado em todos os hosts restaurados.
- Confirmar `auto_rebuild` da CRL ainda ativo no OpenBao (`pki_root/config/crl` e `pki_int/config/crl`).
- Reconfigurar o backup diário (`/usr/local/sbin/proxmox-backup.sh`, cron `0 2 * * *`) e a rotina off-site — não presumir que sobreviveu à reinstalação do zero.
- Registrar neste arquivo qualquer desvio real da ordem/procedimento acima, para a próxima vez.

## Fora do escopo deste roteiro

- Credenciais PPPoE do provedor de internet.
- Dados perdidos que não estavam no backup mais recente restaurado.
- Restauração da workstation de administração do usuário — não faz parte do homelab.
