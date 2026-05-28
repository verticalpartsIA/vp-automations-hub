# Evolution API v2 — Gerenciamento de Instâncias

> Fonte: [doc.evolution-api.com — Instance Controller](https://doc.evolution-api.com/v2/api-reference/instance-controller/)

**Base URL:** `http://SEU_HOST:8080`  
**Auth:** `apikey: SUA_API_KEY`

---

## Criar Instância

```
POST /instance/create
```

```json
{
  "instanceName": "pv360",
  "token": "TOKEN_OPCIONAL",          // Se omitido, gerado automaticamente
  "number": "5511997663780",           // Número que será conectado
  "qrcode": true,                      // Gerar QR Code na criação
  "integration": "WHATSAPP-BAILEYS"   // WHATSAPP-BAILEYS | WHATSAPP-BUSINESS
}
```

**Resposta:**
```json
{
  "instance": { "instanceName": "pv360", "status": "created" },
  "hash": { "apikey": "52EEA06A-F7CB-4350-B16F-729D757842EF" },
  "qrcode": { "base64": "data:image/png;base64,..." }
}
```

---

## Conectar / Gerar QR Code

```
GET /instance/connect/{instance}
```

```bash
# Via navegador (exibe QR Code)
http://72.61.48.156:8080/instance/connect/pv360

# Via API
curl http://72.61.48.156:8080/instance/connect/pv360 \
  -H "apikey: suporte123"
```

---

## Status da Conexão

```
GET /instance/connectionState/{instance}
```

```bash
curl http://72.61.48.156:8080/instance/connectionState/pv360 \
  -H "apikey: suporte123"
```

**Resposta:**
```json
{
  "instance": {
    "instanceName": "pv360",
    "state": "open"             // open | close | connecting
  }
}
```

Estados:
- `open` — conectado ✅
- `connecting` — aguardando QR scan ⏳
- `close` — desconectado ❌

---

## Listar Instâncias

```
GET /instance/fetchInstances
```

```bash
curl http://72.61.48.156:8080/instance/fetchInstances \
  -H "apikey: suporte123"
```

---

## Desconectar (Logout)

```
DELETE /instance/logout/{instance}
```

> ⚠️ Após logout, o número precisa escanear QR Code novamente.

```bash
curl -X DELETE http://72.61.48.156:8080/instance/logout/pv360 \
  -H "apikey: suporte123"
```

---

## Deletar Instância

```
DELETE /instance/delete/{instance}
```

> ⚠️ **DESTRUTIVO** — remove instância permanentemente.

```bash
curl -X DELETE http://72.61.48.156:8080/instance/delete/pv360 \
  -H "apikey: suporte123"
```

---

## Reiniciar Instância

```
PUT /instance/restart/{instance}
```

```bash
curl -X PUT http://72.61.48.156:8080/instance/restart/pv360 \
  -H "apikey: suporte123"
```

---

## Definir Presença

```
POST /instance/setPresence/{instance}
```

```json
{ "presence": "available" }   // available | unavailable
```

---

## Configurações da Instância pv360 (VP)

```
Container:    evolution-api (porta 8080)
Instância:    pv360
Token:        52EEA06A-F7CB-4350-B16F-729D757842EF
APIKEY:       suporte123
ownerJid:     5511997663780@s.whatsapp.net
Status:       open (conectado)
Proxy:        gw.dataimpulse.com:10000 (routing Brasil)
              user: 464a00b2776c21d41dd6__cr.br
Webhook URL:  https://posvenda360.vpsistema.com/api/whatsapp/webhook
Eventos:      MESSAGES_UPSERT, MESSAGES_UPDATE, SEND_MESSAGE, CONNECTION_UPDATE
```

---

## Migração futura: pv360 porta 8080 → 8081

> **NÃO fazer sem planejamento** — requer reconexão WhatsApp com QR Code.

```bash
# Passos para migrar

# 1. Criar instância no novo container (8081)
curl -X POST http://72.61.48.156:8081/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: VPEvol2026key" \
  -d '{"instanceName":"pv360","qrcode":true}'

# 2. Escanear QR Code com o número (11) 99766-3780
# Abrir: http://72.61.48.156:8081/instance/connect/pv360

# 3. Configurar webhook
curl -X POST http://72.61.48.156:8081/webhook/set/pv360 \
  -H "Content-Type: application/json" \
  -H "apikey: VPEvol2026key" \
  -d '{
    "url": "https://posvenda360.vpsistema.com/api/whatsapp/webhook",
    "enabled": true,
    "webhookByEvents": false,
    "events": ["MESSAGES_UPSERT","MESSAGES_UPDATE","SEND_MESSAGE","CONNECTION_UPDATE"]
  }'

# 4. Atualizar server.mjs: trocar porta 8080 → 8081 e API key
# 5. Deploy do pv360
# 6. Verificar webhooks chegando
# 7. Desligar instância do 8080 (opcional)
```
