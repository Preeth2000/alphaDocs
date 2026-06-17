---
page: Ports-and-Endpoints
tags:
  - reference
  - ports
  - networking
---

# Ports and Endpoints

[[README]] · [[reference/Glossary]] · [[reference/Event-Channels]]

All ports and internal service addresses used across projectAlpha.

---

## External Ports (Host-exposed)

| Port | Service | Protocol | Access |
|---|---|---|---|
| `80` | [[alphaFrame\|Nginx]] | HTTP | Public — redirects to 443 |
| `443` | [[alphaFrame\|Nginx]] | HTTPS (self-signed) | Public — reverse proxy (sole ingress) |

All other ports are bound to `127.0.0.1` (localhost only) — not reachable from LAN or external networks.

| Port | Service | Protocol | Localhost-only |
|---|---|---|---|
| `3001` | Grafana | HTTP | ✅ — observability UI |
| `5000` | MLflow | HTTP | ✅ — experiment tracking + registry UI |
| `5432` | PostgreSQL | TCP | ✅ — SQL database |
| `6379` | Redis | TCP | ✅ — Celery broker / pub/sub (auth required) |
| `9000` | MinIO | HTTP | ✅ — S3-compatible API |
| `9001` | MinIO | HTTP | ✅ — MinIO web console |
| `9090` | Prometheus | HTTP | ✅ — metrics query UI + remote write |
| `9093` | Alertmanager | HTTP | ✅ — alert routing UI |
| `3100` | Loki | HTTP | ✅ — log API |
| `3200` | Tempo | HTTP | ✅ — trace API |
| `4317` | OTel Collector | gRPC | ✅ — OTLP trace/metric/log ingress |
| `4318` | OTel Collector | HTTP | ✅ — OTLP HTTP ingress |

---

## Internal Services (Platform Network)

All services communicate via Docker `platform` bridge network using service names as hostnames.

| Service | Internal Host | Port(s) | Protocol | Purpose |
|---|---|---|---|---|
| [[alphaFrame\|postgres]] | `postgres` | `5432` | TCP/SQL | Primary DB — 4 databases |
| [[alphaFrame\|redis]] | `redis` | `6379` | TCP | Celery broker (db0), results (db1), pub/sub |
| [[alphaFrame\|minio]] | `minio` | `9000` | HTTP/S3 | Object storage |
| [[alphaFrame\|MLflow]] | `mlflow` | `5000` | HTTP | Experiment tracking + model registry |
| [[alphaGen\|alphaGen API]] | `alphagen-api` | `8000` | HTTP | Training jobs REST + SSE |
| [[alphaGen\|alphaGen Worker]] | `alphagen-worker` | — | (Celery) | Training job executor |
| [[alphaGen\|Flower]] | `alphagen-flower` | `5555` | HTTP | Celery monitoring UI |
| [[alphaTrade\|alphaTrade]] | `alphatrade` | `8081` (API), `8080` (health), `9090` (metrics) | HTTP | Trading executor |
| [[alphaKey\|alphaKey]] | `alphakey-api` | `8000` | HTTP | Auth + account service |
| [[alphaLink\|alphaLink]] | `alphalink` | `3000` | HTTP | Next.js frontend |
| OTel Collector | `otel-collector` | `4317` (gRPC), `4318` (HTTP), `8888` (metrics), `13133` (health) | gRPC/HTTP | Telemetry aggregation |
| Prometheus | `prometheus` | `9090` | HTTP | Metrics storage |
| Loki | `loki` | `3100` | HTTP | Log aggregation |
| Tempo | `tempo` | `3200` | HTTP | Trace storage |
| Grafana | `grafana` | `3000` → host `3001` | HTTP | Unified observability UI |
| Alertmanager | `alertmanager` | `9093` | HTTP | Alert routing |
| Node Exporter | `node-exporter` | `9100` | HTTP | Host metrics |
| cAdvisor | `cadvisor` | `8080` | HTTP | Container metrics |
| Postgres Exporter | `postgres-exporter` | `9187` | HTTP | PostgreSQL metrics |
| Redis Exporter | `redis-exporter` | `9121` | HTTP | Redis metrics |

---

## Nginx Routes (Port 443)

[[alphaFrame|Nginx]] acts as the TLS-terminating reverse proxy.

| Path | Upstream | Notes |
|---|---|---|
| `/auth/internal/` | — | `return 404` — blocked, never proxied |
| `/auth/` | `alphakey-api:8000/auth/` | Rate-limited |
| `/alphagen/runs/<id>/(events\|log)` | `alphagen-api:8000` | SSE — no buffering, 3600s timeout, no rate limit |
| `/alphagen/runs/events` | `alphagen-api:8000/runs/events` | SSE — no buffering, 3600s timeout, no rate limit |
| `/alphagen/` | `alphagen-api:8000/` | Rate-limited |
| `/stream` | `alphatrade:8081/stream` | SSE — no buffering, 3600s timeout |
| `/` (default) | `alphalink:3000` | Rate-limited — Next.js UI |

---

## PostgreSQL Databases

Single PostgreSQL instance, separate databases per service:

| Database | Owner | Used by |
|---|---|---|
| `alphagen` | `platform` | [[alphaGen\|alphaGen]] — `runs`, `validation_settings` |
| `alphatrade` | `platform` | [[alphaTrade\|alphaTrade]] — all trading state |
| `mlflow` | `platform` | [[alphaFrame\|MLflow]] — experiment metadata |
| `alphakey` | `platform` | [[alphaKey\|alphaKey]] — users, tokens, vault |

---

## Redis Key Spaces

| DB | Used by | Purpose |
|---|---|---|
| `db 0` | alphaGen (Celery broker), alphaTrade (pub/sub cache), alphaKey (token denylist) | Celery task routing + pub/sub channels |
| `db 1` | alphaGen (Celery results) | Task state + return values |

See [[reference/Event-Channels]] for pub/sub channel names.

---

## MinIO Buckets

Per-service MinIO users are created by `minio-init` with policies scoped to their bucket only.

| Bucket | Service Account | Contents | Written by | Read by |
|---|---|---|---|---|
| `models` | `alphagen` | `model.onnx`, `manifest.json`, `backtest.json` | [[alphaGen\|alphaGen]] | [[alphaTrade\|alphaTrade]] |
| `trades` | `alphatrade` | Trade logs, reports | [[alphaTrade\|alphaTrade]] | Manual |
| `mlflow` | `mlflow` | MLflow artifact versions | MLflow (via alphaGen) | MLflow, [[alphaTrade\|alphaTrade]] |

---

*Update when: new service added, port changed, new Nginx route, new DB or bucket.*
