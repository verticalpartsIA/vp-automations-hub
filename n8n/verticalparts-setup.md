# n8n — Setup VerticalParts (Produção)

> Estado em: 2026-05-28

---

## Container em produção

```
Nome:       vpautomation-n8n
Imagem:     n8nio/n8n:latest
Porta:      5678 (interna)
Rede:       vp-automation
Path:       /docker/vpautomation-n8n/
```

---

## Acesso

- **URL interna:** `http://72.61.48.156:5678`
- Para acesso externo via Traefik, configurar labels e subdomínio

---

## Comandos operacionais

```bash
# Status
docker ps | grep n8n

# Logs
docker logs vpautomation-n8n -f --tail 50

# Reiniciar
cd /docker/vpautomation-n8n && docker compose restart

# Entrar no container
docker exec -it vpautomation-n8n sh

# Backup dos workflows
docker exec vpautomation-n8n n8n export:workflow --all --output=/tmp/workflows.json
docker cp vpautomation-n8n:/tmp/workflows.json ./n8n-workflows-backup.json
```

---

## Integrações planejadas

### Fase 1 — Notificações

- [ ] Webhook Evolution API → Notificação Slack/Telegram quando chega mensagem WhatsApp
- [ ] Novo ticket pv360 → Notificação para a equipe

### Fase 2 — Automação

- [ ] Triagem automática de tickets por categoria (Claude)
- [ ] Relatório diário de atendimentos

### Fase 3 — Integração ERP

- [ ] Sincronização pedidos ERP → Supabase pv360
- [ ] Status de entrega → WhatsApp automático

---

## Credenciais necessárias

| Serviço | Tipo | Status |
|---------|------|--------|
| Evolution API (pv360) | HTTP Header Auth | 🔜 Configurar |
| Supabase pv360 | API Key | 🔜 Configurar |
| Anthropic Claude | API Key | 🔜 Configurar |
| Telegram Bot | Bot Token | 🔜 Configurar |
