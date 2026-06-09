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
| `80` | [[services/alphaFrame/alphaFrame\|Nginx]] | HTTP | Public — redirects to 443 |
| `443` | [[services/alphaFrame/alphaFrame\|Nginx]] | HTTPS (self-signed) | Public — reverse proxy |
| `3000` | [[services/alphaLink/alphaLink\|alphaLink]] | HTTP | Public — Next.js frontend |
| `3001` | Grafana | HTTP | Internal — observability UI |
| `5000` | MLflow | HTTP | Internal — experiment tracking + registry UI |
| `5432` | PostgreSQL | TCP | Internal — SQL database |
| `6379` | Redis | TCP | Internal — Celery broker / pub/sub |
| `9000` | MinIO | HTTP | Internal — S3-compatible API |
| `9001` | MinIO | HTTP | Internal — MinIO web console |
| `9090` | Prometheus | HTTP | Internal — metrics query UI + remote write |
| `9093` | Alertmanager | HTTP | Internal — alert routing UI |
| `3100` | Loki | HTTP | Internal — log API |
| `3200` | Tempo | HTTP | Internal — trace API |

---

## Internal Services (Platform Network)

All services communicate via Docker `platform` bridge network using service names as hostnames.

| Service | Internal Host | Port(s) | Protocol | Purpose |
|---|---|---|---|---|
| [[services/alphaFrame/alphaFrame\|postgres]] | `postgres` | `5432` | TCP/SQL | Primary DB — 4 databases |
| [[services/alphaFrame/alphaFrame\|redis]] | `redis` | `6379` | TCP | Celery broker (db0), results (db1), pub/sub |
| [[services/alphaFrame/alphaFrame\|minio]] | `minio` | `9000` | HTTP/S3 | Object storage |
| [[services/alphaFrame/alphaFrame\|MLflow]] | `mlflow` | `5000` | HTTP | Experiment tracking + model registry |
| [[services/alphaGen/alphaGen\|alphaGen API]] | `alphagen-api` | `8000` | HTTP | Training jobs REST + SSE |
| [[services/alphaGen/alphaGen\|alphaGen Worker]] | `alphagen-worker` | — | (Celery) | Training job executor |
| [[services/alphaGen/alphaGen\|Flower]] | `alphagen-flower` | `5555` | HTTP | Celery monitoring UI |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | `alphatrade` | `8081` (API), `8080` (health), `9090` (metrics) | HTTP | Trading executor |
| [[services/alphaKey/alphaKey\|alphaKey]] | `alphakey-api` | `8000` | HTTP | Auth + account service |
| [[services/alphaLink/alphaLink\|alphaLink]] | `alphalink` | `3000` | HTTP | Next.js frontend |
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

[[services/alphaFrame/alphaFrame|Nginx]] acts as the TLS-terminating reverse proxy.

| Path | Upstream | Notes |
|---|---|---|
| `/auth/` | `alphakey-api:8000/auth/` | Rate-limited. `/auth/internal/*` NOT exposed |
| `/alphagen/` | `alphagen-api:8000/` | Rate-limited |
| `/alphagen/runs/events` | `alphagen-api:8000/runs/events` | SSE — no buffering, 3600s timeout |
| `/stream` | `alphatrade:8081/stream` | SSE — no buffering, 3600s timeout |
| `/` (default) | `alphatrade:8081` | Rate-limited |

---

## PostgreSQL Databases

Single PostgreSQL instance, separate databases per service:

| Database | Owner | Used by |
|---|---|---|
| `alphagen` | `platform` | [[services/alphaGen/alphaGen\|alphaGen]] — `runs`, `validation_settings` |
| `alphatrade` | `platform` | [[services/alphaTrade/alphaTrade\|alphaTrade]] — all trading state |
| `mlflow` | `platform` | [[services/alphaFrame/alphaFrame\|MLflow]] — experiment metadata |
| `alphakey` | `platform` | [[services/alphaKey/alphaKey\|alphaKey]] — users, tokens, vault |

---

## Redis Key Spaces

| DB | Used by | Purpose |
|---|---|---|
| `db 0` | alphaGen (Celery broker), alphaTrade (pub/sub cache), alphaKey (token denylist) | Celery task routing + pub/sub channels |
| `db 1` | alphaGen (Celery results) | Task state + return values |

See [[reference/Event-Channels]] for pub/sub channel names.

---

## MinIO Buckets

| Bucket | Contents | Written by | Read by |
|---|---|---|---|
| `models` | `model.onnx`, `manifest.json`, `backtest.json` | [[services/alphaGen/alphaGen\|alphaGen]] | [[services/alphaTrade/alphaTrade\|alphaTrade]] |
| `trades` | Trade logs, reports | [[services/alphaTrade/alphaTrade\|alphaTrade]] | Manual |
| `mlflow` | MLflow artifact versions | MLflow (via alphaGen) | MLflow, [[services/alphaTrade/alphaTrade\|alphaTrade]] |

---

*Update when: new service added, port changed, new Nginx route, new DB or bucket.*
