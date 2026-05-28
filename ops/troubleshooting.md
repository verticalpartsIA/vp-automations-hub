# Troubleshooting — Problemas Comuns e Soluções

---

## Hermes não responde no Telegram

### Diagnóstico

```bash
# 1. Verificar se container está rodando
docker ps | grep hermes

# 2. Ver logs do gateway
tail -50 /docker/vpautomation-hermes/data/logs/gateway.log

# 3. Verificar conexão com Telegram
grep "Connected to Telegram" /docker/vpautomation-hermes/data/logs/gateway.log | tail -1
```

### Causas e soluções

| Causa | Log | Solução |
|-------|-----|---------|
| Primeiro start após deploy | Nenhum log ainda | Aguardar ~14 minutos |
| Container reiniciado | `SIGTERM` | Aguardar ~14 segundos |
| Memória cheia | `Memory at 2177/2200` | `echo "" > .../memories/main.md` |
| APIConnectionError | `APIConnectionError (3 attempts)` | Auto-recupera; verificar ANTHROPIC_API_KEY |
| Container parado | — | `docker start vpautomation-hermes` |

```bash
# Limpar memória
echo "" > /docker/vpautomation-hermes/data/memories/main.md

# Resetar sessões
rm -f /docker/vpautomation-hermes/data/sessions/*

# Reiniciar (reconecta em ~14s)
docker restart vpautomation-hermes
```

---

## WhatsApp não recebe/envia mensagens (pv360)

### Diagnóstico

```bash
# 1. Status da instância pv360
curl http://72.61.48.156:8080/instance/connectionState/pv360 \
  -H "apikey: suporte123"
# Esperado: "state": "open"

# 2. Configuração do webhook
curl http://72.61.48.156:8080/webhook/find/pv360 \
  -H "apikey: suporte123"

# 3. Logs do Evolution API
docker logs evolution-api --tail 50

# 4. Status do servidor pv360
curl https://posvenda360.vpsistema.com/api/whatsapp/status
```

### Casos comuns

**Estado `close` (desconectado):**
```bash
# Reconectar — abrir no navegador e escanear QR
http://72.61.48.156:8080/instance/connect/pv360
```

**Webhook não chegando:**
```bash
# Reconfigurar webhook
curl -X POST http://72.61.48.156:8080/webhook/set/pv360 \
  -H "Content-Type: application/json" \
  -H "apikey: suporte123" \
  -d '{
    "url": "https://posvenda360.vpsistema.com/api/whatsapp/webhook",
    "enabled": true,
    "webhookByEvents": false,
    "events": ["MESSAGES_UPSERT","MESSAGES_UPDATE","SEND_MESSAGE","CONNECTION_UPDATE"]
  }'
```

**Mensagens não aparecem na UI:**
```bash
# Verificar Supabase Realtime
# No browser: inspecionar WebSocket connections no DevTools → Network → WS
# Se não conecta: verificar SUPABASE_URL e SUPABASE_PUBLISHABLE_KEY no .env
```

---

## Auto-reply Claude não funciona

### Diagnóstico

```bash
# 1. Verificar se está habilitado
curl https://posvenda360.vpsistema.com/api/whatsapp/status
# Verificar: "auto_reply_ativo": true

# 2. Testar Anthropic API
curl https://posvenda360.vpsistema.com/api/whatsapp/test-claude
```

### Causas

| Causa | Diagnóstico | Solução |
|-------|-------------|---------|
| `HERMES_AUTO_REPLY=false` | `auto_reply_ativo: false` | Mudar para `true` no .env do Hostinger |
| `ANTHROPIC_API_KEY` não configurada | `claude_key_set: false` | Configurar no painel Hostinger |
| Contato é `@lid` | Mensagem salva com `lid_local_only: true` | Limitação conhecida — ver [at-lid-limitation.md](../evolution-api/at-lid-limitation.md) |
| Contato é grupo | JID contém `@g.us` | Esperado — grupos são ignorados |

---

## n8n não recebe webhooks do Evolution API

### Diagnóstico

```bash
# 1. n8n está rodando?
docker ps | grep n8n

# 2. Workflow está ativo? (verificar na UI do n8n)
# Status deve ser "Active" (não "Inactive")

# 3. URL do webhook está correta?
# Verificar na UI do n8n: Webhook node → "Webhook URL" (produção, não teste)

# 4. Evolution API está configurado?
curl http://72.61.48.156:8080/webhook/find/pv360 -H "apikey: suporte123"
```

### Solução

```bash
# Reconfigurar webhook no Evolution API apontando para n8n
curl -X POST http://72.61.48.156:8080/webhook/set/pv360 \
  -H "Content-Type: application/json" \
  -H "apikey: suporte123" \
  -d '{
    "url": "http://72.61.48.156:5678/webhook/SEU_PATH",
    "enabled": true,
    "events": ["MESSAGES_UPSERT"]
  }'
```

> Lembre: se Evolution API e n8n estão na mesma rede Docker (`vp-automation`),
> pode usar `http://vpautomation-n8n:5678/webhook/SEU_PATH` como URL interna.

---

## Container não inicia após docker compose up

```bash
# Ver erro detalhado
docker compose logs SERVICO_NAME

# Causas comuns:
# 1. Porta já em uso
netstat -tulpn | grep PORTA

# 2. Volume com permissão errada
ls -la /docker/vpautomation-hermes/data/

# 3. Variável de ambiente faltando
docker compose config   # verifica se .env foi carregado corretamente

# 4. Rede não existe
docker network ls | grep vp-automation
docker network create vp-automation  # criar se não existir
```

---

## Disco cheio no VPS

```bash
# Ver uso de disco
df -h

# Docker está usando muito espaço
docker system df

# Limpar recursos não usados (seguro)
docker system prune -f

# Limpar incluindo volumes não usados (CUIDADO!)
docker system prune -f --volumes

# Ver maiores diretórios
du -sh /docker/* | sort -rh | head -10

# Logs de containers grandes
du -sh /var/lib/docker/containers/*/ | sort -rh | head -5
```

---

## Traefik não gera certificado HTTPS

```bash
# Ver logs do Traefik
docker logs traefik --tail 50

# Verificar acme.json (deve ter chmod 600)
ls -la /docker/traefik/acme.json

# Corrigir permissão se necessário
chmod 600 /docker/traefik/acme.json
docker restart traefik

# DNS propagado? (subdomínio deve apontar para 72.61.48.156)
nslookup vpautomation-hermes.srv1510643.hstgr.cloud
```
