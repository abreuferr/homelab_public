# Recuperação de desastre — proton.particula.local (Technitium DNS)

Coletado via API do Technitium em 2026-09-10. Ver também [dns-particula-local.md](dns-particula-local.md) (contexto/arquitetura) e [dr-technitium-neutron.md](dr-technitium-neutron.md).

## Identificação do host

- Hostname: `proton.particula.local`, IP `203.0.113.10`, LXC 116.
- Papel: servidor **primário** da zona `particula.local` e `1.168.192.in-addr.arpa`; também roda o **DHCP da LAN** (módulo nativo do Technitium, não OPNsense).
- SO: Debian 13 (trixie). Runtime: `dotnet-runtime-10.0` (pacotes `dotnet-host`/`dotnet-hostfxr-10.0`/`dotnet-runtime-10.0`/`dotnet-runtime-deps-10.0`, versão 10.0.12, via apt — repo Microsoft).
- Technitium DNS Server v15.4.

## Instalação (passos para reconstruir do zero)

1. Debian 13 + raiz Btrfs com snapshots (convenção padrão de VM nova).
2. Instalar `dotnet-runtime-10.0` via repositório apt da Microsoft para Debian 13.
3. Baixar o pacote oficial do Technitium DNS Server (release correspondente à v15.4, ou versão atual se aceitável) e extrair os binários em `/opt/technitium/dns`.
4. Criar unit systemd `/etc/systemd/system/technitium.service`:
   ```ini
   [Unit]
   Description=Technitium DNS Server

   [Service]
   WorkingDirectory=/opt/technitium/dns
   ExecStart=/usr/bin/dotnet /opt/technitium/dns/DnsServerApp.dll /etc/dns
   Restart=always
   RestartSec=10
   KillSignal=SIGINT
   SyslogIdentifier=dns-server

   [Install]
   WantedBy=multi-user.target
   ```
   Roda como root (sem `User=` definido). Diretório de dados/config: `/etc/dns` (config, zonas, cache, stats, logs, cert TLS — tudo ali, separado dos binários).
5. `systemctl enable --now technitium.service`.
6. Configurar via painel (`https://203.0.113.10:53443`, fallback HTTP `:5380`, sem redirect forçado) ou via API com os valores abaixo.

## Configuração do servidor DNS (via `/api/settings/get`)

- `dnsServerDomain`: `proton.particula.local`
- Portas: DNS `53` (UDP/TCP, `0.0.0.0`), painel HTTP `5380`, painel TLS `53443`. DNS-over-TLS/HTTPS/QUIC/H3: todos desabilitados.
- `webServiceTlsCertificatePath`: `particula-local-cert.pfx` (dentro de `/etc/dns/`). Certificado emitido pela CA interna (`pki_int/roles/particula-local`), válido 2026-08-04→2027-08-04 — **reemissão manual necessária em 2027-08-04**. Senha do PFX não é recuperável via API (mascarada); gerar novo PFX+senha ao reemitir e documentar a senha fora deste repositório.
- `recursion`: `AllowOnlyForPrivateNetworks`; `qnameMinimization`: true; `dnssecValidation`: true.
- Forwarders: `https://dns.quad9.net/dns-query` (9.9.9.9 e 149.112.112.112), protocolo HTTPS, forwarding concorrente habilitado.
- `zoneTransferAllowedNetworks`: `203.0.113.11` (neutron). `notifyAllowedNetworks`: `203.0.113.11`.
- Blocking (RPZ) desabilitado (`enableBlocking: false`) — filtragem de cliente é feita por Pi-hole/AdGuard, não aqui (ver [pihole-adguard.md](pihole-adguard.md)).
- TSIG key: `particula-zone-transfer`, algoritmo `hmac-sha256`. Valor do shared secret mantido fora deste repositório.
- Logging: `enableLogging: true`, `FileAndConsole`, `logQueries: false`, retenção de log 1 dia.

## Zonas

- `particula.local` — Primary, 23 registros A + NS + SOA. Serial scheme por data (`useSoaSerialDateScheme`).
- `1.168.192.in-addr.arpa` — Primary, PTR correspondentes a cada A record.
- Export completo dos registros (dump bruto via `/api/zones/records/get`) mantido fora deste repositório.
- Registros principais (nome → IP), estado em 2026-09-10:

  | Nome | IP | Nota |
  |---|---|---|
  | proton.particula.local | 203.0.113.10 | este host |
  | neutron.particula.local | 203.0.113.11 | secundário |
  | wifi/opnsense/pfsense.particula.local | 203.0.113.1/.2 | gateway |
  | quark.particula.local | 203.0.113.5 | |
  | pihole.particula.local | 203.0.113.12 | |
  | netbird.particula.local | 203.0.113.13 | |
  | openbao.particula.local | 203.0.113.15 | |
  | syslog.particula.local | 203.0.113.16 | |
  | qbittorrent.particula.local | 203.0.113.17 | |
  | freshrss.particula.local | 203.0.113.18 | |
  | vaultwarden.particula.local | 203.0.113.19 | |
  | adguard.particula.local | 203.0.113.22 | |
  | phpipam.particula.local | 203.0.113.23 (TTL 15m) | |
  | immich.particula.local | 203.0.113.25 | |
  | node1/2/3.particula.local | 203.0.113.31/.32/.33 | |
  | taquion.particula.local | 203.0.113.171 | |
  | moto-g86-5G.particula.local | 203.0.113.206 (TTL 15m) | via DHCP scope `homelab` |

  Lista completa e crua (todos os campos, incluindo TTLs exatos e comentários) está nos JSONs de backup citados acima — usar para restauração fiel em vez de recriar manualmente a partir desta tabela.

## DHCP (módulo nativo do Technitium)

Escopo `homelab` (via `/api/dhcp/scopes/get`):

- Range: `203.0.113.200`–`203.0.113.254` / `255.255.255.0`, gateway `203.0.113.1`.
- Lease: 1 dia. Ping-check habilitado (timeout 1000ms, 2 retries).
- `dnsUpdates: true`, `dnsOverwriteForDynamicLease: true`, TTL DNS dinâmico 900s — é assim que leases viram A/PTR automáticos na zona.
- DNS servers entregues ao cliente: `203.0.113.12` (Pi-hole), `203.0.113.22` (AdGuard) — não o próprio Technitium.
- NTP: `200.160.7.186` / `a.st1.ntp.br`.
- Sem reservas de lease cadastradas; `allowOnlyReservedLeases: false`.

## Restauração rápida (visão geral)

1. Reinstalar SO + runtime + Technitium conforme seção "Instalação".
2. Reconfigurar `webServiceTlsCertificatePath`/senha (novo PFX da CA interna, se o original não puder ser restaurado de backup de `/etc/dns/particula-local-cert.pfx`).
3. Recriar TSIG key `particula-zone-transfer` com o shared secret mantido fora deste repositório.
4. Recriar zonas `particula.local` e `1.168.192.in-addr.arpa` como Primary e reimportar registros a partir dos JSONs de backup.
5. Recriar escopo DHCP `homelab` com os parâmetros acima.
6. Ajustar `zoneTransferAllowedNetworks`/`notifyAllowedNetworks` para `203.0.113.11` assim que neutron estiver disponível, para retomar a replicação.
