# Docker — Comandos Úteis

Referência rápida para operações Docker no VPS.

---

## Containers

```bash
# Listar containers ativos
docker ps

# Listar todos (incluindo parados)
docker ps -a

# Stats em tempo real
docker stats

# Stats snapshot (sem ficar rodando)
docker stats --no-stream

# Stats de container específico
docker stats vpautomation-hermes --no-stream
```

---

## Logs

```bash
# Logs de um container (últimas 100 linhas)
docker logs vpautomation-hermes --tail 100

# Logs em tempo real
docker logs vpautomation-hermes -f

# Logs com timestamp
docker logs vpautomation-hermes -f --timestamps

# Logs das últimas 2 horas
docker logs vpautomation-hermes --since 2h
```

---

## Execução e Terminal

```bash
# Entrar no container (bash)
docker exec -it vpautomation-hermes bash

# Entrar no container (sh — para Alpine)
docker exec -it evolution_api sh

# Executar comando único
docker exec vpautomation-hermes cat /opt/data/config.yaml

# Copiar arquivo do container para host
docker cp vpautomation-hermes:/opt/data/gateway.log ./gateway.log

# Copiar arquivo do host para container
docker cp ./SOUL.md vpautomation-hermes:/opt/data/SOUL.md
```

---

## Gerenciamento de Containers

```bash
# Iniciar container parado
docker start CONTAINER_NAME

# Parar container
docker stop CONTAINER_NAME

# Reiniciar
docker restart CONTAINER_NAME

# Remover container (parado)
docker rm CONTAINER_NAME

# Forçar remoção (mesmo rodando)
docker rm -f CONTAINER_NAME
```

---

## Docker Compose

```bash
# Subir serviços em background
docker compose up -d

# Ver logs (todos os serviços)
docker compose logs -f

# Parar e remover containers (mantém volumes)
docker compose down

# Parar, remover containers E volumes
docker compose down -v

# Recriar containers forçando rebuild
docker compose up -d --force-recreate

# Atualizar imagens e reiniciar
docker compose pull && docker compose up -d

# Restart de serviço específico
docker compose restart hermes
```

---

## Imagens

```bash
# Listar imagens
docker images

# Baixar imagem
docker pull atendai/evolution-api:v2.2.3

# Remover imagem
docker rmi atendai/evolution-api:v2.2.3

# Limpar imagens não usadas
docker image prune -f

# Limpar TUDO não usado (cuidado!)
docker system prune -f
```

---

## Volumes

```bash
# Listar volumes
docker volume ls

# Inspecionar volume
docker volume inspect evolution_instances

# Remover volume (DESTRUTIVO — perda de dados!)
docker volume rm evolution_instances

# Backup de volume
docker run --rm \
  -v VOLUME_NAME:/source \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup-$(date +%Y%m%d).tar.gz -C /source .

# Restaurar volume
docker run --rm \
  -v VOLUME_NAME:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup-20260528.tar.gz -C /target
```

---

## Redes

```bash
# Listar redes
docker network ls

# Inspecionar rede (ver containers conectados)
docker network inspect vp-automation

# Criar rede
docker network create vp-automation

# Conectar container a rede
docker network connect vp-automation evolution-api

# Desconectar
docker network disconnect vp-automation CONTAINER
```

---

## Comandos específicos VP

```bash
# ─── Hermes ───────────────────────────────────────────────────────────────────
docker exec vpautomation-hermes tail -f /opt/data/logs/gateway.log
docker restart vpautomation-hermes
echo "" > /docker/vpautomation-hermes/data/memories/main.md
rm -f /docker/vpautomation-hermes/data/sessions/*

# ─── Evolution API (pv360) ───────────────────────────────────────────────────
docker logs evolution-api --tail 50
docker restart evolution-api
# Status da instância pv360:
curl http://localhost:8080/instance/connectionState/pv360 -H "apikey: suporte123"

# ─── n8n ──────────────────────────────────────────────────────────────────────
docker logs vpautomation-n8n --tail 50
cd /docker/vpautomation-n8n && docker compose restart

# ─── Infraestrutura ──────────────────────────────────────────────────────────
# Ver uso de disco
df -h
docker system df

# Ver uso de memória por container
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}"
```
