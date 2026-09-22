# DNS de particula.local

## Servidores autoritativos/recursivos

`proton.particula.local` (203.0.113.10, LXC 116) é o servidor **primário** e `neutron.particula.local` (203.0.113.11, LXC 117) é o **secundário** da zona `particula.local`, rodando **Technitium DNS Server** (v15.4, atualizado para Debian 13/trixie em 2026-08-23). Zona replica de proton para neutron via zone transfer (TSIG) — editar registros só em proton; a API do Technitium recusa alteração direta em zona secundária (`Cannot delete record in Secondary zone`), e a mudança propaga automaticamente para neutron (confirmado 2026-09-09).

Painel web em `https://<host>.particula.local:53443` (HTTP 5380 mantido como fallback, sem redirect forçado). Certificados TLS emitidos pela CA interna (`pki_int/roles/particula-local`), válidos 2026-08-04→2027-08-04 — **precisa reemissão manual em 2027-08-04**, sem renovação automática configurada.

**Regra permanente:** hosts do tipo servidor (VMs/CTs de infraestrutura, resolver do OPNsense, etc.) devem usar `203.0.113.10`/`203.0.113.11` como DNS primário/secundário — não Pi-hole/AdGuard, que existem para uso de cliente/filtragem (ver [pihole-adguard.md](pihole-adguard.md)).

## Auditoria e hardening (concluída 2026-08-04)

- Zone transfer protegido por TSIG (`particula-zone-transfer`, HMAC-SHA256) entre proton e neutron.
- Dynamic Update desabilitado (`update=Deny`) nas 4 combinações proton/neutron × zona direta/reversa — DHCP (ver seção abaixo) não está integrado com Dynamic DNS Update, e o `neutron` estava com `update: Allow` sem ACL nenhuma.
- Zona órfã `home` (só NS+SOA, sem registro real, resquício de nome cogitado antes de `particula.local` virar definitivo) removida.
- DNSSEC validation + QNAME minimization habilitados nos dois servidores (validado funcionalmente: domínio com DNSSEC quebrado dá SERVFAIL, domínio válido resolve normal).
- Zonas internas seguem **sem assinatura DNSSEC própria** (Unsigned) — considerado opcional para domínio só-interno sem delegação externa.

## Upgrade Debian 12→13 (2026-08-23)

Testado antes via clone completo dos LXCs (a partir de snapshot, com IP de teste `.30`/`.31` para não colidir com produção) — upgrade seguro, sem efeito colateral na produção (clones tentaram enviar NOTIFY para o `neutron` real e foram recusados pelo TSIG/ACL, confirmando isolamento). Executado em produção logo depois: `neutron` (secundário) primeiro, validado, depois `proton` (primário) — um de cada vez, sempre com snapshot novo antes do upgrade real, mesmo já validado em clone. Runtime .NET subiu de 8.x para 10.0.11 automaticamente; serviço chama-se `technitium.service`.

## DHCP

O serviço de DHCP da LAN está a cargo de `proton.particula.local` (não do OPNsense) — usa o módulo DHCP nativo do Technitium DNS Server. `neutron` é só o secundário de DNS, não assume DHCP se `proton` cair.

## Registros conhecidos

**Lista completa revalidada em 2026-09-22** (via API do Technitium em proton, todos os A records da zona `particula.local`, IPs sanitizados no range `203.0.113.0/24`):

| Host | IP |
|---|---|
| wifi.particula.local | .2 |
| quark.particula.local | .5 |
| proton.particula.local | .10 |
| neutron.particula.local | .11 |
| pihole.particula.local | .12 |
| netbird.particula.local | .13 |
| openbao.particula.local | .15 |
| syslog.particula.local | .16 |
| qbittorrent.particula.local | .17 |
| freshrss.particula.local | .18 |
| vaultwarden.particula.local | .19 |
| adguard.particula.local | .22 |
| phpipam.particula.local | .23 |
| immich.particula.local | .25 |
| opnsense.particula.local / pfsense.particula.local | .1 (dois nomes pro mesmo IP) |
| taquion.particula.local | .171 |

Além destes, existem alguns hosts registrados via DHCP dinâmico (estação de trabalho, celular) que não são serviços de infraestrutura e não estão listados aqui.

**Correção 2026-09-22:** `myspeed.particula.local` (.15) **não existe mais** — nome antigo desta doc estava desatualizado. O registro em .15 foi trocado para `openbao.particula.local` em 2026-08-25 (junto com a instalação da CA de produção no LXC 105).

`ca.particula.local` (.14) e `bitwarden.particula.local` (.21, host antigo) tiveram as VMs apagadas em 2026-09-09; A records removidos no mesmo dia (via API em proton, propagado a neutron).

Tokens de admin do Technitium usados para gerenciar registros via API pontualmente **nunca são persistidos** em arquivo/memória — são passados na sessão e descartados (`shred`) logo após o uso. Exceção: os tokens fixos `api_token` de proton/neutron, gerados em 2026-09-10 especificamente para fins de disaster recovery, são mantidos fora deste repositório — ver [dr-technitium-proton.md](dr-technitium-proton.md) e [dr-technitium-neutron.md](dr-technitium-neutron.md).

## Cache negativo do dnsmasq local em clientes

Quando um registro novo em proton/neutron não resolve num cliente, mesmo confirmado via `dig` direto no servidor, suspeitar de **cache negativo do dnsmasq local** do cliente (`/etc/dnsmasq.d/split-dns.conf`, ver [pihole-adguard.md](pihole-adguard.md) para o mecanismo de `all-servers`) — o cliente cacheou a resposta negativa de antes do registro existir/propagar (TTL negativo = SOA minimum, 900s nessa zona). Resolve com `systemctl restart dnsmasq` no cliente.
