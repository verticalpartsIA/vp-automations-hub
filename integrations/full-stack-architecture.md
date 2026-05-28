# Arquitetura Completa — VP Automation Stack

> Mapeamento de toda a infraestrutura de automação VerticalParts

---

## Diagrama de alto nível

```
┌─────────────────────────────────────────────────────────────────┐
│                    VPS: 72.61.48.156                            │
│                                                                  │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  Traefik │    │    Hermes Agent   │    │    n8n           │  │
│  │ (HTTPS)  │───▶│  vpautomation-   │    │  vpautomation-   │  │
│  │          │    │  hermes          │    │  n8n :5678       │  │
│  └──────────┘    │  Telegram Bot    │    │                  │  │
│                  └──────────────────┘    └──────────────────┘  │
│                           │                        │            │
│                    ┌──────┴──────┐                 │            │
│                    │ Anthropic   │                 │            │
│                    │ Claude API  │                 │            │
│                    └─────────────┘                 │            │
│                                                    │            │
│  ┌──────────────────────────────────────────────┐ │            │
│  │           Evolution API :8080                 │ │            │
│  │           (container: evolution-api)          │◀┘            │
│  │           instância: pv360                    │              │
│  │           número: (11) 99766-3780             │              │
│  └──────────────────────────────────────────────┘              │
│           │                                                      │
│           │ webhook POST                                         │
│           ▼                                                      │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │  PostgreSQL      │    │  Redis                            │   │
│  │  (vp-infra)      │    │  (vp-infra)                      │   │
│  └─────────────────┘    └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │
         │ webhook POST /api/whatsapp/webhook
         ▼
┌─────────────────────────────────────────────────────────────────┐
│               Hostinger Node.js Hosting                          │
│         posvenda360.vpsistema.com                                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  node hostinger/server.mjs                                │  │
│  │                                                            │  │
│  │  POST /api/whatsapp/webhook  ← Evolution API              │  │
│  │  POST /api/whatsapp/send     ← UI (painel)                │  │
│  │  GET  /api/whatsapp/status   ← diagnóstico                │  │
│  │                                                            │  │
│  │  automateIncoming():                                       │  │
│  │    1. Salvar mensagem → Supabase                          │  │
│  │    2. Criar ticket (se não existe)                        │  │
│  │    3. Se HERMES_AUTO_REPLY=true:                          │  │
│  │       → callClaudeWithHistory()                           │  │
│  │       → Anthropic API (claude-haiku-4-5)                 │  │
│  │       → sendText via Evolution API                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                          │ Supabase JS client                    │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  TanStack Start (React SSR)                               │  │
│  │                                                            │  │
│  │  /whatsapp-threads  ← lista de conversas                  │  │
│  │  /thread/$jid       ← conversa individual                 │  │
│  │  /ocorrencia/$id    ← ticket (RO)                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │
         │ Supabase Realtime (WebSocket)
         ▼
┌─────────────────────────────────────────────────────────────────┐
│               Supabase                                           │
│         jkbklzlbhhfnamaeislb.supabase.co                        │
│                                                                  │
│  Tables:                                                         │
│    whatsapp_messages   ← todas as msgs WhatsApp                  │
│    tickets             ← ROs (registros de ocorrência)           │
│    ticket_messages     ← mensagens linkadas a tickets            │
│                                                                  │
│  Realtime:                                                       │
│    INSERT em whatsapp_messages → UI atualiza em tempo real      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Fluxo completo de uma mensagem WhatsApp

```
1. RECEBIMENTO
   Cliente envia mensagem WhatsApp
         ↓
   Evolution API (pv360) recebe via Baileys
         ↓
   POST https://posvenda360.vpsistema.com/api/whatsapp/webhook
   { event: "messages.upsert", data: { remoteJid, pushName, message } }

2. PROCESSAMENTO (server.mjs)
   Verifica: fromMe=false, não é grupo
         ↓
   Salva em Supabase → whatsapp_messages
         ↓
   Supabase Realtime → UI atualiza (novo bubble aparece)
         ↓
   automateIncoming() roda async:
     - Busca/cria ticket no Supabase
     - Se HERMES_AUTO_REPLY=true:
         → Busca histórico (últimas 20 msgs)
         → Chama Anthropic API
         → Envia resposta via Evolution API
         → Salva resposta no Supabase

3. VISUALIZAÇÃO
   Operador abre painel → /whatsapp-threads
   Vê thread atualizada em tempo real
   Para @s.whatsapp.net: pode digitar e enviar resposta
   Para @lid: vê painel informativo (limitação WhatsApp)

4. RESPOSTA MANUAL (para @s.whatsapp.net)
   Operador digita mensagem → UI
         ↓
   POST /api/whatsapp/send → server.mjs
         ↓
   POST http://72.61.48.156:8080/message/sendText/pv360
         ↓
   Mensagem entregue ao cliente
         ↓
   Evolution API dispara SEND_MESSAGE
         ↓
   webhook salva no Supabase (from_me: true)
         ↓
   UI mostra mensagem como enviada (checkmark)
```

---

## Hermes como agente de controle VPS

O Hermes (container `vpautomation-hermes`) serve como **interface de controle** do VPS via Telegram:

```
Gelson ou Diego → Telegram → Bot do Hermes
                                    ↓
                            vpautomation-hermes
                            (ttyd web terminal)
                                    ↓
                            Terminal do VPS
                            (acesso completo via PID 10)
```

### Casos de uso do Hermes no VPS

- Verificar logs dos containers
- Reiniciar serviços
- Atualizar configurações (SOUL.md, config.yaml)
- Executar comandos Docker
- Verificar status das instâncias Evolution API
- Deploy de atualizações

---

## Repositórios GitHub

| Repo | Propósito | Deploy |
|------|-----------|--------|
| [verticalpartsIA/resolve-360](https://github.com/verticalpartsIA/resolve-360) | VP Pós-Venda 360 (pv360) | Auto (push → main) → Hostinger |
| [verticalpartsIA/vp-automations-hub](https://github.com/verticalpartsIA/vp-automations-hub) | Este repo — documentação | — |

---

## Stack tecnológica completa

| Camada | Tecnologia |
|--------|-----------|
| Frontend | React + TanStack Start (SSR) |
| Backend | Node.js (server.mjs nativo) |
| Database | Supabase (PostgreSQL + Realtime) |
| WhatsApp | Evolution API v2.2.3 (Baileys) |
| AI | Anthropic Claude (Haiku para auto-reply) |
| AI Agent | Hermes (NousResearch) via Telegram |
| Automação | n8n (self-hosted) |
| Infra | Docker + Traefik + VPS Hostinger |
| CI/CD | GitHub + Hostinger auto-deploy |
