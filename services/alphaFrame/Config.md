---
service: alphaFrame
page: Config
tags:
  - service/alphaFrame
  - config
---

# alphaFrame — Config

[[services/alphaFrame/alphaFrame|alphaFrame]] · [[services/alphaFrame/Architecture|Architecture]] · [[services/alphaFrame/Interactions|Interactions]] · [[services/alphaFrame/API|API]] · [[services/alphaFrame/Data|Data]]

---

## Environment Variables

Source: `alphaFrame/.env.example` → copy to `.env`

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `POSTGRES_PASSWORD` | `changeme` | ✅ | Shared PostgreSQL password for `platform` user |
| `REDIS_PASSWORD` | `changeme` | ✅ | Redis `requirepass` — propagated to all service Redis URLs |
| `MINIO_ROOT_USER` | `minioadmin` | ✅ | MinIO admin credentials (not used by services directly) |
| `MINIO_ROOT_PASSWORD` | `changeme` | ✅ | MinIO admin credentials (not used by services directly) |
| `MINIO_ALPHAGEN_USER` | `alphagen` | ✅ | MinIO service account for alphaGen (models bucket only) |
| `MINIO_ALPHAGEN_PASSWORD` | `changeme` | ✅ | MinIO service account password for alphaGen |
| `MINIO_ALPHATRADE_USER` | `alphatrade` | ✅ | MinIO service account for alphaTrade (trades bucket only) |
| `MINIO_ALPHATRADE_PASSWORD` | `changeme` | ✅ | MinIO service account password for alphaTrade |
| `MINIO_MLFLOW_USER` | `mlflow` | ✅ | MinIO service account for MLflow (mlflow bucket only) |
| `MINIO_MLFLOW_PASSWORD` | `changeme` | ✅ | MinIO service account password for MLflow |
| `NGINX_RATE_LIMIT` | `120r/m` | ⬜ | Nginx requests-per-minute rate limit |
| `NGINX_BURST` | `60` | ⬜ | Nginx burst allowance |
| `POLYGON_API_KEY` | — | ⬜ | Passed to alphaGen + alphaTrade (data provider) |
| `GRAFANA_ADMIN_USER` | `admin` | ⬜ | Grafana admin login |
| `GRAFANA_ADMIN_PASSWORD` | `changeme` | ✅ | Grafana admin password |
| `SLACK_WEBHOOK_URL` | — | ⬜ | Alertmanager Slack webhook for `severity=critical` alerts |
| `PAGERDUTY_ROUTING_KEY` | — | ⬜ | Alertmanager PagerDuty routing key for `severity=critical` alerts |
| `VAULT_MASTER_KEY` | — | ✅ | 32-byte base64 KEK for alphaKey credential vault |
| `ALPHAKEY_SERVICE_TOKENS` | — | ✅ | `token:principal` pairs for service-to-service auth |
| `ALPHAKEY_SERVICE_TOKEN_TRADE` | — | ✅ | alphaTrade's service token (subset of SERVICE_TOKENS) |
| `ALPHAKEY_SERVICE_TOKEN_GEN` | — | ✅ | alphaGen's service token (subset of SERVICE_TOKENS) |
| `ACCESS_TOKEN_TTL` | `600` | ⬜ | JWT access token lifetime (seconds) |
| `REFRESH_TOKEN_TTL` | `1209600` | ⬜ | Refresh token lifetime (seconds; 14 days) |
| `ALPHAKEY_USER_ID` | — | ⬜ | Default user UUID for vault namespace |
| `SECRETS_SOURCE` | `db` | ⬜ | `db` (BotSettings DB columns) or `alphakey` (vault) |
| `AUTH_MODE` | `legacy` | ⬜ | `legacy` (no JWT) or `alphakey` (JWT required) |

---

## Per-Service Env Wiring (docker-compose.yml)

These vars are wired from `.env` into each container automatically.

### postgres
```yaml
POSTGRES_USER: platform
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
POSTGRES_DB: platform
```

### minio
```yaml
MINIO_ROOT_USER: ${MINIO_ROOT_USER}
MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
```

### minio-init
Creates per-service MinIO users and scoped bucket policies on first run:
- `alphagen` user → read/write `models` bucket only
- `alphatrade` user → read/write `trades` bucket only
- `mlflow` user → read/write `mlflow` bucket only

### mlflow
```yaml
MLFLOW_S3_ENDPOINT_URL: http://minio:9000
AWS_ACCESS_KEY_ID: ${MINIO_MLFLOW_USER}
AWS_SECRET_ACCESS_KEY: ${MINIO_MLFLOW_PASSWORD}
```
Command: `mlflow server --backend-store-uri postgresql://platform:${PG_PW}@postgres:5432/mlflow --default-artifact-root s3://mlflow --no-serve-artifacts`

### alphakey-api
```yaml
DATABASE_URL: postgresql://platform:${POSTGRES_PASSWORD}@postgres:5432/alphakey
REDIS__URL: redis://redis:6379/0
VAULT_MASTER_KEY: ${VAULT_MASTER_KEY}
ALPHAKEY_SERVICE_TOKENS: ${ALPHAKEY_SERVICE_TOKENS}
ACCESS_TOKEN_TTL: ${ACCESS_TOKEN_TTL}
REFRESH_TOKEN_TTL: ${REFRESH_TOKEN_TTL}
OTEL_SERVICE_NAME: alphakey
OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL: grpc
```

### alphagen-api + alphagen-worker
```yaml
DATABASE_URL: postgresql://platform:${POSTGRES_PASSWORD}@postgres:5432/alphagen
REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
CELERY_RESULT_BACKEND: redis://:${REDIS_PASSWORD}@redis:6379/1
MLFLOW_TRACKING_URI: http://mlflow:5000
MLFLOW_S3_ENDPOINT_URL: http://minio:9000
MINIO_ENDPOINT: http://minio:9000
MINIO_ACCESS_KEY: ${MINIO_ALPHAGEN_USER}
MINIO_SECRET_KEY: ${MINIO_ALPHAGEN_PASSWORD}
MINIO_BUCKET: models
POLYGON_API_KEY: ${POLYGON_API_KEY}
ALPHAKEY_URL: http://alphakey-api:8000
ALPHAKEY_SERVICE_TOKEN: ${ALPHAKEY_SERVICE_TOKEN_GEN}
ALPHAKEY_USER_ID: ${ALPHAKEY_USER_ID}
SECRETS_SOURCE: ${SECRETS_SOURCE}
OTEL_SERVICE_NAME: alphagen-api | alphagen-worker
OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
```

### alphatrade
```yaml
DATABASE_URL: postgresql://platform:${POSTGRES_PASSWORD}@postgres:5432/alphatrade
REDIS__URL: redis://:${REDIS_PASSWORD}@redis:6379/0
ALPHAKEY_URL: http://alphakey-api:8000
ALPHAKEY_SERVICE_TOKEN: ${ALPHAKEY_SERVICE_TOKEN_TRADE}
ALPHAKEY_USER_ID: ${ALPHAKEY_USER_ID}
SECRETS_SOURCE: ${SECRETS_SOURCE}
MLFLOW_TRACKING_URI: http://mlflow:5000
MLFLOW_S3_ENDPOINT_URL: http://minio:9000
AWS_ACCESS_KEY_ID: ${MINIO_ALPHATRADE_USER}
AWS_SECRET_ACCESS_KEY: ${MINIO_ALPHATRADE_PASSWORD}
OTEL_SERVICE_NAME: alphatrade
OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
```

### alphalink
```yaml
ALPHAGEN_API_URL: http://alphagen-api:8000
ALPHATRADE_API_URL: http://alphatrade:8081/api/v1
ALPHATRADE_HEALTH_URL: http://alphatrade:8080
ALPHAKEY_API_URL: http://alphakey-api:8000
```

### nginx
```yaml
NGINX_RATE_LIMIT: ${NGINX_RATE_LIMIT}
NGINX_BURST: ${NGINX_BURST}
```

### grafana
```yaml
GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
GF_AUTH_ANONYMOUS_ENABLED: false
GF_FEATURE_TOGGLES_ENABLE: traceqlEditor
```

---

## Observability Config Files

| File | Purpose |
|---|---|
| `observability/otel-collector/config.yaml` | 3 pipelines: traces→Tempo, metrics→Prometheus, logs→Loki |
| `observability/prometheus/prometheus.yml` | Scrape targets, 15d retention, remote-write receiver |
| `observability/prometheus/rules/alerts.yml` | 7 alert rules |
| `observability/loki/config.yaml` | 7d retention, TSDB v13, volume enabled |
| `observability/tempo/config.yaml` | 7d retention, service-graphs + span-metrics processors |
| `observability/alertmanager/config.yaml` | Null default receiver; critical route (30s group_wait) → Slack + PagerDuty via env vars; inhibit rules |
| `observability/grafana/provisioning/datasources/datasources.yaml` | Prometheus+Loki+Tempo with exemplar/trace links |
| `observability/grafana/provisioning/dashboards/dashboards.yaml` | File provider for dashboard JSON |
| `observability/grafana/dashboards/services.json` | Platform services dashboard |
| `observability/grafana/dashboards/celery.json` | Celery worker dashboard |

---

## No Override Chain

alphaFrame has no application-layer config overrides — all configuration is static Docker Compose env vars. Changes require `docker compose up -d --build` to take effect.
