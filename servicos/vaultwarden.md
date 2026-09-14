# Cofre de senhas: Bitwarden → Vaultwarden

## Estado atual

`vaultwarden.particula.local` (203.0.113.19, LXC 110 em quark, **Alpine Linux 3.24.1**) é o cofre de produção desde 2026-08-23, substituindo o `bitwarden.particula.local` (VM KVM antiga, VMID 113), apagada (`qm destroy --purge`) em 2026-09-09.

**Motivo da migração:** Bitwarden oficial é pesado para escala de homelab pessoal (11 containers Docker, incluindo mssql). Vaultwarden é a alternativa leve (Rust, binário único) compatível com o mesmo protocolo/clientes.

## Vaultwarden (atual)

- SO real é Alpine (apesar de o script de provisionamento community-scripts ter sido rodado com `var_os='debian'`) — não presumir Debian neste host.
- Serviço nativo via `apk`, não Docker — OpenRC (`/etc/init.d/vaultwarden`, `rc-service vaultwarden {status,restart}`).
- Config em `/etc/conf.d/vaultwarden`: `DATA_FOLDER=/var/lib/vaultwarden`, sem proxy reverso — Vaultwarden serve HTTPS direto via Rocket/rustls na **porta 8000** (não 443): `https://vaultwarden.particula.local:8000`.
- Certificado TLS emitido pela CA interna (`pki_int`, role `particula-local`), válido até 2027-08-23, aplicado por cima dos arquivos self-signed originais (mesmo nome, para não precisar editar `ROCKET_TLS`).
- Hardening SSH: host nasceu com config frouxa (`PermitRootLogin prohibit-password`, `PasswordAuthentication yes`, parâmetros soltos) — corrigido via drop-in `/etc/ssh/sshd_config.d/99-hardening.conf` (Alpine já inclui esse diretório por padrão). Único host do homelab hardenizado via drop-in em vez de editar `sshd_config` direto — repetir esse método se outro host Alpine aparecer.
- Migração de dados: exportação JSON do Bitwarden antigo, importada pelo navegador no Vaultwarden novo. Extensão do Chrome precisa apontar para a URL **com a porta** (`:8000`), senão dá "Failed to fetch".

## Bitwarden (host antigo, histórico)

VM KVM `bitwarden.particula.local` (`203.0.113.21`, VMID 113 em `quark`) — instância self-hosted oficial do Bitwarden (não Vaultwarden), desativada em 2026-08-23 e substituída pelo LXC acima. O projeto próprio (`git@github.com:abreuferr/bitwarden.git`) foi removido por redundância; este é o histórico consolidado.

**Infra:** 2 vCPU, 3.8GB RAM, 20GB disco. Debian 13 (trixie) com raiz **ext4** (não Btrfs — provisionada antes desse padrão ser adotado, nunca convertida). Rede `eth0` = `203.0.113.21/24`, containers Docker em `docker0`/`br-*` (172.17-19.0.0/16). Aplicação: Bitwarden `2026.8.0` em 11 containers (`nginx`, `admin`, `sso`, `notifications`, `events`, `api`, `identity`, `icons`, `web`, `mssql`, `attachments`).

**Por que não migrou pra LXC:** avaliado e descartado (2026-08-23) — `mssql` (SQL Server) é sensível a virtualização aninhada, e Docker em LXC exige container privilegiado (`nesting=1`, `keyctl=1`), o que enfraqueceria o isolamento de uma stack que guarda senhas. Ganho de recursos seria pequeno (VM já enxuta). Essa é justamente a razão pela qual, quando a substituição por Vaultwarden apareceu, ela rodou nativa (sem Docker) — elimina o problema pela raiz.

### Instalação inicial (~2026-04-02)

Instalador oficial (`bitwarden.sh`, via `curl` de `func.bitwarden.com`) em `/opt/bitwarden`, usuário de sistema dedicado `bitwarden`. Domínio `bitwarden.particula.local`, sem Let's Encrypt — certificado próprio (CA ad-hoc via `openssl genrsa`/`req -x509`, SAN com `DNS:bitwarden.particula.local` + `IP:203.0.113.21`). Banco `vault` (mssql, containerizado). Durante o boot, `bitwarden-nginx` aparece brevemente em `Restarting` enquanto os demais containers sobem (`health: starting` até o SQL Server terminar de inicializar) — comportamento transitório esperado, não é falha.

### Reemissão do certificado via PKI interna (2026-08-04)

Certificado antigo era assinado direto pela raiz `Particula CA` (não pela intermediária) — fora do padrão do resto do domínio. Reemitido via `pki_int` (OpenBao, `Particula Intermediate CA`), mesma SAN, válido 04/ago/2026→04/ago/2027. Arquivos trocados em `/opt/bitwarden/bwdata/ssl/bitwarden.particula.local/` (dono `bitwarden:bitwarden`): `certificate.crt` (644), `ca.crt` (644), `private.key` (600). Aplicado via `sudo cp` + `sudo /opt/bitwarden/bitwarden.sh restart`.

### Troubleshooting: Chrome "conexão não segura" (resolvido 2026-08-04)

Causa: raiz `Particula CA` não marcada como confiável no navegador (normal para CA privada nova). Chrome no Linux **não** lê `/etc/ssl/certs` do sistema — usa Chrome Root Store/NSS próprio. Fix: importar a raiz via `chrome://settings/certificates` → Autoridades → Importar `.pem` → marcar "confiar para identificar sites" (ou `certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Particula CA" -i <arquivo>`).

### Hardening SSH root (2026-08-23)

`PermitRootLogin` estava em `prohibit-password` (default OpenSSH) e `/root/.ssh/authorized_keys` tinha uma chave de acesso cadastrada com `command=` forçado (padrão de imagem cloud Debian — só imprime aviso e desconecta, sem shell real). Endurecido: `PermitRootLogin no` explícito em `sshd_config`, `authorized_keys` de root zerado. Acesso root via SSH ficou impossível (nem chave nem senha); `sudo` de um usuário administrativo é o único caminho pra root. Mesmo hardening aplicado em paralelo em `ca.particula.local`.

### Troubleshooting: acesso só funcionava em aba anônima do Chrome (resolvido 2026-08-10)

Causa real: conflito com a extensão **Dark Reader** no perfil normal do Chrome (extensões costumam ficar desativadas por padrão em aba anônima, mascarando o problema). Lado do servidor foi descartado como causa (sem erros de TLS nos logs do nginx, HSTS correto, sessão funcional).

### Update 2026.7.2 → 2026.8.0 (2026-08-23)

`bitwarden.sh update` reportou "Update not needed" até rodar `bitwarden.sh updateself` primeiro. Update aplicado com `sudo su - bitwarden -c 'yes | /opt/bitwarden/bitwarden.sh update'` — a sessão SSH travou (exit 139) no meio do `docker pull` por volume de output, mas o processo continuou no servidor até o fim (confirmado depois, containers voltaram `healthy`). Rodar updates assim de novo: considerar `nohup`/log em arquivo em vez de acompanhar streaming direto.

**Padrão recorrente pós-update — web vault não loga, extensão sim:** o navegador mantém em cache o `index.html` antigo, que referencia hash de bundle JS que não existe mais (404, app trava sem terminar de carregar). A extensão do Chrome não é afetada (código embutido). Fix: hard refresh (`Ctrl+Shift+R`); se não bastar, limpar dados do site em `chrome://settings/content/all`. Conferir isso primeiro em updates futuros do Vaultwarden, antes de investigar credencial/servidor.
