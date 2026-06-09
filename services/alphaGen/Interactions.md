---
service: alphaGen
page: Interactions
tags:
  - service/alphaGen
  - interactions
---

# alphaGen — Interactions

[[services/alphaGen/alphaGen|alphaGen]] · [[services/alphaGen/Architecture|Architecture]] · [[services/alphaGen/API|API]] · [[services/alphaGen/Data|Data]] · [[services/alphaGen/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger | Handled by |
|---|---|---|---|---|
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /runs` | JSON `RunCreate {config_yaml, visibility}` | User submits training job | `api/routers/runs.py create_run()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /runs` | Query params: status, run_name, limit, offset | User views job list | `runs.py list_runs()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /runs/{id}` | Path param | User opens job detail | `runs.py get_run()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `DELETE /runs/{id}` | Path param | User cancels job | `runs.py cancel_run()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /runs/{id}/force-save` | Path param | User bypasses gate | `runs.py force_save_run()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /runs/{id}/publish` | Path param | User publishes model | `runs.py publish_run()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /runs/{id}/log` | SSE long-poll | User views live logs | `runs.py stream_log()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET/PATCH /config/validation` | JSON `ValidationSettingsIn` | Admin adjusts gate | `config.py` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /models` | — | User browses published models | `models.py list_models()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /models/{run_name}` | Path param | User inspects model | `models.py get_model()` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /models/{run_name}/promote` | JSON `{version?}` | Admin promotes to production | `models.py promote_model()` |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | `GET /runs/events` (SSE) | — | On alphaTrade startup | `runs.py run_events()` → listens for model.ready |
| Celery (Redis db0) | Internal task | Pickled task args | After POST /runs | `worker.py train_and_export()` |
| yfinance / Polygon.io | HTTP | OHLCV JSON/CSV | Inside Celery task | `att.data.fetch` |
| [[services/alphaKey/alphaKey\|alphaKey]] JWKS | `GET /auth/.well-known/jwks.json` | JWK Set JSON | On token verification | `att.security.alphakey_auth` |
| [[services/alphaKey/alphaKey\|alphaKey]] vault | `GET /auth/internal/secrets/{user_id}` | JSON secrets | When `SECRETS_SOURCE=alphakey` | `att.secrets.alphakey_client` |

---

## Outputs

| Destination | Mechanism | Format | Trigger | Produced by |
|---|---|---|---|---|
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | MinIO PUT `models/{user}/{account}/{run_name}/{version}/` | `model.onnx`, `manifest.json`, `backtest.json` | After successful training + publish | `att.publish.minio.publish()` |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | Redis pub/sub `model.ready` | `{run_name, version, artifact_prefix, published_at}` | POST /runs/{id}/publish | `worker.py` + `runs.py` |
| [[services/alphaLink/alphaLink\|alphaLink]] | HTTP responses | JSON `RunSummary`, `RunOut`, `ModelInfo` | Per API call | All routers |
| [[services/alphaLink/alphaLink\|alphaLink]] | SSE stream `GET /runs/{id}/log` | `{type, line/status, ts}` JSON events | During training | Redis `run:{id}:log` → SSE |
| [[services/alphaLink/alphaLink\|alphaLink]] | SSE stream `GET /runs/events` | `{run_name, version, ...}` model.ready event | On publish | Redis `model.ready` → SSE |
| MLflow | SDK calls | Params, metrics, model artifact | After training success | `att.mlflow_utils.log_and_register_run()` |
| Redis `run:{id}:log` | Pub/sub PUBLISH | `{type:log, line, ts}` or `{type:status, status}` | Per log line during training | `att.api.worker.RedisLogHandler` |
| OTel Collector | OTLP gRPC | Traces + metrics | Always | `att.telemetry.setup_telemetry()` |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Postgres | `runs`, `validation_settings` tables | ✅ |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Redis | Celery broker (db0), results (db1), pub/sub | ✅ |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → MinIO | `models` bucket for artifact storage | ✅ |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → MLflow | Experiment tracking + model registry | ✅ |
| [[services/alphaKey/alphaKey\|alphaKey]] | JWT JWKS (token verification) | ⬜ (when AUTH_MODE=alphakey) |
| [[services/alphaKey/alphaKey\|alphaKey]] | Vault secrets (Polygon key, MinIO creds) | ⬜ (when SECRETS_SOURCE=alphakey) |
| yfinance (public API) | OHLCV historical data | ✅ (when provider=yfinance) |
| Polygon.io | OHLCV data (intraday) | ⬜ (when provider=polygon) |

---

## Downstream Consumers

| Service | What it consumes | Mechanism |
|---|---|---|
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | `model.ready` events | Redis pub/sub + SSE `/runs/events` |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | model.onnx + manifest.json | MinIO S3 download |
| [[services/alphaLink/alphaLink\|alphaLink]] | All REST + SSE endpoints | HTTP via BFF |

---

## Error Outputs

| Condition | Output | Destination |
|---|---|---|
| Gate failure | `run.status=failed`, `error=gate:cat:decision:reason` | PostgreSQL + Redis `run:{id}:log` status event |
| Celery exception | `run.status=failed`, error stored | PostgreSQL |
| MLflow failure | Loud exception (no silent success) | Run marked failed |
| Training timeout (>30m) | Run auto-marked failed on worker restart | PostgreSQL |
