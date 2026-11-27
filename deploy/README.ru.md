# Тестовый деплой агента на VPS

**[English version](README.md)**

Стек может включать:
- `agent` (gRPC + ADNL)
- `prometheus`
- `grafana`
- `loki`

## Production / VPS (GHCR) — рекомендуется

Для VPS с ограниченным диском: образ публикует задание `publish` в `CI` после проверок при каждом push в `master` (`ghcr.io/mytonprovider/mytonprovider-agent:<tag>`, тег — первые 7 символов коммита), на VPS только **pull** — без кэша `golang` и сборки исходников на сервере.

| Шаг | Где | Действие |
|-----|-----|----------|
| 1 | GitHub | Merge в `master` → задание `publish` в `CI` публикует образ |
| 2 | VPS | Клон репо (или только `deploy/` + secrets), `task`, Docker |
| 3 | VPS | `task hub:init` → правка `deploy/.env.hub` → `task hub:up` или `task hub:stack:up` |

**Зачем:** `task deploy:up` делает `docker compose ... --build` — на VPS тянется `golang:bookworm` и идёт компиляция. Hub compose (`docker-compose.hub.yml`, `docker-compose.hub.stack.yml`) используют `image: ${AGENT_IMAGE}` и `pull_policy: always` — без сборки агента на VPS.

### A) Сборка и push руками (dev-машина, запасной путь)

```bash
docker login ghcr.io
AGENT_IMAGE=ghcr.io/mytonprovider/mytonprovider-agent:<tag> task image:build:push
```

По отдельности: `task image:build`, `task image:push` (нужен `AGENT_IMAGE`).

### B) Только агент на VPS

```bash
task hub:init
nano deploy/.env.hub
task hub:up
```

Перед `up` положите TLS (см. [secrets/README.ru.md](secrets/README.ru.md)):
- `deploy/secrets/server.crt`
- `deploy/secrets/server.key`
- **`chmod 644`** на оба файла после копирования

В `.env.hub`:
- `AGENT_IMAGE=ghcr.io/mytonprovider/mytonprovider-agent:<tag>`
- `AGENT_AUTH_TOKEN=...`
- пути TLS `/run/secrets/server.crt` и `/run/secrets/server.key`
- `AGENT_METRICS_LISTEN_ADDR=:9090` и `AGENT_LOKI_URL=http://loki:3100`, если позже поднимете полный stack

### C) Агент + мониторинг на VPS (без сборки)

```bash
task hub:stack:up
task hub:stack:ps
```

UI:
- Prometheus: `http://<vps-ip>:${PROMETHEUS_PORT}`
- Grafana: `http://<vps-ip>:${GRAFANA_PORT}`

Остановка: `task hub:stack:down` или `task hub:down`.

### Задачи Task (hub)

| Task | Описание |
|------|----------|
| `image:build` | Сборка образа локально (нужен `AGENT_IMAGE`) |
| `image:push` | Push тега |
| `image:build:push` | Сборка + push |
| `hub:init` | Создать `.env.hub` из example |
| `hub:up` / `down` / `ps` / `logs` | Только агент, pull образа |
| `hub:stack:up` / `down` / `ps` / `logs` | Агент + Prometheus + Grafana + Loki |

### Какой compose-файл

| Файл | Task | Имя контейнера (типично) | Состав |
|------|------|--------------------------|--------|
| `docker-compose.hub.yml` | `hub:up` | `agent-1` | только агент |
| `docker-compose.hub.stack.yml` | `hub:stack:up` | `deploy-agent-1` | агент + Prometheus + Grafana + Loki |

На VPS репозиторий часто лежит в `~/mytonprovider-agent` (не обязательно `/opt/...`). Команды compose запускайте из `deploy/`.

### Подключение к новому coordinator

1. Новые `server.crt` / `server.key` (CA нового coordinator) → `deploy/secrets/`, `chmod 644`.
2. `AGENT_AUTH_TOKEN` в `.env.hub` = токен координатора (см. [mytonprovider-backend](https://github.com/mytonprovider/mytonprovider-backend)).
3. На coordinator: `AGENT_ENDPOINTS=<tailscale-ip-этого-агента>:8443,...`
4. Перезапуск агента из `deploy/` (см. [secrets/README.ru.md](secrets/README.ru.md)).

Проверка: в логах `agent gRPC server started`, затем `RunChecks completed` (запросы с coordinator).

### Smoke checks (hub)

```bash
task hub:stack:ps   # или hub:ps
docker logs deploy-agent-1 --tail 5   # или agent-1
grpc_health_probe -addr=127.0.0.1:${AGENT_GRPC_PORT:-8443} -tls -tls-no-verify
```

Если `connection refused` на `8443` с coordinator — см. логи агента на `permission denied` для `server.key`.

---

## Сборка на VPS (dev / отладка)

Тяжелее по диску и CPU: образ агента собирается на сервере через `docker compose ... --build`.

Файлы: `deploy/docker-compose.yml`, `deploy/.env`.

### 1) Подготовка

Из корня репозитория:

```bash
task deploy:init
nano deploy/.env
```

TLS:
- `deploy/secrets/server.crt`
- `deploy/secrets/server.key`

### 2) Запуск

```bash
task deploy:up
```

### 3) Smoke checks

```bash
task deploy:ps
grpc_health_probe -addr=127.0.0.1:${AGENT_GRPC_PORT:-8443} -tls -tls-no-verify
```

- Prometheus: `http://<vps-ip>:${PROMETHEUS_PORT}`
- Grafana: `http://<vps-ip>:${GRAFANA_PORT}`

### 4) Логи / остановка

```bash
task deploy:logs
task deploy:down
```
