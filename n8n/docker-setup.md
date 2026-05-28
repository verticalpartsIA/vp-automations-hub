# n8n — Docker Setup (Self-Hosted)

> Fonte: [docs.n8n.io/hosting/installation/docker](https://docs.n8n.io/hosting/installation/docker/)

---

## docker-compose.yml (com PostgreSQL)

```yaml
version: "3.9"

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: vpautomation-n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # n8n config
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}               # ex: n8n.meu-dominio.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=America/Sao_Paulo
      - N8N_LOG_LEVEL=info
      # Encryption
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - vp-automation
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_n8n:/var/lib/postgresql/data
    networks:
      - vp-automation

volumes:
  n8n_data:
  postgres_n8n:

networks:
  vp-automation:
    external: true
```

---

## .env

```bash
# Database
POSTGRES_PASSWORD=senha_postgres_aqui

# n8n Auth
N8N_USER=admin
N8N_PASSWORD=senha_n8n_aqui

# n8n Config
N8N_HOST=n8n.meu-dominio.com       # ou IP para acesso direto
N8N_ENCRYPTION_KEY=chave_32_chars_aleatoria

# Timezone
GENERIC_TIMEZONE=America/Sao_Paulo
```

---

## docker-compose.yml simplificado (sem PostgreSQL, com SQLite)

```yaml
version: "3.9"
services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=America/Sao_Paulo
    volumes:
      - ~/.n8n:/home/node/.n8n
```

> ⚠️ SQLite não recomendado para produção — usar PostgreSQL.

---

## Variáveis de ambiente importantes

| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `N8N_HOST` | Hostname público | `localhost` |
| `N8N_PORT` | Porta | `5678` |
| `N8N_PROTOCOL` | http ou https | `http` |
| `WEBHOOK_URL` | URL base para webhooks | `http://N8N_HOST:N8N_PORT/` |
| `N8N_ENCRYPTION_KEY` | Chave de criptografia das credenciais | gerada automaticamente |
| `N8N_BASIC_AUTH_ACTIVE` | Habilitar autenticação básica | `false` |
| `N8N_BASIC_AUTH_USER` | Usuário autenticação básica | — |
| `N8N_BASIC_AUTH_PASSWORD` | Senha autenticação básica | — |
| `DB_TYPE` | Tipo de banco | `sqlite` |
| `DB_POSTGRESDB_HOST` | Host PostgreSQL | — |
| `GENERIC_TIMEZONE` | Timezone (IANA) | `UTC` |
| `N8N_LOG_LEVEL` | debug, info, warn, error | `info` |
| `EXECUTIONS_DATA_PRUNE` | Auto-limpar execuções antigas | `false` |
| `EXECUTIONS_DATA_MAX_AGE` | Horas para manter execuções | `336` (14 dias) |

---

## Comandos operacionais

```bash
# Subir n8n
docker compose up -d

# Ver logs
docker logs vpautomation-n8n -f

# Acessar n8n
http://localhost:5678

# Parar
docker compose down

# Atualizar (cuidado — fazer backup antes)
docker compose pull && docker compose up -d

# Backup do volume de dados
docker run --rm \
  -v n8n_data:/source \
  -v $(pwd):/backup \
  alpine tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz -C /source .
```

---

## Acesso com Traefik (HTTPS)

```yaml
# Adicionar labels ao serviço n8n:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`n8n.srv1510643.hstgr.cloud`)"
  - "traefik.http.routers.n8n.entrypoints=websecure"
  - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
  - "traefik.http.services.n8n.loadbalancer.server.port=5678"
```
