# Recuperação de desastre — neutron.particula.local (Technitium DNS)

Coletado via API do Technitium em 2026-09-10. Ver também [dns-particula-local.md](dns-particula-local.md) (contexto/arquitetura) e [dr-technitium-proton.md](dr-technitium-proton.md) — a maior parte da config é idêntica ao primário; este arquivo documenta só as diferenças e o que é específico de neutron.

## Identificação do host

- Hostname: `neutron.particula.local`, IP `203.0.113.11`, LXC 117.
- Papel: servidor **secundário** da zona `particula.local` e `1.168.192.in-addr.arpa` (zone transfer via TSIG a partir de proton). Não assume DHCP se proton cair — é só o secundário de DNS.
- SO: Debian 13 (trixie). Runtime `dotnet-runtime-10.0` (mesmos pacotes de proton). Technitium DNS Server v15.4.

## Instalação (passos para reconstruir do zero)

Idêntica à de proton (ver [dr-technitium-proton.md](dr-technitium-proton.md), seção "Instalação"): mesmo unit systemd `technitium.service`, mesmo diretório de dados `/etc/dns`, binários em `/opt/technitium/dns`.

## Configuração do servidor DNS (via `/api/settings/get`)

Praticamente idêntica a proton, exceto:

- `dnsServerDomain`: `neutron.particula.local`
- `defaultResponsiblePerson`: `null` (proton usa `admin@particula.local` — diferença não intencional, mas presente no estado atual).
- `zoneTransferAllowedNetworks` e `notifyAllowedNetworks`: **vazios** (neutron não é fonte de transferência — só recebe de proton).
- `logQueries`: `true` (em proton está `false`).
- Mesmo certificado TLS (`particula-local-cert.pfx`, mesma CA/validade que proton — arquivos são independentes por host, senha também não recuperável via API).
- Mesma TSIG key `particula-zone-transfer` (hmac-sha256) — shared secret idêntico ao de proton, mantido fora deste repositório.
- Forwarders, recursion, DNSSEC validation, blocking: mesmos valores de proton.

## Zonas

- `particula.local` e `1.168.192.in-addr.arpa` — **Secondary**, replicadas de proton via zone transfer (TSIG). Não editar aqui; API recusa alteração direta em zona secundária.
- Não é necessário restaurar registros manualmente: basta recriar as zonas como Secondary apontando para proton (`203.0.113.10`) com a TSIG key, e o zone transfer repovoa tudo automaticamente.
- Export bruto do estado das zonas (metadados, sem registros individuais — não aplicável a secundária), mantido fora deste repositório.

## DHCP

Escopo `default` existe mas está **desabilitado** (`enabled: false`) — neutron não serve DHCP em operação normal. Parâmetros (caso precise habilitar em contingência, proton indisponível):

- Range: `203.0.113.200`–`203.0.113.250` / `255.255.255.0`.
- Sem os demais parâmetros do escopo `homelab` de proton configurados (é um escopo default, não replicado).

## Restauração rápida (visão geral)

1. Reinstalar SO + runtime + Technitium conforme seção "Instalação" (igual a proton).
2. Reconfigurar certificado TLS (novo PFX da CA interna, se não houver backup de `/etc/dns/particula-local-cert.pfx`).
3. Recriar a TSIG key `particula-zone-transfer` com o shared secret mantido fora deste repositório.
4. Criar as zonas `particula.local` e `1.168.192.in-addr.arpa` como **Secondary**, apontando para proton (`203.0.113.10`) com a TSIG key — os registros vêm via zone transfer, não precisam ser reimportados manualmente.
5. Não habilitar DHCP aqui a menos que proton esteja indisponível permanentemente (nesse caso, ver escopo `homelab` completo em [dr-technitium-proton.md](dr-technitium-proton.md)).
6. Se proton for reconstruído do zero, garantir que `zoneTransferAllowedNetworks`/`notifyAllowedNetworks` em proton incluam `203.0.113.11` para a replicação retomar.
