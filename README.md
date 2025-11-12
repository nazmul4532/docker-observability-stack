# Docker Observability Stack (Grafana + Prometheus + Loki + Tempo + OTel Collector)

A tiny, batteries-included observability stack you can run with Docker Compose. It renders configs from templates, wires up Grafana datasources, and exposes OTLP (gRPC/HTTP) so apps can ship traces, metrics, and logs through the OpenTelemetry Collector to Tempo/Prometheus/Loki.

Repo: `https://github.com/nazmul4532/docker-observability-stack`

---

## What’s inside

- **Grafana** (web UI)  
- **Prometheus** (metrics DB & scraper)  
- **Loki** (logs)  
- **Tempo** (traces)  
- **OpenTelemetry Collector** (ingest OTLP, export to the backends)  
- **`init-configs` one-shot container** that `envsubst`s templates → `_rendered/`  
  - `grafana/provisioning/datasources/datasources.yml.tmpl`  
  - `prometheus/prometheus.yml.tmpl`

> Rendered files live in `./_rendered` and are mounted read-only by services, so you can safely regenerate them without bouncing the world.

---

## Prerequisites

- Docker & Docker Compose
- A terminal in the repo root
- Ports available (defaults):  
  - Grafana `3000`, Prometheus `9090`, Loki HTTP `3100`, Tempo HTTP `3200`  
  - OTel Collector: OTLP gRPC `4317`, OTLP HTTP `4318`, Prom exporter `8889`
  - Modify .env file if these defaults are  not available

---

## Configuration

All tunables are in **`.env`** (already provided):

```env
# UI ports
PROMETHEUS_PORT=9090
GRAFANA_PORT=3000
LOKI_PORT=3100

# OTel Collector listener ports
OTEL_GRPC_PORT=4317
OTEL_HTTP_PORT=4318
OTEL_PROM_EXPORTER_PORT=8889

# Loki internal ports
LOKI_HTTP_PORT=3100
LOKI_GRPC_PORT=9096

# Tempo internal ports
TEMPO_HTTP_PORT=3200
TEMPO_OTLP_GRPC_PORT=4317
TEMPO_OTLP_HTTP_PORT=4318

# Grafana auth
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin

# Datasource URLs used during template rendering
PROMETHEUS_DS_URL=http://prometheus:${PROMETHEUS_PORT}
LOKI_DS_URL=http://loki:${LOKI_HTTP_PORT}
TEMPO_DS_URL=http://tempo:${TEMPO_HTTP_PORT}
```

> Change ports or passwords here before first run if you need to.

---

## Quick start

```bash
# from repo root
rm -rf _rendered
mkdir -p _rendered/grafana/datasources _rendered/prometheus

docker compose up -d
```

- The **`init-configs`** job runs once to render templates into `_rendered/`.
- Then Grafana/Prometheus/Loki/Tempo/OTel Collector start and mount those rendered files.

### Stop & clean

```bash
docker compose down -v
rm -rf _rendered
mkdir -p _rendered/grafana/datasources _rendered/prometheus
```

> Recreate `_rendered` folders so the next `up -d` can mount them cleanly.

---

## URLs

- Grafana: `http://localhost:3000` (admin / admin by default)  
- Prometheus: `http://localhost:9090`  
- Loki (HTTP API): `http://localhost:3100`  
- Tempo (status): `http://localhost:3200`  
- OTel Collector (OTLP):
  - gRPC: `0.0.0.0:4317`
  - HTTP: `0.0.0.0:4318`

---

## Health checks

```bash
# Grafana
curl -s http://localhost:${GRAFANA_PORT}/api/health

# Prometheus (simple page check)
curl -sI http://localhost:${PROMETHEUS_PORT}/ | head -n1

# Loki
curl -s "http://localhost:${LOKI_PORT}/ready"   # or /metrics

# Tempo
curl -s "http://localhost:${TEMPO_HTTP_PORT}/"  # or /metrics

# OTel Collector Prometheus exporter (scraped by Prometheus)
curl -s "http://localhost:${OTEL_PROM_EXPORTER_PORT}/metrics" | head
```

Inspect logs if something restarts:

```bash
docker logs loki --tail=200
docker logs tempo --tail=200
docker logs otel-collector --tail=200
```

---

## How the templates work

- **Grafana datasources** (`grafana/provisioning/datasources/datasources.yml.tmpl`) are rendered with the URLs from `.env`, wiring **Prometheus**, **Loki**, and **Tempo** automatically.
- **Prometheus** (`prometheus/prometheus.yml.tmpl`) is rendered to scrape:
  - itself (`prometheus:9090`)
  - the **OTel Collector’s** Prometheus exporter (`otel-collector:${OTEL_PROM_EXPORTER_PORT}`)

Rendered files end up in `./_rendered`:
```
_rendered/
  grafana/datasources/datasources.yml
  prometheus/prometheus.yml
```

---

## Shipping telemetry from an app (example)

### Option A — Java app with the OTel Java Agent

1) Add the agent and point OTLP to the collector (already included in `./otel/opentelemetry-javaagent.jar`):

```bash
docker run -d --name spring-jvm-app \
  --restart unless-stopped \
  --network observability \
  -p 8080:8080 \
  -e OTEL_SERVICE_NAME="spring-jvm-app" \
  -e OTEL_TRACES_SAMPLER="always_on" \
  -e OTEL_TRACES_EXPORTER="otlp" \
  -e OTEL_METRICS_EXPORTER="otlp" \
  -e OTEL_LOGS_EXPORTER="otlp" \
  -e OTEL_EXPORTER_OTLP_PROTOCOL="grpc" \
  -e OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317" \
  -e JAVA_TOOL_OPTIONS='-javaagent:/otel/opentelemetry-javaagent.jar' \
  -v "$PWD/otel:/otel:ro" \
  nazmul4532brainstation/springboot-jvm:latest
```

2) Hit your app to generate activity, then:
- **Traces**: Grafana → Explore → Tempo  
- **Logs**: Grafana → Explore → Loki  
- **Metrics**: Grafana → Explore → Prometheus (or build a dashboard)

### Option B — Any language SDK

- Point your exporter to `http://localhost:4317` (gRPC) or `http://localhost:4318` (HTTP).
- Export traces/metrics/logs over OTLP.
- No app changes needed on the backend side; OTel Collector fans out to Tempo/Loki/Prometheus.

---

## Troubleshooting

- **Grafana says no datasources**  
  The `init-configs` job might not have created `_rendered` files. Re-run:
  ```bash
  docker compose down -v
  rm -rf _rendered && mkdir -p _rendered/grafana/datasources _rendered/prometheus
  docker compose up -d
  ```
- **Ports already in use**  
  Change them in `.env` and `docker compose down -v && docker compose up -d`.
- **No traces/logs/metrics appear**  
  - Verify your app exports to `otel-collector:4317/4318` (inside the `observability` network) or `localhost` from the host.
  - Check `docker logs otel-collector`.
- **Prometheus shows targets down**  
  - Confirm `http://localhost:${OTEL_PROM_EXPORTER_PORT}/metrics` returns data.
  - Check the rendered `./_rendered/prometheus/prometheus.yml`.

---

## Useful Docker commands

```bash
# See what’s running
docker ps

# Tail a service’s logs
docker logs -f grafana
docker logs -f prometheus
docker logs -f loki
docker logs -f tempo
docker logs -f otel-collector
```

---

## Notes

- Loki & Tempo use local disk volumes by default and are single-node configs appropriate for dev/demo.
- The stack runs on a user-defined network named **`observability`**.
- You can safely customize `.env` and re-render without touching your data volumes.
