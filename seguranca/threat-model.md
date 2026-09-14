# Modelo de ameaça do homelab

O ambiente `particula.local` é assumido como **acesso só de dentro do LAN**. Não há exposição externa a mitigar para os serviços restritos à rede local (painéis administrativos, DNS interno, etc.) — o gateway (pfSense/OPNsense) é que fala com a internet via PPPoE com IP público, os serviços atrás dele não são expostos diretamente.

**Como aplicar:** ao sugerir hardening para serviços do homelab, **não recomendar proativamente** fail2ban, proxy reverso ou outras defesas focadas em exposição à internet (mitigam brute-force/scanning externo, que não é o risco presente aqui), a menos que o usuário informe que algum serviço específico passou a ser exposto externamente (port-forward, VPN pass-through, Cloudflare Tunnel). Continuam válidas medidas que protegem mesmo dentro do LAN — TLS para não vazar credenciais em texto claro na rede local, TSIG, ACL por IP. A distinção é especificamente sobre defesas de superfície de internet.

Todos os 15 hosts do homelab passaram por varredura de segurança completa em 2026-08-23 (SSH root desabilitado, hardening granular de 7 parâmetros, updates de SO).
