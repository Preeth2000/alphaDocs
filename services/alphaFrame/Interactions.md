---
service: alphaFrame
page: Interactions
tags:
  - service/alphaFrame
  - interactions
---

# alphaFrame — Interactions

[[services/alphaFrame/alphaFrame|alphaFrame]] · [[services/alphaFrame/Architecture|Architecture]] · [[services/alphaFrame/API|API]] · [[services/alphaFrame/Data|Data]] · [[services/alphaFrame/Config|Config]]

---

## Inputs

alphaFrame is **pure infrastructure** — it receives no application-layer requests. All inputs are Docker-level.

| Source | Mechanism | Purpose |
|---|---|---|
| `docker compose up` operator | Docker Compose CLI | Start entire stack |
| `scripts/reset-dev.sh` | Bash → Docker exec | Truncate all data (schema preserved) |
| `scripts/clear-models.sh` | Bash → Docker exec | Clear model artifacts + deployment records |
| App services (alphaKey, alphaGen, alphaTrade) | TCP connections | Connect to Postgres, Redis, MinIO, MLflow |
| App services | OTLP gRPC :4317 | Send traces, metrics, logs to OTel Collector |
| Prometheus | HTTP pull | Scrape metrics from exporters + app `/metrics` endpoints |
| Browser / alphaLink | HTTP :80/:443 | Nginx proxy for all app services |

---

## Outputs

| Destination | Mechanism | Format | What |
|---|---|---|---|
| [[services/alphaKey/alphaKey\|alphaKey]] | TCP :5432 | SQL | `alphakey` database queries |
| [[services/alphaGen/alphaGen\|alphaGen]] API + Worker | TCP :5432 | SQL | `alphagen` database queries |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | TCP :5432 | SQL | `alphatrade` database queries |
| MLflow server | TCP :5432 | SQL | `mlflow` database queries |
| All app services | TCP :6379 | Redis protocol | Pub/sub, Celery broker, token denylist |
| [[services/alphaGen/alphaGen\|alphaGen]], [[services/alphaTrade/alphaTrade\|alphaTrade]] | HTTP :9000 | S3 API | Model artifacts (models bucket), trade logs (trades bucket), MLflow artifacts (mlflow bucket) |
| All app services | HTTP :5000 | REST | MLflow experiment tracking + model registry API |
| [[services/alphaLink/alphaLink\|alphaLink]] | HTTP/HTTPS | Proxied REST + SSE | Nginx forwards to app services |
| Grafana | HTTP | Dashboard queries | Prometheus metrics, Loki logs, Tempo traces |
| Alertmanager | HTTP | Alert delivery | Fires webhook to `localhost:9999` (stub) |

---

## Service Dependency Map

```mermaid
flowchart TD
    PG[(postgres)] --> PGE[postgres-exporter]
    RD[(redis)] --> RDE[redis-exporter]
    MN[(minio)] --> MI[minio-init] --> ML[mlflow]
    MI --> AK[alphakey-api]
    MI --> AG_A[alphagen-api]
    MI --> AG_W[alphagen-worker]
    MI --> AT[alphatrade]
    PG --> AK & AG_A & AG_W & AT & ML
    RD --> AK & AG_A & AG_W & AT
    ML --> AG_A & AT
    OC[otel-collector] --> AK & AG_A & AG_W & AT
    PGE & RDE & AG_F[alphagen-flower] --> PR[prometheus]
    PR --> AM[alertmanager]
    PR & LK[loki] & TM[tempo] --> GR[grafana]
    OC --> PR & LK & TM
```

---

## Data Reset — What Gets Cleared

**`scripts/reset-dev.sh`** (requires typing "RESET" then "YES"):
- TRUNCATE CASCADE: all user tables in `alphagen` and `alphatrade` DBs (preserves `alembic_version`)
- DELETE: all MLflow runs, experiments (keeps experiment id=0), model versions
- `mc rm --recursive`: all objects in `models`, `trades`, `mlflow` MinIO buckets
- `redis-cli FLUSHALL`
- Clear alphaTrade local model cache (`/app/models/*`)
- Does NOT touch: bucket structure, DB schemas, migration state

**`scripts/clear-models.sh`**:
- MinIO `models` + `mlflow` buckets cleared
- MLflow DB: runs, registered models, model versions, traces
- alphaTrade DB: backtesttrade, backtestmodelrun, backtestrun, model_deployments, model_override, modelperformance, signal
- Local alphaTrade model cache

---

## No Application Outputs

alphaFrame does not produce business data — it stores, forwards, and observes data created by application services. No events, no REST responses to upstream callers.
