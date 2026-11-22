# FreshRSS

`freshrss.particula.local` (203.0.113.18), Debian, Apache2 + PHP, instalação em `/opt/freshrss` (DocumentRoot `/opt/freshrss/p`). Autenticação `form`.

## API (apps móveis, ex.: Capy Reader)

- `api_enabled => true` em `/opt/freshrss/data/config.php` (habilitado em 2026-11-22).
- Senha de API por usuário, separada da senha de login. **Definir pela UI** (Configurações → Perfil → Senha da API). A CLI `cli/update-user.php --api-password '<senha>'` expõe a senha em `ps` e no histórico do shell; evitar. O valor não fica neste repositório.
- Endpoint Google Reader: `https://freshrss.particula.local/api/greader.php` (responde `OK` quando ativo).
- No app: servidor `https://freshrss.particula.local`, usuário e senha de API.

## HTTPS

- Vhost `*:443` em `/etc/apache2/sites-available/freshrss-ssl.conf` (`a2enmod ssl`, `a2ensite freshrss-ssl`). TLS 1.2 e 1.3. O vhost `*:80` faz `Redirect permanent /` para `https://freshrss.particula.local/` (2026-11-22), para que login e senha de API nunca trafeguem em HTTP.
- Certificado emitido pela CA interna (`pki_int`, role `particula-local`, `CN=freshrss.particula.local`, IP SAN 203.0.113.18, TTL 8760h, vence em 2027-11-22). Ver [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md).
- Arquivos em `/etc/ssl/freshrss/`: `server.crt` e `ca-chain.pem` (644), `server.key` (600 root:root). `ca-chain.pem` inclui a raiz, buscada à parte em `/v1/pki_root/ca/pem` (gotcha de `ca_chain` incompleta já documentado em [mariadb-galera.md](mariadb-galera.md)).
- Emissão feita com token escopado (policy `pki-issue-https` — ainda em uso aqui; não remover sem trocar este procedimento por `pki-operator`; TTL 15 min, `use-limit=2`). O token de login foi revogado com `revoke-self`, o que revoga também o filho.
- Cofre selado na emissão: foi preciso desselar (2 de 3 unseal keys, custódia em [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md)) antes do login. **Selar de novo após emitir** (`bao operator seal`). Login via `BAO_ADDR=https://203.0.113.15:8200`, pois `127.0.0.1` não consta no SAN.
- **Renovar antes de 2027-11-22.**

## Cliente Android

O celular precisa confiar na raiz `Particula CA` (Configurações → Segurança → Instalar certificado → Certificado de CA). Sem isso o Capy Reader recusa o certificado. O acesso externo exige a VPN ativa.
