# Integração — Evolution API → n8n

> Guia para conectar webhooks do Evolution API a workflows n8n

---

## Visão geral

```
WhatsApp → Evolution API → webhook POST → n8n Webhook Node
                                              ↓
                                    Workflow n8n processa
                                    (notifica, cria ticket, chama IA, etc.)
```

---

## Passo 1 — Configurar Webhook node no n8n

1. Criar workflow → adicionar **Webhook node**
2. Configurar:
   ```
   HTTP Method:     POST
   Path:            evolution-pv360
   Authentication:  Header Auth
     Header Name:   apikey
     Header Value:  suporte123
   Response Mode:   Immediately (respond with 200 immediately)
   ```
3. URL resultante: `https://n8n.host/webhook/evolution-pv360`
4. **Ativar o workflow** (toggle no canto superior direito)

---

## Passo 2 — Configurar webhook no Evolution API

```bash
curl -X POST http://72.61.48.156:8080/webhook/set/pv360 \
  -H "Content-Type: application/json" \
  -H "apikey: suporte123" \
  -d '{
    "url": "http://72.61.48.156:5678/webhook/evolution-pv360",
    "enabled": true,
    "webhookByEvents": false,
    "webhookBase64": false,
    "events": [
      "MESSAGES_UPSERT",
      "MESSAGES_UPDATE",
      "CONNECTION_UPDATE"
    ]
  }'
```

> **Nota:** Use IP interno `172.x.x.x` se ambos containers estiverem na mesma rede Docker, ou o IP/domínio externo. Ambos estão na rede `vp-automation`.

---

## Passo 3 — Workflow exemplo completo

```json
// Estrutura lógica do workflow n8n:

[Webhook: evolution-pv360]
        ↓
[IF node: filtrar mensagens relevantes]
  Condition 1: {{ $json.event }} = "messages.upsert"
  Condition 2: {{ $json.data.key.fromMe }} = false
  Condition 3: {{ $json.data.key.remoteJid }} NOT contains "@g.us"
        ↓ TRUE
[Set node: extrair dados]
  event_type: {{ $json.event }}
  jid:        {{ $json.data.key.remoteJid }}
  phone:      {{ $json.data.key.remoteJid.replace('@s.whatsapp.net','').replace('@lid','') }}
  name:       {{ $json.data.pushName || 'Desconhecido' }}
  message:    {{ $json.data.message.conversation || $json.data.message.extendedTextMessage.text || '[mídia]' }}
  is_lid:     {{ $json.data.key.remoteJid.endsWith('@lid') }}
  timestamp:  {{ $json.data.messageTimestamp }}
        ↓
[IF node: verificar se é @lid]
        ↓ FALSE (pode responder)        ↓ TRUE (não pode responder)
[HTTP Request: enviar resposta]    [Telegram: notificar operador sobre @lid]
  POST .../sendText/pv360
```

---

## Expressões úteis no n8n para payload Evolution

```javascript
// Tipo de evento
{{ $json.event }}

// JID do contato
{{ $json.data.key.remoteJid }}

// É mensagem enviada pelo bot?
{{ $json.data.key.fromMe }}

// Nome do contato
{{ $json.data.pushName }}

// Texto da mensagem (cobre tipos comuns)
{{ $json.data.message.conversation 
   ?? $json.data.message.extendedTextMessage?.text 
   ?? '[mídia]' }}

// Tipo de mídia
{{ $json.data.messageType }}

// Timestamp Unix → Data legível
{{ new Date($json.data.messageTimestamp * 1000).toLocaleString('pt-BR') }}

// Extrair número do JID
{{ $json.data.key.remoteJid.replace('@s.whatsapp.net', '').replace('@lid', '') }}

// Verificar se é grupo
{{ $json.data.key.remoteJid.includes('@g.us') }}

// Verificar se é @lid
{{ $json.data.key.remoteJid.endsWith('@lid') }}
```

---

## Workflow: Evolution API → Supabase direto no n8n

```
[Webhook: evolution-pv360]
        ↓
[IF: é mensagem nova de cliente?]
        ↓ TRUE
[HTTP Request: POST Supabase]
  URL:  https://jkbklzlbhhfnamaeislb.supabase.co/rest/v1/whatsapp_messages
  Method: POST
  Headers:
    apikey:        eyJ... (SUPABASE_SERVICE_ROLE_KEY)
    Authorization: Bearer eyJ... (SUPABASE_SERVICE_ROLE_KEY)
    Content-Type:  application/json
    Prefer:        return=minimal
  Body:
    {
      "instance": "pv360",
      "remote_jid": "{{ $json.data.key.remoteJid }}",
      "push_name": "{{ $json.data.pushName }}",
      "from_me": false,
      "message_id": "{{ $json.data.key.id }}",
      "body": "{{ $json.data.message.conversation ?? '' }}",
      "raw": {{ JSON.stringify($json.data) }}
    }
```

---

## Tratamento de erros no n8n

- Adicionar **Error Trigger node** ao workflow para capturar falhas
- Configurar **Retry on fail** nos nodes HTTP Request críticos (3 tentativas, 2s intervalo)
- Usar **Catch node** após Evolution API para tratar 400 @lid graciosamente

---

## Considerações de performance

- Evolution API pode enviar muitos eventos em burst (histórico inicial `MESSAGES_SET`)
- Usar **IF node** logo após o Webhook para descartar eventos desnecessários rapidamente
- Considerar `MESSAGES_UPSERT` apenas para reduzir volume (excluir `MESSAGES_SET`)
