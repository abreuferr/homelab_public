# FileBrowser Quantum — qbittorrent.particula.local

Instalado como binário nativo (`/usr/local/bin/filebrowser`, script community-scripts), não Docker. Systemd unit `filebrowser.service` (`User=root`, `WorkingDirectory=/usr/local/community-scripts`), config em `/usr/local/community-scripts/fq-config.yaml`, banco `database.db` no mesmo diretório.

## Bug de instalação: autenticação desabilitada (corrigido 2026-08-31)

O script de instalação deixou `auth.methods.noauth: true` no config — acesso liberado sem login para qualquer um na rede, independente de senha de usuário. Corrigido trocando por `auth.methods.password.enabled: true`. Backup do config antigo em `fq-config.yaml.bak-20260831-093134` no mesmo diretório. Conferir esse campo em qualquer reinstalação futura via community-scripts.

**Não usar** `auth.methods.password.adminPassword` no config — esse campo reseta a senha do admin a cada boot do serviço, sobrescrevendo qualquer troca manual.

## Reset de senha via CLI — gotchas

Comando: `filebrowser set -u <usuario>,<senha> -a -c <config>` (`-a` = cria/marca como admin). **Não** é `filebrowser set user -u ...` — apesar da mensagem de erro do parser sugerir `set user`, o subcomando correto é `set` direto com as flags.

Dois cuidados que já causaram banco espúrio:
- O CLI resolve `database.db` **relativo ao diretório de trabalho do processo**, não ao caminho do `-c`. Rodar fora de `/usr/local/community-scripts` cria um `database.db` novo e vazio no CWD errado em vez de editar o banco real — sempre `cd /usr/local/community-scripts` antes (ou usar `FILEBROWSER_DATABASE` absoluto).
- O SQLite fica lockado enquanto o `filebrowser.service` está rodando (`the database is locked`). É preciso `systemctl stop filebrowser`, rodar o `set`, depois `systemctl start filebrowser`.

Credenciais atuais gerenciadas fora deste repositório (usuário `admin`, resetadas 2026-08-31).
