# Hermes — Referência Completa do config.yaml

> Fonte: [hermes-agent.nousresearch.com/docs/user-guide/configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

---

## Regra de ouro

| Tipo | Arquivo |
|------|---------|
| API keys, tokens, passwords | `~/.hermes/.env` |
| Toda outra configuração | `~/.hermes/config.yaml` |

**Precedência (maior → menor):** CLI args > `config.yaml` > `.env` > defaults internos

---

## Modelo e Provider

```yaml
model:
  default: anthropic/claude-opus-4   # ou claude-haiku-4-5 para uso leve
  provider: anthropic                 # anthropic | openrouter | openai | google-gemini-cli
  context_length: 200000
```

Providers suportados: `anthropic`, `openrouter`, `openai`, `google-gemini-cli`, `minimax-oauth`, `xai-oauth`, endpoints customizados.

---

## Terminal Backend

```yaml
terminal:
  backend: docker          # local | docker | ssh | modal | daytona | singularity
  docker_image: nikolaik/python-nodejs:python3.11-nodejs20
  docker_mount_cwd_to_workspace: false
  docker_run_as_host_user: false
  docker_volumes:
    - "/home/user/projects:/workspace/projects"
    - "/home/user/data:/data:ro"
  docker_extra_args:
    - "--gpus=all"
  timeout: 180
  cwd: "."
```

---

## Memória

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200      # ~800 tokens — limpar main.md se ultrapassar
  user_char_limit: 1375        # ~500 tokens por perfil de usuário
```

> ⚠️ **Atenção VP:** Monitorar `Memory at XXXX/2200 chars` no gateway.log. Se chegar perto de 2200, limpar entradas antigas do `memories/main.md`.

---

## Compressão de Contexto

```yaml
compression:
  enabled: true
  threshold: 0.50        # Ativa ao usar 50% do context limit
  target_ratio: 0.20     # Mantém 20% como cauda recente
  protect_last_n: 20     # Mín. mensagens recentes não comprimidas
  protect_first_n: 3     # Pin das primeiras mensagens

auxiliary:
  compression:
    model: ""            # Vazio = usa modelo principal
    provider: "auto"
    base_url: null
```

---

## Modelos Auxiliares (Visão, Web, Compressão)

```yaml
auxiliary:
  vision:
    provider: "auto"
    model: "google/gemini-2.5-flash"
    base_url: ""
    timeout: 120
  web_extract:
    provider: "auto"
    model: "google/gemini-2.5-flash"
    timeout: 360
  compression:
    provider: "auto"
    model: ""
    timeout: 120
```

`"auto"` detecta automaticamente: OpenRouter → Nous Portal → Codex.

---

## Agente — Comportamento

```yaml
agent:
  reasoning_effort: ""        # "" (medium) | minimal | low | medium | high | xhigh
  max_turns: 90               # Budget de iterações por conversa
  api_max_retries: 3          # Tentativas antes de fallback
  tool_use_enforcement: "auto"
  disabled_toolsets:
    - memory                  # Desabilitar toolsets específicos
    - web
```

---

## Display e Idioma

```yaml
display:
  tool_progress: all          # off | new | all | verbose
  streaming: false
  show_cost: false
  timestamps: false
  language: pt-BR             # en | pt-BR | zh | ja | de | es | fr | tr | uk
  show_reasoning: false
  compact: false
  
  platforms:                  # Overrides por plataforma
    telegram:
      tool_progress: verbose
    signal:
      tool_progress: "off"
```

> **VP:** Usamos `language: pt-BR` no container `vpautomation-hermes`.

---

## Delegação (Subagentes)

```yaml
delegation:
  model: ""                       # Vazio = herda do parent
  provider: ""
  max_concurrent_children: 3      # Filhos paralelos por batch
  max_spawn_depth: 1              # Profundidade da árvore (1-3)
  orchestrator_enabled: true
```

---

## Segurança

```yaml
security:
  redact_secrets: false           # Redacta API keys no output/logs
  tirith_enabled: true            # Scan de comandos perigosos
  tirith_fail_open: true          # Permite se tirith indisponível
  website_blocklist:
    enabled: false
    domains:
      - "*.internal.company.com"
```

---

## Aprovações (Smart Approvals)

```yaml
approvals:
  mode: manual      # manual | smart | off
```

---

## Streaming (Gateway)

```yaml
streaming:
  enabled: true
  transport: edit
  edit_interval: 0.3
  buffer_threshold: 40
  cursor: " ▉"
  fresh_final_after_seconds: 60   # Telegram: reenvia mensagem final após N segundos
```

---

## Chat em Grupo e DMs

```yaml
group_sessions_per_user: true     # Sessões isoladas por usuário em grupos
unauthorized_dm_behavior: pair    # pair | ignore
```

---

## Fuso Horário

```yaml
timezone: "America/Sao_Paulo"     # IANA identifier
```

---

## Quick Commands

```yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  restart:
    type: alias
    target: /gateway restart
```

---

## Referência de Variáveis de Ambiente (`.env`)

```bash
# Anthropic (obrigatório para Claude)
ANTHROPIC_API_KEY=sk-ant-...

# OpenRouter (alternativa)
OPENROUTER_API_KEY=sk-or-...

# Telegram Gateway
TELEGRAM_BOT_TOKEN=...
TELEGRAM_ALLOWED_USERS=2129471333,7175937401   # IDs numéricos (não username)
TELEGRAM_HOME_CHANNEL=-100xxxxxxxxx             # Grupo para cron outputs

# Telegram Webhook (alternativa ao polling)
TELEGRAM_WEBHOOK_URL=https://meu-app.fly.dev/telegram
TELEGRAM_WEBHOOK_SECRET=...

# Terminal backend
TERMINAL_BACKEND=docker

# Working directory do gateway
MESSAGING_CWD=/path/para/projeto

# Discord (se usar)
DISCORD_BOT_TOKEN=...
DISCORD_GUILD_ID=...
```

---

## Comandos de Gerenciamento

```bash
hermes setup --portal       # Setup OAuth inicial
hermes config               # Ver config atual
hermes config edit          # Abrir em editor
hermes config set KEY VAL   # Setar um valor
hermes config check         # Verificar opções faltando
hermes config migrate       # Adicionar opções novas
hermes model                # Picker interativo de modelo
hermes doctor               # Diagnóstico completo
hermes tools                # Enable/disable toolsets

# Gateway
hermes gateway              # Iniciar gateway (todos os canais configurados)
hermes gateway run          # Mesmo que acima
hermes gateway setup        # Wizard de configuração
hermes gateway install      # Instalar como serviço persistente (systemd/launchd)
hermes gateway status       # Ver status do gateway
```

---

## Referenciando variáveis de ambiente no config.yaml

```yaml
auxiliary:
  vision:
    api_key: ${GOOGLE_API_KEY}
    base_url: ${CUSTOM_VISION_URL}

# Múltiplas refs na mesma string:
server:
  url: "${HOST}:${PORT}"
```

Vars não definidas permanecem como literal.

---

## Limites de Iteração (Budget Pressure)

O agente injeta avisos automáticos no contexto:

```
70% do budget → [BUDGET: 63/90. 27 iterations left. Start consolidating.]
90% do budget → [BUDGET WARNING: 81/90. Only 9 left. Respond NOW.]
```
