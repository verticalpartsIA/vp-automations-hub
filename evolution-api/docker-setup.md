# Evolution API v2 — Docker Setup

> Fonte: [doc.evolution-api.com/v2/en/install/docker](https://doc.evolution-api.com/v2/en/install/docker)

---

## docker-compose.yml completo

```yaml
version: "3.9"

services:
  evolution-api:
    container_name: evolution_api
    image: atendai/evolution-api:v2.2.3    # Fixar versão em produção!
    restart: unless-stopped
    ports:
      - "8080:8080"
    env_file:
      - .env
    volumes:
      - evolution_instances:/evolution/instances
    networks:
      - vp-automation
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    container_name: evolution_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: evolution
      POSTGRES_USER: evolution
      POSTGRES_PASSWORD: evolution_password_change_me
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - vp-automation

  redis:
    image: redis:7-alpine
    container_name: evolution_redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - vp-automation

volumes:
  evolution_instances:
  postgres_data:
  redis_data:

networks:
  vp-automation:
    external: true
```

---

## .env — Variáveis de Ambiente Completas

```bash
# ─── Servidor ─────────────────────────────────────────────────────────────────
SERVER_TYPE=http
SERVER_PORT=8080
SERVER_URL=https://meu-dominio.com     # URL pública (usada nos webhooks de retorno)

# ─── CORS ─────────────────────────────────────────────────────────────────────
CORS_ORIGIN=*
CORS_METHODS=POST,GET,PUT,DELETE
CORS_CREDENTIALS=true

# ─── Logs ─────────────────────────────────────────────────────────────────────
LOG_LEVEL=ERROR,WARN,DEBUG,INFO,LOG,VERBOSE,DARK,WEBHOOKS
LOG_COLOR=true
LOG_BAILEYS=error

# ─── Instâncias ───────────────────────────────────────────────────────────────
DEL_INSTANCE=false              # false = nunca deletar automaticamente

# ─── Database (PostgreSQL) ────────────────────────────────────────────────────
DATABASE_ENABLED=true
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://evolution:senha@postgres:5432/evolution
DATABASE_CONNECTION_CLIENT_NAME=evolution_v2
DATABASE_SAVE_DATA_INSTANCE=true
DATABASE_SAVE_DATA_NEW_MESSAGE=true
DATABASE_SAVE_MESSAGE_UPDATE=true
DATABASE_SAVE_DATA_CONTACTS=true
DATABASE_SAVE_DATA_CHATS=true
DATABASE_SAVE_DATA_LABELS=true
DATABASE_SAVE_DATA_HISTORIC=true

# ─── Redis ────────────────────────────────────────────────────────────────────
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://redis:6379
CACHE_REDIS_PREFIX_KEY=evolution
CACHE_REDIS_SAVE_INSTANCES=false
CACHE_LOCAL_ENABLED=false

# ─── Webhook Global ───────────────────────────────────────────────────────────
WEBHOOK_GLOBAL_ENABLED=false
WEBHOOK_GLOBAL_URL=
WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS=false

# Eventos habilitados globalmente
WEBHOOK_EVENTS_APPLICATION_STARTUP=false
WEBHOOK_EVENTS_QRCODE_UPDATED=true
WEBHOOK_EVENTS_MESSAGES_SET=true
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
WEBHOOK_EVENTS_MESSAGES_UPDATE=true
WEBHOOK_EVENTS_MESSAGES_DELETE=true
WEBHOOK_EVENTS_SEND_MESSAGE=true
WEBHOOK_EVENTS_CONTACTS_SET=true
WEBHOOK_EVENTS_CONTACTS_UPSERT=true
WEBHOOK_EVENTS_CONTACTS_UPDATE=true
WEBHOOK_EVENTS_PRESENCE_UPDATE=true
WEBHOOK_EVENTS_CHATS_SET=true
WEBHOOK_EVENTS_CHATS_UPSERT=true
WEBHOOK_EVENTS_CHATS_UPDATE=true
WEBHOOK_EVENTS_CHATS_DELETE=true
WEBHOOK_EVENTS_GROUPS_UPSERT=true
WEBHOOK_EVENTS_GROUPS_UPDATE=true
WEBHOOK_EVENTS_GROUP_PARTICIPANTS_UPDATE=true
WEBHOOK_EVENTS_CONNECTION_UPDATE=true
WEBHOOK_EVENTS_CALL=true
WEBHOOK_EVENTS_NEW_JWT_TOKEN=false

# ─── QR Code ──────────────────────────────────────────────────────────────────
QRCODE_LIMIT=30
QRCODE_COLOR=#198754

# ─── Sessão ───────────────────────────────────────────────────────────────────
CONFIG_SESSION_PHONE_CLIENT=Evolution API
CONFIG_SESSION_PHONE_NAME=Chrome

# ─── Autenticação ─────────────────────────────────────────────────────────────
AUTHENTICATION_API_KEY=SUA_API_KEY_AQUI    # OBRIGATÓRIO — mudar antes de produção!
AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES=true

# ─── Integrações ──────────────────────────────────────────────────────────────
OPENAI_ENABLED=false
DIFY_ENABLED=false
CHATWOOT_ENABLED=false
TYPEBOT_API_VERSION=latest

# ─── Telemetria ───────────────────────────────────────────────────────────────
TELEMETRY=false
```

---

## Comandos de Deploy

```bash
# Subir serviços
docker compose up -d

# Ver logs
docker logs evolution_api -f

# Acessar API
curl http://localhost:8080 -H "apikey: SUA_API_KEY"

# Parar
docker compose down

# Atualizar imagem
docker compose pull && docker compose up -d
```

---

## Verificação pós-deploy

```bash
# Health check
curl http://localhost:8080

# Listar instâncias
curl http://localhost:8080/instance/fetchInstances \
  -H "apikey: SUA_API_KEY"

# Ver versão
curl http://localhost:8080 | jq .version
```
