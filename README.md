# homelab

Projeto sobre o meu homelab, construído pelo Claude Code.

Domínio: `particula.local` (203.0.113.0/24). Host físico único de virtualização: `quark` (Proxmox).

## Índice

- [`infra/`](infra/) — host físico Proxmox (`quark`).
- [`rede/`](rede/) — DNS, filtros (Pi-hole/AdGuard), gateway/firewall.
- [`seguranca/`](seguranca/) — PKI/CA interna (OpenBao, CA de produção em `203.0.113.15`), modelo de ameaça.
- [`servicos/`](servicos/) — serviços individuais (cofre de senhas, phpIPAM, syslog, MariaDB Galera Cluster).
- [`notas/`](notas/) — padrões operacionais e gotchas técnicos recorrentes.
- [`planejamento/`](planejamento/) — roteiros de mudanças futuras ainda não executadas (ex. migração DNS Technitium → BIND9).

Em caso de desastre (perda do host `quark`), ver [RECUPERACAO-DESASTRE.md](dr/RECUPERACAO-DESASTRE.md).

Ver [CLAUDE.md](CLAUDE.md) para a regra de organização deste repositório.
