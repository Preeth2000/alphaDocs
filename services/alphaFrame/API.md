---
service: alphaFrame
page: API
tags:
  - service/alphaFrame
  - api
---

# alphaFrame — API

[[services/alphaFrame/alphaFrame|alphaFrame]] · [[services/alphaFrame/Architecture|Architecture]] · [[services/alphaFrame/Interactions|Interactions]] · [[services/alphaFrame/Data|Data]] · [[services/alphaFrame/Config|Config]]

---

> [!note] No application API
> alphaFrame has no FastAPI / application-layer endpoints. It exposes infrastructure ports consumed by other services and operators.

---

## Exposed Ports

See [[reference/Ports-and-Endpoints]] for the complete port map. Key ports:

| Port | Service | Access | Consumer |
|---|---|---|---|
| `443` | Nginx HTTPS | External | Browsers, [[services/alphaLink/alphaLink\|alphaLink]], CI |
| `80` | Nginx HTTP | External | → redirects to 443 |
| `5000` | MLflow | Internal | [[services/alphaGen/alphaGen\|alphaGen]], [[services/alphaTrade/alphaTrade\|alphaTrade]] |
| `5432` | PostgreSQL | Internal | All app services |
| `6379` | Redis | Internal | All app services |
| `9000` | MinIO S3 API | Internal | [[services/alphaGen/alphaGen\|alphaGen]], [[services/alphaTrade/alphaTrade\|alphaTrade]] |
| `9001` | MinIO Console | Internal | Operators |
| `3001` | Grafana | Internal | Operators |
| `9090` | Prometheus | Internal | OTel Collector (remote write), operators |
| `9093` | Alertmanager | Internal | Prometheus (alerts) |
| `3100` | Loki | Internal | OTel Collector |
| `3200` | Tempo | Internal | OTel Collector |
| `4317` | OTel Collector gRPC | Internal | All app services |
| `4318` | OTel Collector HTTP | Internal | All app services |

---

## Nginx Routes (:443)

| Method | Path | Upstream | Rate-limited | Notes |
|---|---|---|---|---|
| ALL | `/auth/` | `alphakey-api:8000/auth/` | ✅ | User-facing auth only. `/auth/internal/*` blocked. |
| ALL | `/alphagen/` | `alphagen-api:8000/` | ✅ | Standard reverse proxy |
| `GET` | `/alphagen/runs/events` | `alphagen-api:8000/runs/events` | ❌ | SSE stream — buffering off, timeout 3600s |
| `GET` | `/stream` | `alphatrade:8081/stream` | ❌ | SSE stream — buffering off, timeout 3600s |
| ALL | `/` | `alphatrade:8081` | ✅ | Default — all other alphaTrade routes |

**Rate limit:** `${NGINX_RATE_LIMIT}` (default `120r/m`) with burst `${NGINX_BURST}` (default `60`) — configurable via env.

---

## Prometheus Scrape Targets

Prometheus pulls metrics from these targets every 15s:

| Target | Address | Metrics source |
|---|---|---|
| prometheus | `prometheus:9090` | Self-metrics |
| node-exporter | `node-exporter:9100` | Host CPU, memory, disk |
| cadvisor | `cadvisor:8080` | Container resource usage |
| postgres-exporter | `postgres-exporter:9187` | PostgreSQL query stats |
| redis-exporter | `redis-exporter:9121` | Redis keyspace, commands |
| alphagen-flower | `alphagen-flower:5555/metrics` | Celery task counts |
| alphatrade | `alphatrade:9090` | Business metrics (signals, orders, P&L) |
| otel-collector | `otel-collector:8888` | Collector pipeline metrics |

---

## Alert Rules (Prometheus → Alertmanager)

| Alert | Condition | Duration | Severity |
|---|---|---|---|
| `ServiceDown` | `up == 0` for any scrape target | 1m | critical |
| `HighErrorRate` | HTTP 5xx rate > 5% (5m window) | 5m | warning |
| `HighLatencyP99` | p99 > 2s (5m window) | 5m | warning |
| `CeleryTaskFailures` | Failure rate > 0 (5m window) | 5m | warning |
| `PostgresDown` | postgres-exporter `up == 0` | 1m | critical |
| `HighMemoryUsage` | Container mem > 80% of limit | 5m | warning |
| `DiskPressure` | Host disk > 85% | 5m | warning |

Alerts route to `local-log` receiver → HTTP POST to `localhost:9999/alert`. Replace with Slack/PagerDuty by updating `alertmanager/config.yaml` receiver only.
