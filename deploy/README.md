# Agent VPS test deploy

**[Русская версия](README.ru.md)**

This stack can run:
- `agent` (gRPC + ADNL)
- `prometheus`
- `grafana`
- `loki`

## Production / VPS (GHCR) — recommended

Use this on VPS with limited disk: the image is published by the `Publish` workflow on every push to `master` (`ghcr.io/mytonprovider/mytonprovider-agent:<tag>`, the tag is the first 7 characters of the commit), on the VPS only **pull** the image — no `golang` build cache or source compile on the server.

| Step | Where | Action |
|------|--------|--------|
| 1 | GitHub | Merge to `master` → `Publish` workflow pushes the image |
| 2 | VPS | Repo clone (or copy `deploy/` + secrets only), `task`, Docker |
| 3 | VPS | `task hub:init` → edit `deploy/.env.hub` → `task hub:up` or `task hub:stack:up` |

**Why:** `task deploy:up` runs `docker compose ... --build`, which pulls `golang:bookworm` and compiles on the VPS. Hub compose files (`docker-compose.hub.yml`, `docker-compose.hub.stack.yml`) use `image: ${AGENT_IMAGE}` with `pull_policy: always` — no agent build on VPS.

### A) Build and push by hand (dev machine, fallback)

```bash
docker login ghcr.io
AGENT_IMAGE=ghcr.io/mytonprovider/mytonprovider-agent:<tag> task image:build:push
```

Separate steps: `task image:build`, `task image:push` (both require `AGENT_IMAGE`).

### B) Agent only on VPS

```bash
task hub:init
nano deploy/.env.hub
task hub:up
```

Place TLS files before `up` (see [secrets/README.md](secrets/README.md)):
- `deploy/secrets/server.crt`
- `deploy/secrets/server.key`
- run **`chmod 644`** on both after copying

In `.env.hub` set:
- `AGENT_IMAGE=ghcr.io/mytonprovider/mytonprovider-agent:<tag>`
- `AGENT_AUTH_TOKEN=...`
- TLS paths `/run/secrets/server.crt` and `/run/secrets/server.key`
- `AGENT_METRICS_LISTEN_ADDR=:9090` and `AGENT_LOKI_URL=http://loki:3100` if you use the full stack later

### C) Agent + monitoring on VPS (still no build)

```bash
task hub:stack:up
task hub:stack:ps
```

UIs:
- Prometheus: `http://<vps-ip>:${PROMETHEUS_PORT}`
- Grafana: `http://<vps-ip>:${GRAFANA_PORT}`

Stop: `task hub:stack:down` or `task hub:down`.

### Hub Task reference

| Task | Description |
|------|-------------|
| `image:build` | Build image locally (`AGENT_IMAGE` required) |
| `image:push` | Push existing tag |
| `image:build:push` | Build + push |
| `hub:init` | Create `.env.hub` from example |
| `hub:up` / `down` / `ps` / `logs` | Agent only, pull image |
| `hub:stack:up` / `down` / `ps` / `logs` | Agent + Prometheus + Grafana + Loki |

### Which compose file

| File | Task | Typical container name | Stack |
|------|------|------------------------|--------|
| `docker-compose.hub.yml` | `hub:up` | `agent-1` | agent only |
| `docker-compose.hub.stack.yml` | `hub:stack:up` | `deploy-agent-1` | agent + monitoring |

Repo path on VPS is often `~/mytonprovider-agent`. Run compose from `deploy/`.

### Connecting to a new coordinator

1. New `server.crt` / `server.key` (new CA) → `deploy/secrets/`, `chmod 644`.
2. `AGENT_AUTH_TOKEN` in `.env.hub` matches the coordinator token (see [mytonprovider-backend](https://github.com/mytonprovider/mytonprovider-backend)).
3. On coordinator: `AGENT_ENDPOINTS` lists this agent’s IP (same as cert SAN).
4. Recreate agent container — see [secrets/README.md](secrets/README.md).

### Smoke checks (hub)

```bash
task hub:stack:ps
docker logs deploy-agent-1 --tail 5
grpc_health_probe -addr=127.0.0.1:${AGENT_GRPC_PORT:-8443} -tls -tls-no-verify
```

If coordinator gets `connection refused` on `:8443`, check agent logs for `server.key` permission errors.

---

## Build on VPS (dev / debugging)

Heavier on disk and CPU: compiles the agent image on the server via `docker compose ... --build`.

Uses `deploy/docker-compose.yml` and `deploy/.env`.

### 1) Prepare

From repository root:

```bash
task deploy:init
nano deploy/.env
```

TLS files:
- `deploy/secrets/server.crt`
- `deploy/secrets/server.key`

### 2) Start

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

### 4) Logs / stop

```bash
task deploy:logs
task deploy:down
```
