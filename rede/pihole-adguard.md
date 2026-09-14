# Pi-hole e AdGuard Home

Os dois coexistem **intencionalmente** em produção real — não é resquício de migração nem redundância a resolver, não sugerir consolidação/remoção de nenhum dos dois. Servem clientes/filtragem; servidores de infraestrutura usam proton/neutron como DNS (ver [dns-particula-local.md](dns-particula-local.md)).

## Pi-hole — pihole.particula.local (203.0.113.12, CT 104 em quark)

Acesso SSH: administrativo (root via SSH desativado), porta 22.

**API (Pi-hole v6 / FTL v6.7):** `POST /api/auth` com `{"password": "..."}` retorna `sid`/`csrf` para as chamadas seguintes. Sessões via app password são **somente leitura** (`webserver.api.app_sudo = false` bloqueia `PATCH /api/config`) — mudanças de config exigem SSH + `pihole-FTL --config <chave> <valor>` (aplica e recarrega em tempo real) ou editar `/etc/pihole/pihole.toml` + `systemctl restart pihole-FTL`.

**Bug conhecido:** se a senha principal do admin (`webserver.api.pwhash`) estiver vazia, autenticação via app password falha com "password incorrect" mesmo com o app password correto ([FTL#2304](https://github.com/pi-hole/FTL/issues/2304)). Fix: `pihole setpassword '<senha>'`.

**Cuidado com o CLI:** `pihole setpassword --help`/`-h` na verdade **define a senha literalmente como `--help`/`-h`** — o script (`SetWebPassword()`) trata qualquer `$2` como nova senha, sem validar flag. Sempre checar a implementação antes de assumir que uma flag existe em comandos que alteram credenciais.

**Correções aplicadas (2026-08-05):** adlists quebradas do bloqueio de redes sociais (apontavam para `blob/` do GitHub em vez de `raw.githubusercontent.com`); retenção de histórico aumentada de 1 para 14 dias; CSP sem `form-action 'self'` restaurada; certificado TLS self-signed sem renovação automática — cron mensal criado (`/etc/cron.d/pihole-tls-renew`).

## AdGuard Home — 203.0.113.22 (CT 103 em quark)

Descoberto em 2026-08-05 durante um bug de DNS (ver corrida abaixo) — até então só o Pi-hole era conhecido no inventário de DNS filtrante.

Acesso SSH: padrão de todo o homelab, porta 22 (só `quark` usa 59022).

**HTTPS configurado em 2026-08-30:** certificado emitido pela `Particula Intermediate CA` (fluxo padrão, ver [pki-ca-openbao.md](../seguranca/pki-ca-openbao.md)), válido 2026-08-31 a 2027-08-31, SAN `DNS:adguard.particula.local, IP:203.0.113.22`. Instalado em `/opt/AdGuardHome/tls/{adguard.crt,adguard.key}`, referenciado em `AdGuardHome.yaml` (`tls.certificate_path`/`private_key_path`), `tls.enabled: true`. `force_https` deixado em `false` (HTTP:80 continua respondendo em paralelo) — não foi pedido redirecionamento forçado. Backup do yaml anterior em `AdGuardHome.yaml.bak-20260830231003` no próprio host. Renovação anual manual (sem cron automático, diferente do Pi-hole).

## Corrida não-determinística entre os dois

Clientes que usam `dnsmasq` local com `no-resolv` + `all-servers` (ex. `/etc/dnsmasq.d/split-dns.conf`), listando Pi-hole (.12) e AdGuard (.22) como upstreams genéricos, sofrem uma corrida: com `all-servers`, o dnsmasq consulta **todos** os upstreams em paralelo e aceita a **primeira resposta que chegar** — sem prioridade fixa.

**Caso concreto (bug do Kompress/Headroom, 2026-08-05):** o AdGuard bloqueava `hf.co` e subdomínios (usado pela Hugging Face para blobs/LFS), retornando `0.0.0.0`; o Pi-hole resolvia normal. Como a resposta do AdGuard geralmente ganhava a corrida, downloads que dependiam desse domínio travavam silenciosamente. Resolvido liberando `*.hf.co` nos dois filtros.

**Como investigar bloqueios/DNS estranhos nesta rede:** sempre checar os **dois** filtros (.12 e .22) e também via `dig @127.0.0.1` (dnsmasq local do host, que é o que containers Docker realmente usam via `127.0.0.11`) — testar um único servidor isoladamente pode não refletir o que o tráfego real recebe.
