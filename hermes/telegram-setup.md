# Hermes — Setup do Gateway Telegram

> Fonte: [hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram)

---

## 1. Criar o Bot no BotFather

1. Abrir Telegram → buscar **@BotFather**
2. Enviar `/newbot`
3. Escolher nome de exibição e username (deve terminar em `bot`)
4. BotFather retorna o **API token** — guardar em lugar seguro

> ⚠️ Qualquer pessoa com esse token controla o bot. Nunca commitar no git.

---

## 2. Descobrir seu User ID

Não é o username (`@fulano`) — é um **número inteiro** como `2129471333`.

**Como descobrir:**
- Telegram → buscar **@userinfobot** ou **@get_id_bot**
- Enviar qualquer mensagem
- Ele retorna seu ID numérico

---

## 3. Configuração

### Método 1 — Wizard Interativo (recomendado)

```bash
hermes gateway setup
# Selecionar Telegram quando perguntado
# Informar: bot token + allowed user IDs
```

### Método 2 — Manual

Adicionar ao `~/.hermes/.env` (ou `/opt/data/.env` no container):

```bash
TELEGRAM_BOT_TOKEN=7123456789:AAHdqTcvCH1vGWJxfSeofSEs0NxUUnsaUlY4
TELEGRAM_ALLOWED_USERS=2129471333,7175937401
```

### Método 3 — Variável de ambiente no container

```yaml
# docker-compose.yml
services:
  hermes:
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - TELEGRAM_ALLOWED_USERS=2129471333,7175937401
```

---

## 4. Iniciar o Gateway

```bash
hermes gateway
# ou
hermes gateway run
```

Log de sucesso:
```
Connected to Telegram (polling mode)
```

---

## 5. Polling vs Webhook

| Aspecto | Polling (padrão) | Webhook |
|---------|-----------------|---------|
| Direção | Gateway → Telegram | Telegram → Gateway |
| Melhor para | Servidores locais/VPS always-on | Plataformas cloud (Fly.io, Railway) |
| Custo idle | Máquina fica rodando | Pode hibernar entre mensagens |

### Configurar Webhook (opcional)

```bash
# No .env
TELEGRAM_WEBHOOK_URL=https://meu-app.fly.dev/telegram
TELEGRAM_WEBHOOK_SECRET="$(openssl rand -hex 32)"
```

---

## 6. Privacy Mode em Grupos (CRÍTICO)

Por padrão, bots só veem mensagens que começam com `/`, replies ao bot e mensagens de serviço. Para monitorar grupos inteiros:

1. Mensagem ao **@BotFather**
2. Enviar `/mybots` → selecionar seu bot
3. **Bot Settings → Group Privacy → Turn off**
4. **Remover e readicionar o bot ao grupo** — o Telegram cacheia o estado na entrada

Alternativa: promover o bot a administrador do grupo.

---

## 7. Adicionando Usuários ao Time

### Método estático (requer restart)

```bash
TELEGRAM_ALLOWED_USERS=2129471333,7175937401,NOVO_ID
# Reiniciar gateway após mudança
```

### DM Pairing (sem restart — recomendado para equipes)

1. Usuário não autorizado envia DM para o bot
2. Bot retorna código temporário de pairing
3. Admin executa: `hermes pairing approve <código>`
4. Aprovação tem efeito imediato, sem restart

---

## 8. Home Channel (Cron Jobs)

Configurar um grupo Telegram onde os resultados de tarefas agendadas são entregues:

```bash
# No .env
TELEGRAM_HOME_CHANNEL=-1001234567890   # ID do grupo (negativo para grupos)
```

O usuário pode definir via comando: `/sethome` dentro do grupo.

---

## 9. Features do Telegram

| Feature | Como usar |
|---------|-----------|
| **Voice** | Enviar áudio — transcrição automática via Whisper |
| **Arquivos** | Enviar imagem/doc/vídeo — processado nativamente |
| **Multi-sessão** | `/topic` em DM — cria conversas forum-style paralelas |
| **Verbosidade** | `/verbose` — toggle de progresso de ferramentas |
| **Aprovação de comandos** | Botões nativos Telegram para comandos perigosos |
| **Emoji reactions** | ✅/❌ para status de aprovação |

---

## 10. Diagnóstico

```bash
# Ver log do gateway
tail -f /opt/data/logs/gateway.log

# Padrões importantes no log:
# Connected to Telegram (polling mode)   → OK
# inbound message msg='...'             → Mensagem recebida
# Flushing text batch (N chars)         → Resposta enviada
# SIGTERM                                → Container reiniciado
# APIConnectionError (3 attempts)       → Falha transiente Anthropic (auto-recupera)
# Channel directory built: 0 target(s) → Sessões limpas (rebuilda automaticamente)
```

---

## 11. Usuários autorizados na VP

| Pessoa | Telegram ID | Papel |
|--------|-------------|-------|
| Gelson Simões | `2129471333` | Consultor Estratégico |
| Diego Maeno | `7175937401` | CEO |

Configurados em `SOUL.md` seção "Quem pode me acionar" e em `TELEGRAM_ALLOWED_USERS`.
