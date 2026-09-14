# Roteiro de migração gradual: Technitium DNS → BIND9

Substituição dos dois servidores DNS do homelab (`proton.particula.local` e `neutron.particula.local`) por BIND9 em novos LXC Debian 13, mantendo IPs e hostnames. Baseado no estado documentado em [dr-technitium-proton.md](../rede/dr-technitium-proton.md), [dr-technitium-neutron.md](../rede/dr-technitium-neutron.md) e [dns-particula-local.md](../rede/dns-particula-local.md).

Status: **planejamento, não executado.**

## 1. Objetivo

Substituir os dois servidores Technitium DNS por BIND9, preservando endereços IP e nomes atuais:

| Função | Hostname | Endereço IP |
|---|---|---:|
| DNS primário | `proton.particula.local` | `203.0.113.10` |
| DNS secundário | `neutron.particula.local` | `203.0.113.11` |

A migração é executada em **duas etapas**, não em corte único:

1. Substituição do Technitium secundário pelo BIND9 secundário.
2. Substituição do Technitium primário pelo BIND9 primário, junto com a migração do DHCP para o Kea no OPNsense.

Durante a etapa intermediária, o Technitium primário continua sendo origem das zonas e servidor DHCP, enquanto o BIND9 secundário responde consultas e recebe zonas por AXFR/IXFR — reduz o raio de impacto de cada corte e valida o BIND9 recebendo dados reais antes de ele assumir como fonte.

## 2. Arquitetura por etapa

### 2.1 Estado inicial

| IP | Software | Função |
|---:|---|---|
| `203.0.113.10` | Technitium | DNS primário e DHCP |
| `203.0.113.11` | Technitium | DNS secundário |

### 2.2 Estado intermediário (após Fase 2)

| IP | Software | Função |
|---:|---|---|
| `203.0.113.10` | Technitium | DNS primário e DHCP |
| `203.0.113.11` | BIND9 | DNS secundário |

### 2.3 Estado final (após Fase 6)

| IP | Software | Função |
|---:|---|---|
| `203.0.113.10` | BIND9 | DNS primário e receptor de DDNS |
| `203.0.113.11` | BIND9 | DNS secundário |
| `203.0.113.1` | OPNsense | Gateway, firewall, Kea DHCP e atualizador DDNS |

Sem painel web nos DNS, sem certificado TLS de gestão (a porta 53443/TLS do Technitium deixa de existir) — administração dos BIND9 só via SSH + `rndc`. Gestão do DHCP passa a ser pela GUI do OPNsense (Services → Kea DHCP), não mais pelo painel do Technitium.

## 3. Decisões confirmadas

1. **DHCP.** Sai dos servidores DNS e passa a rodar no **OPNsense**, via **Kea DHCP** (plugin `os-kea`), com **DDNS** (RFC 2136) atualizando o BIND9 primário via TSIG — mantém o comportamento atual (lease vira A/PTR automático). Isso concentra mais funções no OPNsense (gateway + firewall + DHCP + atualizador DDNS).
   > Limitação a registrar: Kea não tem equivalente direto ao `pingCheckEnabled`/`pingCheckRetries` do Technitium (checagem de IP livre via ping antes de conceder lease) — não há paridade exata nesse ponto.
2. **Forwarders.** Hoje o Technitium encaminha para o Quad9 via DNS-over-HTTPS (`https://dns.quad9.local/dns-query`). BIND9 não faz DoH nativamente — usar forwarders tradicionais (`9.9.9.9`, `149.112.112.112`, porta 53). Avaliar DoT separadamente se a versão do BIND9 instalada suportar.
3. **IPs e hostnames preservados**: `203.0.113.10`/`proton.particula.local`, `203.0.113.11`/`neutron.particula.local`. Evita reconfiguração em OPNsense, Pi-hole, AdGuard e qualquer host com DNS fixo.
4. **Chaves TSIG**: duas chaves distintas, `particula-transfer` (AXFR/IXFR) e `particula-ddns` (atualizações DDNS) — ver seção 5. A chave `particula-ddns` é um segredo cross-host (BIND9 + OPNsense); documentar os dois lados.
5. **Numeração do LXC (CT ID)**: definir conforme slot livre no hypervisor (`quark`) no momento da execução — não presumido aqui.

## 4. Fora de escopo

- RPZ/bloqueio de anúncios: já desabilitado no Technitium (`enableBlocking: false`); Pi-hole/AdGuard cobrem isso, continuam como estão.
- DNSSEC assinando zonas próprias: zonas atuais são `Unsigned` no Technitium; não introduzir DNSSEC signing nesta migração.
- Painel web de administração: BIND9 não tem GUI própria; gestão via `rndc` + arquivos de zona + SSH.
- QPM rate limiting por prefixo (feature nativa do Technitium): BIND9 tem `rate-limit` (RRL), mecanismo diferente e não equivalente 1:1 — não replicar granularidade exata, usar RRL básico se necessário depois.

## 5. Pré-requisitos

- Backups já existem em um backup mantido fora deste repositório (settings, zonas, registros, DHCP scope — JSON bruto via API).
- Snapshot Btrfs de proton e neutron antes de qualquer intervenção, mesmo com LXC de staging separados (rollback rápido).
- Acesso ao console dos LXC e do OPNsense, independente do DNS (não depender de resolução de nomes para administrar durante a janela).
- Dois novos LXC Debian 13 criados com snapshots Btrfs e hardening padrão de SSH — convenção do repositório.
- IPs de staging reservados, fora da faixa de produção: BIND9 primário `203.0.113.60`, BIND9 secundário `203.0.113.61`.
- **phpIPAM**: reservar os dois IPs de staging via API (`POST /api/{app_id}/addresses/`), com hostname temporário e nota "staging migração BIND9" — evita que outro serviço pegue o mesmo IP durante os testes. Token de API não está salvo neste repositório (ver [servicos/phpipam.md](../servicos/phpipam.md)), mantida fora deste repositório.
- Configuração do Kea preparada no OPNsense, mas mantida **desabilitada** até a Fase 6.
- Não registrar as chaves TSIG neste repositório.
- Janela de manutenção aprovada; plano de comunicação e responsáveis pela execução definidos.

## 6. Chaves TSIG

Gerar duas chaves diferentes:

| Chave | Finalidade | Sistemas autorizados |
|---|---|---|
| `particula-transfer` | Transferências AXFR/IXFR | Technitium primário (temporariamente) e ambos os BIND9 |
| `particula-ddns` | Atualizações DDNS (A/AAAA/PTR/DHCID) | Kea (OPNsense) e BIND9 primário |

Usar `particula-transfer` já na fase de interoperabilidade com o Technitium (Fase 1) — o BIND9 secundário continua apontando para `203.0.113.10` com a mesma chave depois que o primário for substituído, sem mudança estrutural na configuração do secundário.

```bash
sudo tsig-keygen -a hmac-sha256 particula-transfer
sudo tsig-keygen -a hmac-sha256 particula-ddns
```

Armazenar no BIND9 em arquivos `include`d separados, não inline no `named.conf`:

```text
/etc/bind/keys/particula-transfer.key
/etc/bind/keys/particula-ddns.key
```

```bash
sudo chown root:bind /etc/bind/keys/*.key
sudo chmod 0640 /etc/bind/keys/*.key
```

Não registrar os segredos em documentação, tickets, logs ou repositórios Git. O lado OPNsense da chave `particula-ddns` é mantido fora deste repositório.

## 7. Fase 0 — Provisionar LXC de staging

1. Criar os dois LXC (Debian 13, Btrfs) nos IPs temporários `203.0.113.60`/`.61` — não usar `.10`/`.11` ainda, para não colidir com produção.
2. Hostnames temporários (ex. `bind9-staging-1`/`bind9-staging-2`) — renomeados no corte de cada fase.
3. Reservar os IPs de staging no phpIPAM (ver seção 5).

## 8. Instalação e configuração global do BIND9

Em ambos os LXC:

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-dnsutils
sudo systemctl enable bind9
```

```bash
sudo install -d -o root -g bind -m 0750 /etc/bind/keys
sudo install -d -o bind -g bind -m 0750 /var/lib/bind/zones
```

Os arquivos das zonas dinâmicas do primário ficam em `/var/lib/bind/zones` (não em `/var/cache/bind`), porque o `named` precisa criar journals (`.jnl`) e reescrever esses arquivos com as atualizações DDNS. Zonas secundárias (só leitura via AXFR) usam `/var/cache/bind`.

Exemplo de `/etc/bind/named.conf.options`:

```conf
acl "particula-lan" {
    127.0.0.0/8;
    203.0.113.0/24;
};

options {
    directory "/var/cache/bind";

    listen-on { 127.0.0.1; 203.0.113.60; };  // IP de staging; trocar no corte
    listen-on-v6 { none; };

    recursion yes;
    allow-recursion { particula-lan; };
    allow-query { particula-lan; };
    allow-query-cache { particula-lan; };

    forwarders {
        9.9.9.9;
        149.112.112.112;
    };
    forward only;

    dnssec-validation auto;
    minimal-responses yes;
    max-cache-ttl 604800;
    max-ncache-ttl 300;

    querylog no;  // proton tinha false, neutron tinha true no Technitium — uniformizar, ligar sob demanda ao investigar problema
};
```

Durante o staging, ajustar `listen-on` para o endereço temporário de cada servidor; antes de cada corte, trocar para o endereço definitivo (`.10` ou `.11`). Não autorizar indiscriminadamente todas as redes RFC1918 — restringir `particula-lan` às redes realmente presentes; incluir outras VLANs/VPNs autorizadas explicitamente, se existirem.

## 9. Fase 1 — Preparar o BIND9 secundário

### 9.1 Configurar o Technitium primário

No Technitium `203.0.113.10`:

1. Cadastrar a chave `particula-transfer`.
2. Autorizar AXFR/IXFR para o IP de staging `203.0.113.61`.
3. Configurar NOTIFY para `203.0.113.61`, se disponível.
4. Manter o Technitium secundário `203.0.113.11` funcionando normalmente.
5. Não alterar o DHCP.

### 9.2 Configurar zonas secundárias no BIND9 de staging

```conf
include "/etc/bind/keys/particula-transfer.key";

zone "particula.local" {
    type secondary;
    primaries {
        203.0.113.10 key "particula-transfer";
    };
    file "/var/cache/bind/db.particula.local";
};

zone "1.168.192.in-addr.arpa" {
    type secondary;
    primaries {
        203.0.113.10 key "particula-transfer";
    };
    file "/var/cache/bind/db.1.168.192";
};
```

### 9.3 Validar em staging

```bash
sudo named-checkconf -z
sudo systemctl restart bind9
sudo rndc retransfer particula.local
sudo rndc retransfer 1.168.192.in-addr.arpa
```

```bash
dig @203.0.113.10 particula.local SOA +short
dig @203.0.113.61 particula.local SOA +short
dig @203.0.113.61 particula.local NS
dig @203.0.113.61 proton.particula.local A
dig @203.0.113.61 -x 203.0.113.10
dig @203.0.113.61 google.com A
```

Criar um registro temporário no Technitium primário e confirmar que ele chega ao BIND9 secundário. Remover o registro depois do teste.

### 9.4 Critérios para avançar

- `named-checkconf -z` sem erros.
- Mesmo serial SOA no Technitium primário e no BIND9 secundário.
- Transferência autenticada funcionando.
- Alteração de teste replicada.
- Resolução direta e reversa funcionando.
- Resolução recursiva externa funcionando.
- Ausência de erros TSIG, `REFUSED`, `NOTAUTH` ou `SERVFAIL`.

## 10. Fase 2 — Substituir o servidor secundário

### 10.1 Verificações do corte

1. Congelar alterações nas zonas.
2. Confirmar transferência mais recente para `203.0.113.61`.
3. Comparar os seriais SOA.
4. Confirmar acesso ao console de ambos os LXC.
5. Criar snapshot do Technitium secundário.

### 10.2 Executar o corte

1. Parar o Technitium secundário.
2. Desabilitar sua inicialização automática.
3. Desligar completamente o LXC antigo (não apagar — rollback).
4. Confirmar que `203.0.113.11` não responde mais.
5. Parar o BIND9 secundário de staging.
6. Alterar seu IP de `203.0.113.61` para `203.0.113.11`.
7. Alterar o hostname para `neutron.particula.local`.
8. Atualizar `/etc/hosts` e configuração de rede.
9. Alterar `listen-on` para `203.0.113.11`.
10. Validar configuração com `named-checkconf -z`.
11. Atualizar no Technitium primário a autorização de transferência e o NOTIFY de `.61` para `.11`.
12. Iniciar o BIND9 secundário.
13. Forçar nova transferência das duas zonas.
14. Enviar gratuitous ARP, se necessário.

### 10.3 Validação pós-corte

```bash
dig @203.0.113.10 particula.local SOA +short
dig @203.0.113.11 particula.local SOA +short
dig @203.0.113.11 proton.particula.local A
dig @203.0.113.11 -x 203.0.113.10
dig @203.0.113.11 google.com A
dig @203.0.113.12 proton.particula.local A
dig @203.0.113.22 proton.particula.local A
```

```bash
sudo journalctl -u bind9 --since "15 minutes ago"
```

## 11. Fase 3 — Período intermediário de observação

Manter temporariamente:

- Technitium primário/DHCP em `203.0.113.10`.
- BIND9 secundário em `203.0.113.11`.

Durante algumas horas ou até um dia, validar:

- Consultas autoritativas no BIND9 secundário.
- Recursão externa.
- Recebimento de NOTIFY.
- AXFR/IXFR depois de mudanças de zona.
- Registros criados pelo DHCP do Technitium.
- Persistência da zona após reinicialização do BIND9.
- Funcionamento por meio do Pi-hole e do AdGuard.
- Ausência de divergência de serial.

Não substituir o primário enquanto houver falhas nessa etapa.

## 12. Fase 4 — Preparar o BIND9 primário

### 12.1 Exportar as zonas

Fazer um novo export das zonas e registros do Technitium — o export de o backup mantido fora deste repositório não deve ser considerado definitivo, pois podem ter ocorrido mudanças depois dele. Conferir a lista completa contra o JSON de backup antes de aplicar; a tabela abaixo é um resumo, o export bruto é fonte de verdade.

Converter e salvar as zonas em `/var/lib/bind/zones/db.particula.local` e `/var/lib/bind/zones/db.1.168.192`.

Exemplo de `db.particula.local` baseado no último export conhecido:

```text
$TTL 3600
@   IN SOA  proton.particula.local. hostadmin.particula.local. (
                2026091003 ; serial — ver 12.2
                900        ; refresh
                300        ; retry
                604800     ; expire
                900 )      ; negative TTL

    IN NS   proton.particula.local.
    IN NS   neutron.particula.local.

node1       IN A    203.0.113.31
node2       IN A    203.0.113.32
node3       IN A    203.0.113.33
taquion     IN A    203.0.113.171
wifi        IN A    203.0.113.2
opnsense    IN A    203.0.113.1
pfsense     IN A    203.0.113.1
```

Não incluir `moto-g86-5G` estaticamente — esse registro é dinâmico (criado pelo DHCP, TTL 15m), recriado automaticamente pela DDNS quando o dispositivo renovar o lease (Fase 6).

Zona reversa `db.1.168.192` — mesmo SOA/NS, um PTR por A record acima:

```text
10  IN PTR proton.particula.local.
11  IN PTR neutron.particula.local.
5   IN PTR quark.particula.local.
12  IN PTR pihole.particula.local.
13  IN PTR netbird.particula.local.
15  IN PTR openbao.particula.local.
16  IN PTR syslog.particula.local.
17  IN PTR qbittorrent.particula.local.
18  IN PTR freshrss.particula.local.
19  IN PTR vaultwarden.particula.local.
22  IN PTR adguard.particula.local.
23  IN PTR phpipam.particula.local.
25  IN PTR immich.particula.local.
31  IN PTR node1.particula.local.
32  IN PTR node2.particula.local.
33  IN PTR node3.particula.local.
171 IN PTR taquion.particula.local.
2   IN PTR wifi.particula.local.
1   IN PTR opnsense.particula.local.
```

### 12.2 Ajustar o serial SOA

```bash
dig @203.0.113.10 particula.local SOA +short
dig @203.0.113.10 1.168.192.in-addr.arpa SOA +short
```

Os seriais do novo BIND9 primário devem ser superiores aos últimos seriais do Technitium.

### 12.3 Configurar o primário

```conf
include "/etc/bind/keys/particula-transfer.key";
include "/etc/bind/keys/particula-ddns.key";

zone "particula.local" {
    type primary;
    file "/var/lib/bind/zones/db.particula.local";

    allow-transfer {
        key "particula-transfer";
    };

    also-notify {
        203.0.113.11;
    };

    update-policy {
        grant particula-ddns zonesub A AAAA DHCID;
    };
};

zone "1.168.192.in-addr.arpa" {
    type primary;
    file "/var/lib/bind/zones/db.1.168.192";

    allow-transfer {
        key "particula-transfer";
    };

    also-notify {
        203.0.113.11;
    };

    update-policy {
        grant particula-ddns zonesub PTR DHCID;
    };
};
```

`update-policy` com `grant zonesub` restringe o DDNS por tipo de registro (A/AAAA/PTR/DHCID) — mais restrito que um `allow-update { key ...; }` genérico, que autorizaria qualquer tipo de registro na zona.

### 12.4 Validar os arquivos

```bash
sudo named-checkzone particula.local /var/lib/bind/zones/db.particula.local
sudo named-checkzone 1.168.192.in-addr.arpa /var/lib/bind/zones/db.1.168.192
sudo named-checkconf -z
```

## 13. Fase 5 — Preparar o Kea no OPNsense

DHCP não roda em nenhum LXC BIND9 — o plugin `os-kea` é instalado/habilitado no OPNsense (System → Firmware → Plugins). Configurar sem habilitar ainda:

| Parâmetro | Valor |
|---|---|
| Interface | LAN |
| Subnet | `203.0.113.0/24` |
| Pool | `203.0.113.200`–`203.0.113.254` |
| Gateway | `203.0.113.1` |
| DNS entregues | `203.0.113.12`, `203.0.113.22` (Pi-hole/AdGuard — igual hoje, não os BIND9) |
| Domain name | `particula.local` |
| Lease | `86400` segundos (igual ao Technitium) |
| DNS primário para DDNS | `203.0.113.10` |
| Zona direta | `particula.local.` |
| Zona reversa | `1.168.192.in-addr.arpa.` |
| Qualifying suffix | `particula.local.` |
| TSIG | `particula-ddns`, HMAC-SHA256 |

Configurar o DDNS Agent para escutar somente em `127.0.0.1`, porta `53001`. Sem reservas fixas — não havia nenhuma cadastrada no Technitium, nada a migrar aqui. Sem paridade exata para `pingCheckEnabled`/`pingCheckRetries` do Technitium (ver seção 3, decisão 1).

Antes da migração do primário, reduzir temporariamente o lease do Technitium para 15–30 minutos e aguardar a renovação dos leases antigos — reduz o tempo necessário para os clientes passarem a produzir DDNS pelo Kea.

## 14. Fase 6 — Substituir o servidor primário

### 14.1 Verificações antes do corte

1. Confirmar que o BIND9 secundário está saudável.
2. Congelar alterações nas zonas.
3. Exportar novamente zonas, registros e leases.
4. Confirmar o serial final do Technitium.
5. Garantir que os arquivos do BIND9 usam serial superior.
6. Executar `named-checkconf -z` no novo primário.
7. Confirmar que o Kea está configurado, mas desabilitado.
8. Criar snapshot do Technitium primário.
9. Confirmar acesso ao console do OPNsense.

### 14.2 Executar o corte

1. Desabilitar o DHCP no Technitium.
2. Parar o serviço Technitium.
3. Desabilitar sua inicialização automática.
4. Desligar completamente o LXC antigo (não apagar — rollback).
5. Confirmar que `203.0.113.10` não responde mais.
6. Parar o BIND9 primário de staging.
7. Alterar seu IP de `203.0.113.60` para `203.0.113.10`.
8. Alterar o hostname para `proton.particula.local`.
9. Atualizar `/etc/hosts` e a configuração de rede.
10. Alterar `listen-on` para `203.0.113.10`.
11. Executar `named-checkconf -z`.
12. Iniciar o BIND9 primário.
13. Confirmar que ele responde pela zona direta e reversa.
14. No secundário, forçar a transferência das duas zonas.
15. Comparar os seriais SOA.
16. Enviar gratuitous ARP, se necessário.
17. Habilitar o Kea DHCPv4.
18. Habilitar o Kea DDNS Agent.
19. Renovar o lease de um cliente de teste.
20. Confirmar a criação e replicação dos registros A e PTR.

O BIND9 secundário continua usando `primaries { 203.0.113.10 key "particula-transfer"; };` — como o IP e a chave não mudam, não é necessária alteração estrutural na configuração do secundário.

## 15. Fase 7 — Validação final

### 15.1 Autoridade e serial

```bash
dig @203.0.113.10 particula.local SOA +short
dig @203.0.113.11 particula.local SOA +short
dig @203.0.113.10 1.168.192.in-addr.arpa SOA +short
dig @203.0.113.11 1.168.192.in-addr.arpa SOA +short
```

### 15.2 Registros internos

```bash
dig @203.0.113.10 proton.particula.local A
dig @203.0.113.11 proton.particula.local A
dig @203.0.113.10 -x 203.0.113.10
dig @203.0.113.11 -x 203.0.113.10
```

### 15.3 Recursão externa

```bash
dig @203.0.113.10 google.com A
dig @203.0.113.11 google.com A
```

### 15.4 Segurança da transferência

Sem chave, o AXFR deve ser rejeitado:

```bash
dig @203.0.113.10 particula.local AXFR
```

Com a chave correta, deve funcionar:

```bash
dig -k /etc/bind/keys/particula-transfer.key @203.0.113.10 particula.local AXFR
```

### 15.5 DHCP e DDNS

No cliente de teste, renovar o lease conforme o sistema operacional. Depois validar:

```bash
dig @203.0.113.10 cliente.particula.local A
dig @203.0.113.10 -x IP_DO_CLIENTE
dig @203.0.113.11 cliente.particula.local A
dig @203.0.113.11 -x IP_DO_CLIENTE
```

Confirmar no cliente: endereço dentro do pool, gateway `203.0.113.1`, DNS `203.0.113.12`/`203.0.113.22`, domínio de pesquisa correto, tempo de lease correto, registro A criado, registro PTR criado, registros replicados no secundário.

### 15.6 Dependentes

Validar: Pi-hole usando os novos servidores, AdGuard usando os novos servidores, resolução no próprio OPNsense, hosts com DNS configurado diretamente para `.10` ou `.11`, serviços internos críticos, resolução depois de reiniciar cada BIND9 individualmente.

## 16. Critérios de sucesso

A migração está concluída quando:

- Os LXC Technitium estiverem desligados.
- Não houver endereços IP duplicados.
- Os dois BIND9 responderem por TCP e UDP.
- Os seriais SOA forem iguais.
- AXFR/IXFR autenticado funcionar.
- AXFR sem TSIG for rejeitado.
- Recursão estiver limitada às redes autorizadas.
- O Kea conceder leases corretamente.
- DDNS criar A e PTR no primário.
- Os registros dinâmicos forem replicados no secundário.
- Pi-hole e AdGuard resolverem nomes internos e externos.
- Reinicializações individuais não causarem perda das zonas ou dos registros dinâmicos.
- Não houver erros críticos nos logs durante o período de observação.

## 17. Condições para interromper a migração

Interromper e executar rollback se ocorrer qualquer um dos seguintes eventos:

- Conflito de IP.
- BIND9 não iniciar ou apresentar erros de zona.
- Divergência persistente de serial.
- Falha de AXFR/IXFR.
- Respostas `SERVFAIL` para registros válidos.
- Pi-hole ou AdGuard não conseguirem consultar os BIND9.
- Kea não entregar gateway ou DNS corretos.
- DDNS não criar A e PTR.
- Recursão ficar disponível para redes não autorizadas.
- Tempo máximo da janela de manutenção ser atingido.

## 18. Rollback da troca do secundário

1. Parar e desligar o BIND9 `203.0.113.11`.
2. Confirmar que o endereço está livre.
3. Ligar o Technitium secundário antigo.
4. Confirmar sua sincronização com o Technitium primário.
5. Validar resolução interna e externa.
6. Confirmar o funcionamento por Pi-hole e AdGuard.

O DNS primário e o DHCP não terão sido modificados nessa etapa.

## 19. Rollback da troca do primário

1. Desabilitar o Kea DHCPv4.
2. Desabilitar o Kea DDNS Agent.
3. Parar e desligar o BIND9 primário `203.0.113.10`.
4. Confirmar que o endereço está livre.
5. Ligar o Technitium primário antigo.
6. Reabilitar o DHCP do Technitium.
7. Confirmar que o BIND9 secundário transfere a zona do Technitium usando `particula-transfer`.
8. Forçar a transferência das zonas no secundário.
9. Validar DNS, DHCP, Pi-hole e AdGuard.

Se o BIND9 secundário também apresentar inconsistência, desligá-lo e reativar o Technitium secundário antigo.

## 20. Pós-migração

- Monitorar os serviços por 7 a 14 dias.
- Manter os LXC Technitium desligados (não apagados) durante esse período — rollback rápido se necessário.
- **phpIPAM** (via API, não é automático — Kea/BIND9 não fala com o phpIPAM):
  - Editar os registros de `.10` e `.11`: descrição/tag de "Technitium DNS" para "BIND9 DNS" (primário/secundário).
  - Liberar os IPs de staging (`.60`/`.61`) reservados como "staging migração BIND9".
  - Adicionar nota no range `203.0.113.200`–`.254` indicando que o DHCP passou de Technitium (proton) para Kea (OPNsense) — evita confusão futura de quem só olha o phpIPAM.
  - Leases dinâmicos individuais não precisam de entrada manual no phpIPAM — isso é escopo do DDNS/BIND9, não do IPAM.
- Atualizar a documentação de recuperação de desastre (`dr/RECUPERACAO-DESASTRE.md`, `rede/dr-technitium-*.md`).
- Remover do Technitium a chave `particula-transfer` antes do arquivamento definitivo.
- Arquivar os backups do Technitium e mover a documentação de DR do Technitium para uma nota de "descomissionado em \<data\>", mantendo o histórico.
- Somente depois do período de estabilidade, descomissionar (apagar) os LXC antigos.

## 21. Observações de segurança

- TSIG fornece autenticação e integridade, mas **não criptografa** AXFR, IXFR ou DDNS.
- Restringir TCP/UDP 53 às redes autorizadas.
- Permitir transferência somente com a chave `particula-transfer`.
- Permitir DDNS somente com a chave `particula-ddns`, restringindo os tipos permitidos em `update-policy` (ver seção 12.3).
- Restringir SSH à rede administrativa.
- Manter o `rndc` acessível somente localmente.
- Não registrar consultas permanentemente (`querylog no`), salvo necessidade operacional pontual.
- Avaliar DNS-over-TLS para os forwarders se a versão instalada do BIND9 oferecer suporte.
- **DDNS por chave única**: se a chave `particula-ddns` vazar, qualquer host na LAN que a conheça pode atualizar registros — mesmo risco que o Technitium já tinha com `dnsUpdates: true`, não é uma regressão.
- A zona `.local` pode conflitar com mDNS/Avahi (RFC 6762). Mantê-la preserva compatibilidade com o ambiente atual, mas uma futura migração para `home.arpa` deve ser avaliada separadamente.
- **SOA rname inconsistente hoje** (`defaultResponsiblePerson` difere entre proton/neutron nas settings do Technitium, mas o SOA record real usa `hostadmin@particula.local` nos dois) — esta migração resolve isso por construção, usando `hostadmin.particula.local.` fixo nas duas zonas.
