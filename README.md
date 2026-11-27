# mytonprovider-agent

**[Русская версия](README.ru.md)**

Check agent for [mytonprovider.org](https://mytonprovider.org): gRPC service (`RunChecks`, `RunStorageRates`), ADNL transport to providers, optional Prometheus metrics and Loki push. The coordinator from [mytonprovider-backend](https://github.com/mytonprovider/mytonprovider-backend) calls it over gRPC with TLS.

## Repository layout

```
├── cmd/agent/          # entrypoint
├── internal/           # checks, gRPC server, ADNL transport, metrics
├── contracts/          # proto + generated Go (separate Go module)
├── deploy/             # Docker Compose for VPS
├── observability/      # local Prometheus + Grafana + Loki (dev)
├── tests/              # gRPC and Postman tests
└── Taskfile.yml        # proto, tests, deploy tasks
```

## Development

**Prerequisites:** Go 1.26+, [Task](https://taskfile.dev), `grpcurl`, `openssl`, `protoc` (for `task proto:gen`).

```bash
# Regenerate protobuf (after editing contracts/proto)
task proto:gen
task proto:check

# Run agent locally with test TLS + token
task run:test
```

In another terminal — gRPC smoke tests (agent must already be running):

```bash
task test:smoke
```

Live checks against real TON data: [tests/grpc/README.md](tests/grpc/README.md) · [RU](tests/grpc/README.ru.md).

## Docker deploy (VPS)

Docs: [EN](deploy/README.md) · [RU](deploy/README.ru.md). The image is published by the `publish` job in `CI` after checks on every push to `master`: `ghcr.io/mytonprovider/mytonprovider-agent:<tag>`, the tag is the first 7 characters of the commit. Manual `task image:build:push` stays as a fallback.

```bash
# VPS
task hub:init && nano deploy/.env.hub && task hub:stack:up
```

**Agent build on VPS** (dev / debugging only):

```bash
task deploy:init
nano deploy/.env
task deploy:up
```

## Local observability

Prometheus + Grafana + Loki for development: [EN](observability/prometheus-grafana/README.md) · [RU](observability/prometheus-grafana/README.ru.md).

## TLS setup (no DNS)

This service expects TLS cert and key files at startup. Use your own CA (usually on the **coordinator** host) and issue per-agent server certificates with matching **IP SAN** entries.

**Important:** SAN IP must match what the coordinator uses in `AGENT_ENDPOINTS`. For **Tailscale**, use the Tailscale IPv4 (`100.x.x.x`), not necessarily the VPS public IP.

After copying to an agent: `chmod 644` on `server.key` — see [deploy/secrets/README.md](deploy/secrets/README.md).

### 1) Generate CA (once, on coordinator)

```bash
mkdir -p certs/ca
openssl genrsa -out certs/ca/ca.key 4096
openssl req -x509 -new -nodes -key certs/ca/ca.key -sha256 -days 3650 -out certs/ca/ca.crt -subj "/CN=mytonprovider-root-ca"
```

Keep `certs/ca/ca.key` private. Never commit it to git.

### 2) Issue server cert for agent IP

Create OpenSSL config for each agent. Replace `203.0.113.10` with the IP from `AGENT_ENDPOINTS` (public or Tailscale):

```bash
cat > certs/agent-eu.cnf <<'EOF'
[req]
distinguished_name = dn
req_extensions = v3_req
prompt = no

[dn]
CN = agent-eu

[v3_req]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth
keyUsage = digitalSignature,keyEncipherment

[alt_names]
IP.1 = 203.0.113.10
EOF
```

Generate key/csr and sign with your CA:

```bash
mkdir -p certs/agent-eu
openssl genrsa -out certs/agent-eu/server.key 2048
openssl req -new -key certs/agent-eu/server.key -out certs/agent-eu/server.csr -config certs/agent-eu.cnf
openssl x509 -req -in certs/agent-eu/server.csr -CA certs/ca/ca.crt -CAkey certs/ca/ca.key -CAcreateserial -out certs/agent-eu/server.crt -days 365 -sha256 -extensions v3_req -extfile certs/agent-eu.cnf
```

### 3) Runtime env for agent container

```bash
AGENT_LISTEN_ADDR=:8443
AGENT_ID=agent-eu
AGENT_LOCATION=eu
AGENT_AUTH_TOKEN=replace_with_long_random_token
AGENT_TLS_CERT_FILE=/run/secrets/server.crt
AGENT_TLS_KEY_FILE=/run/secrets/server.key
AGENT_MAX_CONCURRENT_PROVIDERS=30
# RunStorageRates / DHT + provider transport (do not share UDP port with coordinator on same host)
AGENT_TON_CONFIG_URL=https://ton-blockchain.github.io/global.config.json
AGENT_ADNL_PORT=16167
# Optional: 64 hex chars = 32-byte Ed25519 seed; if unset, a random key is generated at startup
# AGENT_ADNL_KEY=
# Optional: parallel GetStorageRates cap; if unset, uses AGENT_MAX_CONCURRENT_PROVIDERS
# AGENT_MAX_CONCURRENT_RATES=
# Optional: after each RunChecks, push summary rows to Loki (Grafana tables). No trailing slash. Example: http://127.0.0.1:3100
# AGENT_LOKI_URL=http://127.0.0.1:3100
# Optional: Prometheus /metrics (HTTP). Empty = disabled. Use 0.0.0.0:9090 so Prometheus-in-Docker can scrape via host.docker.internal (127.0.0.1 is host-only).
# AGENT_METRICS_LISTEN_ADDR=0.0.0.0:9090
```

Copy to `deploy/secrets/` on the agent host, then `chmod 644 server.crt server.key`. Files are mounted read-only at `/run/secrets/`.

### 4) gRPC health (`grpc.health.v1.Health`)

The agent registers the standard **gRPC Health Checking** service on the same TLS port as `RunChecks` / `RunStorageRates`.

- **`grpc.health.v1.Health/Check`** does **not** require the coordinator `Authorization: Bearer` header (so probes can stay tokenless).
- After successful startup the overall service name `""` is **`SERVING`**; on process shutdown cleanup sets **`NOT_SERVING`** before closing TON transport.

Example with [grpc_health_probe](https://github.com/grpc-ecosystem/grpc-health-probe) against a test agent (adjust CA / `-tls-no-verify` for your setup):

```bash
grpc_health_probe -addr=127.0.0.1:8443 -tls -tls-no-verify
```

The gRPC server also enables **HTTP/2 keepalive** (server pings idle clients about once per minute; clients may ping at most about every 10s when `PermitWithoutStream` applies) so long-lived connections are less likely to be dropped by NAT/firewalls.

### 5) Coordinator trust

Coordinator gRPC client must trust the CA cert (`ca.crt`) that signed agent server certs.
Do not copy CA private key to any server.

## License

Apache-2.0 — see [LICENSE](LICENSE).
