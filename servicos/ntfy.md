# ntfy.sh — notificações push do homelab

Serviço [ntfy.sh](https://ntfy.sh) (hospedado, conta criada em 2026-09-05) usado como canal único de alerta pra scripts e serviços do homelab, no lugar de e-mail — o postfix de `quark` é `loopback-only` (sem `relayhost`), então qualquer alerta que dependesse de e-mail de cron nunca chegava a lugar nenhum (ver histórico em [../infra/quark-proxmox.md](../infra/quark-proxmox.md)). Regra: **todo alerta do homelab deve usar ntfy**, não e-mail de cron.

## Tópicos

Um tópico ntfy é público — quem souber o nome pode publicar ou assinar. Por isso o valor real de cada tópico só existe nos scripts, no host onde roda, nunca neste repositório (é segredo de baixo risco: pior caso é alguém publicar lixo no tópico, não um vazamento de dado sensível).

- **Geral do homelab** (notificações e alertas de scripts, criado 2026-09-05): usado por `/usr/local/sbin/ntfy-alert.sh` em `quark`.
- **Temperatura de `quark`** (criado 2026-09-05, dedicado — não usa o geral acima, ver justificativa em [../infra/quark-proxmox.md](../infra/quark-proxmox.md)): usado só por `/usr/local/sbin/temp-monitor.sh`.
- **Alertas do Claude Code** (definido pelo usuário em 2026-09-06): tópico dedicado para o Claude usar quando precisar notificar algo fora do fluxo dos scripts do host. Valor mantido fora deste repositório. Usuário já assina este tópico pelo app no celular. Testado ponta a ponta em 2026-09-06 (`curl -d "Hi" ...`, HTTP 200, mensagem recebida no celular).

Tópicos separados por severidade/propósito, não um único canal misturando tudo — decisão do usuário em 2026-09-05.

## Script reutilizável: `ntfy-alert.sh`

Em `quark` (`/usr/local/sbin/ntfy-alert.sh`, root:root 750). Outros scripts do host chamam direto:

```
/usr/local/sbin/ntfy-alert.sh "<titulo>" "<mensagem>" [prioridade] [tags]
# prioridade: min|low|default|high|urgent (padrao: default)
# tags: lista separada por virgula, emoji-code do ntfy (ex.: warning,skull)
```

`curl` com timeout de 10s e `|| true` — uma falha de rede/ntfy nunca deve derrubar o script que está chamando o alerta.

**Adotado em `proxmox-backup.sh` em 2026-09-05** (backup do script original em `/usr/local/sbin/proxmox-backup.sh.bak-20260905113716` no host): alerta de alta prioridade em cada ponto de falha (lock já em uso, `/mnt/backup` não montado, espaço insuficiente, `vzdump` com exit code != 0). Substitui a expectativa antiga de "cron/e-mail sinaliza a falha", que nunca funcionava por causa do postfix loopback-only.

**Ainda não convertido / fora de escopo por ora:** o `ultimate-updater` (BassT23/Proxmox, `/etc/ultimate-updater/update.conf`) só suporta notificação por e-mail nativamente (sem webhook/ntfy) — mas não está agendado via cron nem systemd timer neste host (rodado manualmente pelo usuário), então não há alerta silencioso em produção esperando esse gap. Se um dia for agendado, precisa de um wrapper que capture o log e chame `ntfy-alert.sh`.

## Instalação (app do usuário)

App ntfy instalado no celular (2026-09-05), assinando o tópico geral do homelab (servidor padrão `ntfy.sh`). Testado com `curl -d "..." https://ntfy.sh/<topico-geral>` (fora do host, direto do ntfy.sh): notificação chega mesmo com a tela do celular bloqueada — confirmado pelo usuário em 2026-09-05.

**Cuidado ao testar manualmente:** nunca usar nome de tópico previsível (ex.: o próprio usuário/login) — ntfy.sh não tem autenticação no plano grátis, qualquer um que adivinhe o nome publica ou lê no tópico. Sempre usar os tópicos aleatórios já criados (seção acima).
