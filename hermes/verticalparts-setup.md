# Hermes — Setup VerticalParts (Produção)

> Estado verificado em: 2026-05-28

---

## Container em produção

```
Nome:       vpautomation-hermes
Imagem:     ghcr.io/hostinger/hvps-hermes-agent:latest
Porta ext:  4860 (via Traefik → HTTPS)
URL:        https://vpautomation-hermes.srv1510643.hstgr.cloud
Rede:       vp-automation

Processos internos:
  PID 1:  ttyd (web terminal — porta 4860) → interface de controle
  PID 10: /opt/hermes/.venv/bin/python3 /opt/hermes/.venv/bin/hermes gateway run
```

---

## Arquivos críticos no VPS (host)

```
/docker/vpautomation-hermes/
├── docker-compose.yml
├── .env                              ← ADMIN_USERNAME, ADMIN_PASSWORD, TRAEFIK_HOST
└── data/                             ← montado como /opt/data dentro do container
    ├── SOUL.md                       ← identidade do Hermes
    ├── config.yaml                   ← language: pt-BR (e outras configs)
    ├── .env                          ← TELEGRAM_BOT_TOKEN, ANTHROPIC_API_KEY
    ├── logs/
    │   └── gateway.log               ← log principal (ver aqui primeiro)
    ├── memories/
    │   └── main.md                   ← memórias acumuladas (max ~2200 chars)
    └── sessions/                     ← sessões Telegram
```

---

## docker-compose.yml (referência)

```yaml
version: "3.9"
services:
  vpautomation-hermes:
    image: ghcr.io/hostinger/hvps-hermes-agent:latest
    container_name: vpautomation-hermes
    restart: unless-stopped
    ports:
      - "4860:4860"
    env_file:
      - .env
    volumes:
      - ./data:/opt/data
    networks:
      - vp-automation
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hermes.rule=Host(`vpautomation-hermes.srv1510643.hstgr.cloud`)"
      - "traefik.http.routers.hermes.entrypoints=websecure"
      - "traefik.http.routers.hermes.tls.certresolver=letsencrypt"
      - "traefik.http.services.hermes.loadbalancer.server.port=4860"

networks:
  vp-automation:
    external: true
```

---

## Usuários autorizados

| Pessoa | Telegram ID | Papel |
|--------|-------------|-------|
| Gelson Simões | `2129471333` | Consultor Estratégico |
| Diego Maeno | `7175937401` | CEO |

---

## Diagnóstico pelos logs

```bash
# Ver logs em tempo real
tail -f /docker/vpautomation-hermes/data/logs/gateway.log

# Ou via docker
docker exec vpautomation-hermes tail -f /opt/data/logs/gateway.log

# Últimas 100 linhas
tail -100 /docker/vpautomation-hermes/data/logs/gateway.log
```

### Padrões importantes no gateway.log

| Log | Significado |
|-----|-------------|
| `Connected to Telegram (polling mode)` | ✅ Gateway iniciado com sucesso |
| `Flushing text batch (N chars) → to CHATID` | ✅ Resposta sendo enviada |
| `inbound message msg='...'` | ✅ Mensagem recebida do usuário |
| `Channel directory built: 0 target(s)` | ℹ️ Sessões limpas — rebuilda sozinho |
| `Memory at 2177/2200 chars` | ⚠️ Memória quase cheia — limpar |
| `APIConnectionError (3 attempts)` | ⚠️ Falha transiente Anthropic — auto-recupera |
| `SIGTERM` | ℹ️ Container reiniciado — reconecta em ~14s |

---

## Causas de demora para responder

1. **Primeiro start após deploy:** ~14 minutos para inicializar completamente
2. **Após restart do container:** ~14 segundos para reconectar ao Telegram
3. **Anthropic APIConnectionError:** Falha transiente, recupera automaticamente
4. **Memória cheia:** `memories/main.md` > 2200 chars → limpar entradas antigas
5. **Sessions deletadas:** `rm -f sessions/*` → canal rebuilda automaticamente

---

## Comandos operacionais

```bash
# Ver status do container
docker stats vpautomation-hermes --no-stream

# Entrar no container
docker exec -it vpautomation-hermes bash

# Reiniciar (cuidado — mata gateway por ~14s, reconecta automaticamente)
docker restart vpautomation-hermes

# Ver uso de memória do agente
wc -c /docker/vpautomation-hermes/data/memories/main.md

# Limpar memórias (se > 2200 chars)
echo "" > /docker/vpautomation-hermes/data/memories/main.md

# Resetar sessões Telegram (força rebuild do canal)
rm -f /docker/vpautomation-hermes/data/sessions/*

# Verificar config de idioma
cat /docker/vpautomation-hermes/data/config.yaml

# Ver SOUL.md atual
cat /docker/vpautomation-hermes/data/SOUL.md
```

---

## Atualizar imagem

```bash
cd /docker/vpautomation-hermes
docker compose pull
docker compose up -d
```

> ⚠️ O primeiro start após atualização de imagem pode levar ~14 minutos.
