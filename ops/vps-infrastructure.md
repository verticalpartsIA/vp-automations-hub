# VPS — Infraestrutura Docker

> Estado verificado em: 2026-05-28

---

## Servidor

```
IP:       72.61.48.156
Provider: Hostinger KVM
OS:       Ubuntu 22.04 LTS
Plan:     KVM 2 vCPU / 8GB RAM
Domínio:  srv1510643.hstgr.cloud
```

---

## Organização de diretórios

```
/docker/
├── traefik/                    ← Reverse proxy (HTTPS, Let's Encrypt)
│   ├── docker-compose.yml
│   └── acme.json               ← Certificados TLS (chmod 600)
│
├── vp-infra/                   ← Infraestrutura compartilhada
│   ├── docker-compose.yml
│   └── Serviços:
│       ├── PostgreSQL :5432    ← Banco compartilhado
│       └── Redis :6379         ← Cache compartilhado
│
├── vpautomation-hermes/        ← Hermes AI Agent
│   ├── docker-compose.yml
│   ├── .env                    ← ADMIN_USERNAME, ADMIN_PASSWORD, TRAEFIK_HOST
│   └── data/                   ← Volume /opt/data
│       ├── SOUL.md
│       ├── config.yaml
│       ├── .env                ← TELEGRAM_BOT_TOKEN, ANTHROPIC_API_KEY
│       ├── logs/gateway.log
│       ├── memories/main.md
│       └── sessions/
│
├── vpautomation-n8n/           ← n8n automações
│   ├── docker-compose.yml
│   └── .env
│
└── vpautomation-evolution/     ← Evolution API v2 (novo, porta 8081)
    ├── docker-compose.yml
    └── .env

Containers legados (fora do /docker/):
  evolution-api   ← porta 8080, pv360 conectado (MANTER ATIVO)
  n8n             ← versão antiga (verificar se ainda existe)
```

---

## Rede Docker: vp-automation

Todos os containers novos compartilham a rede `vp-automation`:

```bash
# Criar a rede (se não existir)
docker network create vp-automation

# Ver containers na rede
docker network inspect vp-automation

# Comunicação interna (exemplo):
# vpautomation-hermes pode acessar PostgreSQL em: postgres:5432
# evolution-api pode acessar Redis em: redis:6379
```

---

## Traefik — Reverse Proxy

O Traefik gerencia HTTPS para todos os serviços via Labels Docker:

```yaml
# Exemplo de labels para um serviço:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.NOME.rule=Host(`SUBDOMINIO.srv1510643.hstgr.cloud`)"
  - "traefik.http.routers.NOME.entrypoints=websecure"
  - "traefik.http.routers.NOME.tls.certresolver=letsencrypt"
  - "traefik.http.services.NOME.loadbalancer.server.port=PORTA"
```

Subdomínios ativos:
- `vpautomation-hermes.srv1510643.hstgr.cloud` → container hermes :4860

---

## Containers em execução

```bash
# Ver todos os containers
docker ps -a

# Containers esperados:
NAMES                   STATUS   PORTS
vpautomation-hermes     Up       4860→4860
evolution-api           Up       8080→8080
vpautomation-n8n        Up       5678→5678
traefik                 Up       80→80, 443→443
vp-infra_postgres_1     Up       5432→5432
vp-infra_redis_1        Up       6379→6379
evolution_api           Up       8081→8080   (reserva, vazio)
```

---

## Acesso SSH

```bash
ssh root@72.61.48.156
# Senha: [ver credenciais_master.md]
```

---

## URLs de acesso aos serviços

| Serviço | URL/Porta | Auth |
|---------|-----------|------|
| Hermes Web Terminal | https://vpautomation-hermes.srv1510643.hstgr.cloud | vphermes / VPHermes2026adm |
| Evolution API (pv360) | http://72.61.48.156:8080 | apikey: suporte123 |
| Evolution API (novo) | http://72.61.48.156:8081 | apikey: VPEvol2026key |
| n8n | http://72.61.48.156:5678 | [ver .env] |
| pv360 App | https://posvenda360.vpsistema.com | — |
