# phpIPAM — phpipam.particula.local

LXC 119 em quark, IP 203.0.113.23. Criado em 2026-08-25 via **pacotes nativos** (nginx + php8.4-fpm + mariadb-server), não pelo script community-scripts usado por outros serviços (`pihole`/`adguard`) — decisão pontual dessa sessão, não regra permanente para futuros serviços.

## Instalação

Debian 13.1, unprivileged, 1024MB RAM, hardening SSH completo desde a criação. App via `git clone --recursive` do upstream oficial `phpipam/phpipam`, tag estável (`v1.8.2` no momento da instalação). Schema importado direto de `db/SCHEMA.sql` (sem passar pelo wizard web) — login inicial `admin`/`ipamadmin`, força troca de senha no primeiro acesso (já concluído).

Banco MariaDB local, credenciais geradas via `openssl rand`, salvas **só dentro do container** em `/root/.phpipam_dbpass` (600, root-only).

**Gotcha pós-instalação:** faltava o pacote `php-pear` — o check de requisitos do phpIPAM faz `include_once('PEAR.php')` (pacote de sistema, não os arquivos que já vêm no repo) e bloqueava a tela de login com "PHP extensions are missing" mesmo rodando nginx (mensagem sugeria Apache). Fix: `apt install php-pear` + restart do `php8.4-fpm`.

## HTTPS

Certificado leaf emitido pela CA interna via `openbao.particula.local` (mesmo padrão do resto do domínio, ver [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md)). Nota técnica: conectar direto no IP do OpenBao falha (`VAULT_ADDR=https://127.0.0.1:8200` não funciona remotamente — o certificado não cobre esse nome); usar o hostname/IP real explícito e apontar `VAULT_CACERT` para a raiz baixada do endpoint público sem auth.

## Descoberta de rede (scan)

Funcional desde 2026-08-25. `www-data` (PHP-FPM) precisou da capability `cap_net_raw` no binário `ping` para abrir socket ICMP — mesmo com `exec()` liberado no `php.ini`, sem isso a descoberta falha silenciosamente. Fix: `apt install libcap2-bin` + `setcap cap_net_raw+ep /usr/bin/ping`. Subnet de exemplo `203.0.113.0/24` (já vinha no seed do schema) usada para o primeiro scan — 254 IPs escaneados, 12 hosts novos descobertos com hostname resolvido via PTR reverso (proton/neutron).

Registro DNS `phpipam.particula.local → 203.0.113.23` e PTR reverso criados via API do Technitium, ver [../rede/dns-particula-local.md](../rede/dns-particula-local.md).

## Alerta de instalador

O phpIPAM mostra um aviso na tela enquanto o instalador web não é desativado. Fix: em `/var/www/phpipam/config.php`, `$disable_installer = true;` (backup salvo como `config.php.bak-<data>` antes da mudança).
