# Evolution API — Instância pv360 VerticalParts

> Estado verificado em: 2026-05-28

---

## Configuração atual

```
Container:    evolution-api (LEGADO — não mover sem planejamento)
Porta:        8080
Imagem:       atendai/evolution-api:v2.2.3
Instância:    pv360
Token:        52EEA06A-F7CB-4350-B16F-729D757842EF
APIKEY:       suporte123
ownerJid:     5511997663780@s.whatsapp.net
Número:       (11) 99766-3780 (número pessoal do Gelson — trocar em produção)
Status:       open (conectado)
Proxy:        gw.dataimpulse.com:10000 (http, routing Brasil)
```

---

## Webhook configurado

```
URL:      https://posvenda360.vpsistema.com/api/whatsapp/webhook
Eventos:  MESSAGES_UPSERT, MESSAGES_UPDATE, SEND_MESSAGE, CONNECTION_UPDATE
By events: false (URL única para todos os eventos)
```

---

## Endpoints de diagnóstico

```bash
BASE="http://72.61.48.156:8080"
KEY="suporte123"

# Status da conexão
curl "$BASE/instance/connectionState/pv360" -H "apikey: $KEY"

# Configuração do webhook
curl "$BASE/webhook/find/pv360" -H "apikey: $KEY"

# Listar todas as instâncias
curl "$BASE/instance/fetchInstances" -H "apikey: $KEY"

# Verificar se número existe no WhatsApp
curl -X POST "$BASE/chat/whatsappNumbers/pv360" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"numbers":["5511999999999"]}'

# Histórico de mensagens de um JID
curl -X POST "$BASE/chat/findMessages/pv360" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"where":{"key":{"remoteJid":"5511999999999@s.whatsapp.net"}},"limit":10}'

# Listar contatos
curl -X POST "$BASE/chat/findContacts/pv360" \
  -H "Content-Type: application/json" \
  -H "apikey: $KEY" \
  -d '{"where":{}}'
```

---

## Status do servidor pv360 (Hostinger)

```bash
# Verificar variáveis de ambiente e status
GET https://posvenda360.vpsistema.com/api/whatsapp/status

# Testar conexão com Anthropic API
GET https://posvenda360.vpsistema.com/api/whatsapp/test-claude
```

**Estado verificado (28/05/2026):**
```json
{
  "claude_key_set": true,
  "hermes_auto_reply": "false",
  "auto_reply_ativo": false,
  "evolution_apikey": "supo...",
  "env_file_loaded": true
}
```

---

## Variáveis de ambiente do servidor (hostinger/.env)

```bash
# Supabase
SUPABASE_URL=https://jkbklzlbhhfnamaeislb.supabase.co
SUPABASE_SERVICE_ROLE_KEY=...

# Anthropic / Claude
ANTHROPIC_API_KEY=sk-ant-...
HERMES_AUTO_REPLY=false       # Mudar para "true" para ativar auto-reply

# Evolution API
EVOLUTION_APIKEY=suporte123
```

**Prioridade de carregamento do .env no servidor:**
1. `hostinger/.env` (mesmo dir do server.mjs)
2. `cwd()/.env` (pasta nodejs/)
3. `cwd()/../.env` (pasta pai)
4. `$HOME/.env` (home Hostinger)
5. `$HOME/posvenda360.env`

---

## Como funciona o auto-reply Claude

```
1. Mensagem chega via MESSAGES_UPSERT
2. server.mjs salva em whatsapp_messages (Supabase)
3. automateIncoming() roda fire-and-forget
4. Busca ticket aberto para o remoteJid
5. Se não tem → cria ticket (status: aberto, channel: whatsapp)
6. Vincula mensagem ao ticket
7. Se HERMES_AUTO_REPLY=true:
   a. callClaudeWithHistory(remoteJid) — busca 20 últimas mensagens
   b. Chama Anthropic API (modelo: HERMES_MODEL ou claude-haiku-4-5)
   c. Envia resposta via Evolution API POST /message/sendText/pv360
   d. Salva resposta em whatsapp_messages (from_me: true)
   e. Salva em ticket_messages (author: "Claude (VerticalParts Bot)")
8. Se NOTIFY_WEBHOOK_URL definida → notifica time
```

**Atenção:** Para `@lid`, o passo 7c falha silenciosamente (Evolution API retorna 400). O erro é logado mas não propagado.

---

## Estrutura da tabela whatsapp_messages (Supabase)

```sql
id           uuid PRIMARY KEY
instance     text              -- "pv360"
remote_jid   text              -- JID do contato
phone        text GENERATED    -- JID sem sufixo (gerado automaticamente)
push_name    text              -- Nome no WhatsApp
from_me      boolean           -- true = enviado pelo pv360
message_id   text              -- ID da mensagem no WhatsApp
body         text              -- Texto (ou "[imagem]", "[áudio]", etc.)
media_type   text              -- null | image | video | audio | document | sticker
media_url    text              -- URL da mídia (quando disponível)
ticket_id    uuid FK           -- Ticket associado
raw          jsonb             -- Payload completo do webhook
              -- raw.lid_local_only = true → não enviado ao cliente
created_at   timestamptz
```

---

## Deploy do pv360

- **Repo:** `github.com/verticalpartsIA/resolve-360` (branch: main)
- **Deploy:** Automático — qualquer push para `main` dispara build + deploy no Hostinger
- **Build:** `vite build` (TanStack Start) → `dist/` + `hostinger/server.mjs`
- **Start:** `node hostinger/server.mjs`
- **Env vars:** Persistem entre deploys (configuradas no painel Hostinger)
