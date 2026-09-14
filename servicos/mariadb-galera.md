# MariaDB Galera Cluster — node1/node2/node3.particula.local

3 LXCs em quark (120/121/122), Debian 13, 2 cores/2048MB RAM/16G disco cada, criados em 2026-09-09. IPs estáticos 203.0.113.31/.32/.33, A record + PTR registrados via API do Technitium em proton (ver [../rede/dns-particula-local.md](../rede/dns-particula-local.md)). Acesso administrativo com hardening de SSH completo desde a criação.

Motivo da escolha MariaDB (não MySQL Oracle): Galera Cluster (wsrep) não é suportado nativamente pelo MySQL Server — só por MariaDB (nativo nos repos Debian) ou Percona XtraDB Cluster (repo de terceiros). MariaDB 11.8.6 + `galera-4` já vêm nos repos oficiais do Debian 13.

## Instalação

Pacotes `mariadb-server mariadb-backup galera-4 rsync` via apt em cada nó. Template LXC nasce sem `sudo` — precisou `apt install sudo` antes de configurar sudo para o usuário administrativo.

Config em `/etc/mysql/mariadb.conf.d/60-galera.cnf` (root:mysql, 640 — contém a senha do usuário SST em texto claro, mesmo padrão de exposição do `wsrep_sst_auth` do MariaDB upstream):

- `wsrep_cluster_name = particula_galera`, `wsrep_cluster_address = gcomm://203.0.113.31,203.0.113.32,203.0.113.33`.
- `wsrep_sst_method = mariabackup`, autenticado por `sst_user@localhost` (privilégios `RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT` — **não** existe `BACKUP ADMIN` no MariaDB, é privilégio exclusivo do MySQL 8; tentar conceder dá erro de sintaxe).
- `bind-address = 0.0.0.0` (padrão Debian é `127.0.0.1`, sobrescrito aqui).

## TLS (grupo, SST e cliente-servidor)

Certificados emitidos pela CA interna (`openbao.particula.local`, ver [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md)), um leaf por nó (`CN=nodeN.particula.local`, `ip_sans` do respectivo IP, TTL 8760h), token escopado (policy só `create`/`update` em `pki_int/issue/particula-local`, TTL 15min) — revogado (`revoke-self`) após uso. Arquivos em `/etc/mysql/ssl/` em cada nó: `server.crt`/`ca-chain.pem` (644, legível por qualquer usuário local — são dados públicos) e `server.key` (640 root:mysql). Diretório `/etc/mysql/ssl` em 755 (750 impede cliente sem grupo `mysql` de ler até o `ca-chain.pem` público).

**Gotcha CA incompleta:** o campo `ca_chain` retornado por `pki_int/issue/...` trouxe só a intermediária (`Particula Intermediate CA`), sem a raiz (`Particula CA`) — verificação falhava com `TLS/SSL error: unable to get local issuer certificate` no cliente e `tlsv1 alert unknown ca` na comunicação de grupo (gcomm). Fix: baixar a raiz à parte (`GET /v1/pki_root/ca/pem`, sem auth) e concatenar no fim do `ca-chain.pem` de cada nó.

Três camadas de TLS configuradas:
1. **Comunicação de grupo (gcomm, portas 4567/4568):** `wsrep_provider_options = "socket.ssl_key=...;socket.ssl_cert=...;socket.ssl_ca=..."` — confirmado nos logs (`ssl://<ip>:4567`).
2. **SST (mariabackup + socat):** seção `[sst]` com `tca`/`tcert`/`tkey` **exige também `encrypt = 4`** — sem essa linha o SST roda em claro mesmo com os caminhos de certificado preenchidos (log mostra `MODE='DISABLED'` até a diretiva ser adicionada). Validado forçando full-SST (datadir do node3 apagado) — log passou a mostrar `Using openssl based encryption with socat`.
3. **Cliente-servidor:** `ssl_cert`/`ssl_key`/`ssl_ca` em `[galera]`/`[mysqld]`. Validado com usuário de teste `REQUIRE SSL` conectando via TCP entre nós (`Ssl_cipher = TLS_AES_256_GCM_SHA384`).

**Gotcha cliente local:** o cliente `mysql` moderno (11.x) tenta TLS por padrão mesmo em socket Unix local e falha a verificação contra a CA interna — usar `mysql --skip-ssl` para comandos administrativos via `sudo mysql` no próprio host.

## Bootstrap do cluster

Ordem: parar `mariadb` nos 3 nós → `galera_new_cluster` só no primeiro nó (node1) → criar `sst_user` (precisa existir **antes** dos outros nós pedirem SST ao donor) → `systemctl start mariadb` em node2 e node3 (entram via IST/SST).

**Reiniciar um nó já bootstrapado com `systemctl restart` (em vez de `galera_new_cluster`) falha** (`failed to reach primary view`) se ele for o único membro ativo no momento — precisa `galera_new_cluster` de novo, não um restart simples, quando não há outro nó Primary component ativo.

Estado atual (2026-09-10): `wsrep_cluster_size = 3`, todos `Synced`/`Primary`. Réplica síncrona multi-master testada (escrita em node1, leitura instantânea em node2/node3).

## Segredos

- Senha do `sst_user@localhost`, usada em `wsrep_sst_auth` no `60-galera.cnf` dos 3 nós.

**Decisão:** root do MariaDB fica só em `unix_socket`/`auth_socket` (acesso local via `sudo mysql`), sem senha de rede — decisão consciente do usuário, não pendência. Senha gerada anteriormente para esse fim foi descartada (`shred -u`), nunca chegou a ser aplicada.

Nenhuma chave privada ou resposta bruta da API do OpenBao foi mantida fora dos nós de destino (`shred -u` nas cópias locais usadas só para transferência).

## Firewall

`nftables` habilitado nos 3 nós (`/etc/nftables.conf`, backup do ruleset anterior salvo antes de sobrescrever), política padrão `drop` na cadeia `input`. Só aceita, de origem `203.0.113.0/24`: SSH (22/tcp), MariaDB (3306/tcp), replicação Galera (4567/tcp+udp), IST (4568/tcp), SST (4444/tcp), além de ICMP echo-request e tráfego de conexões já estabelecidas. Testado primeiro em modo transiente (`nft -f`, sem persistir) em cada nó, validado SSH + cluster antes de gravar no arquivo carregado no boot.

## Validação de replicação multi-nó

Escrita testada nos 3 sentidos: insert a partir do node2 propagou para node1/node3; update a partir do node3 propagou para node1/node2. Confirma multi-master real, não só réplica a partir de um nó fixo.

## Certificados — validação por nó

SAN conferido nos 3 (`DNS:nodeN.particula.local` + `IP Address:203.0.113.3N`), validade idêntica nos 3 (2026-09-10 → 2027-09-10).

## Pendências

- **Reemissão dos certificados TLS dos 3 nós antes de 2027-09-10** (validade atual: 2026-09-10 → 2027-09-10, TTL 8760h). Risco: se vencerem, conexões TLS já estabelecidas continuam de pé (sem sintoma visível), mas qualquer evento que force novo handshake (reboot, `systemctl restart mariadb`, IST/SST) falha e o nó não consegue reentrar no cluster (`failed to reach primary view`) até o certificado ser renovado. Afeta as 3 camadas de TLS: gcomm, SST e cliente-servidor. Ver detalhe da cadeia (raiz `Particula CA` até 2036, intermediária `Particula Intermediate CA` até 2031) em [../seguranca/pki-ca-openbao.md](../seguranca/pki-ca-openbao.md). Agendar lembrete de reemissão ~1 mês antes do vencimento.
