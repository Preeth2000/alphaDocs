---
service: alphaGen
page: Interactions
tags:
  - service/alphaGen
  - interactions
---

# alphaGen — Interactions

[[alphaGen|alphaGen]] · [[alphaDocs/services/alphaGen/Architecture|Architecture]] · [[alphaDocs/services/alphaGen/API|API]] · [[alphaDocs/services/alphaGen/Data|Data]] · [[alphaDocs/services/alphaGen/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger | Handled by |
|---|---|---|---|---|
| [[alphaLink\|alphaLink]] BFF | `POST /runs` | JSON `RunCreate {config_yaml, visibility}` | User submits training job | `api/routers/runs.py create_run()` |
| [[alphaLink\|alphaLink]] BFF | `GET /runs` | Query params: status, run_name, limit, offset | User views job list | `runs.py list_runs()` |
| [[alphaLink\|alphaLink]] BFF | `GET /runs/{id}` | Path param | User opens job detail | `runs.py get_run()` |
| [[alphaLink\|alphaLink]] BFF | `DELETE /runs/{id}` | Path param | User cancels job | `runs.py cancel_run()` |
| [[alphaLink\|alphaLink]] BFF | `POST /runs/{id}/force-save` | Path param | User bypasses gate | `runs.py force_save_run()` |
| [[alphaLink\|alphaLink]] BFF | `POST /runs/{id}/publish` | Path param | User publishes model | `runs.py publish_run()` |
| [[alphaLink\|alphaLink]] BFF | `GET /runs/{id}/log` | SSE long-poll | User views live logs | `runs.py stream_log()` |
| [[alphaLink\|alphaLink]] BFF | `GET/PATCH /config/validation` | JSON `ValidationSettingsIn` | Admin adjusts gate | `config.py` |
| [[alphaLink\|alphaLink]] BFF | `GET /models` | — | User browses published models | `models.py list_models()` |
| [[alphaLink\|alphaLink]] BFF | `GET /models/{run_name}` | Path param | User inspects model | `models.py get_model()` |
| [[alphaLink\|alphaLink]] BFF | `POST /models/{run_name}/promote` | JSON `{version?}` | Admin promotes to production | `models.py promote_model()` |
| [[alphaTrade\|alphaTrade]] | `GET /runs/events` (SSE) | — | On alphaTrade startup | `runs.py run_events()` → listens for model.ready |
| Celery (Redis db0) | Internal task | Pickled task args | After POST /runs | `worker.py train_and_export()` |
| yfinance / Polygon.io | HTTP | OHLCV JSON/CSV | Inside Celery task | `att.data.fetch` |
| [[alphaKey\|alphaKey]] JWKS | `GET /auth/.well-known/jwks.json` | JWK Set JSON | Every `/runs` request (JWKS cached 10 min) | `att.security.alphakey_auth.require_auth` |
| [[alphaKey\|alphaKey]] vault | `GET /auth/internal/secrets/{user_id}` | JSON secrets | When `SECRETS_SOURCE=alphakey` | `att.secrets.alphakey_client` |

---

## Outputs

| Destination | Mechanism | Format | Trigger | Produced by |
|---|---|---|---|---|
| [[alphaTrade\|alphaTrade]] | MinIO PUT `models/{user}/{account}/{run_name}/{version}/` | `model.onnx`, `manifest.json`, `backtest.json` | After successful training + publish | `att.publish.minio.publish()` |
| [[alphaTrade\|alphaTrade]] | Redis pub/sub `model.ready` | `{run_name, version, artifact_prefix, published_at}` | POST /runs/{id}/publish | `worker.py` + `runs.py` |
| [[alphaLink\|alphaLink]] | HTTP responses | JSON `RunSummary`, `RunOut`, `ModelInfo` | Per API call | All routers |
| [[alphaLink\|alphaLink]] | SSE stream `GET /runs/{id}/log` | `{type, line/status, ts}` JSON events | During training | Redis `run:{id}:log` → SSE |
| [[alphaLink\|alphaLink]] | SSE stream `GET /runs/events` | `{run_name, version, ...}` model.ready event | On publish | Redis `model.ready` → SSE |
| MLflow | SDK calls | Params, metrics, model artifact | After training success | `att.mlflow_utils.log_and_register_run()` |
| Redis `run:{id}:log` | Pub/sub PUBLISH | `{type:log, line, ts}` or `{type:status, status}` | Per log line during training | `att.api.worker.RedisLogHandler` |
| OTel Collector | OTLP gRPC | Traces + metrics | Always | `att.telemetry.setup_telemetry()` |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[alphaFrame\|alphaFrame]] → Postgres | `runs`, `validation_settings` tables | ✅ |
| [[alphaFrame\|alphaFrame]] → Redis | Celery broker (db0), results (db1), pub/sub | ✅ |
| [[alphaFrame\|alphaFrame]] → MinIO | `models` bucket for artifact storage | ✅ |
| [[alphaFrame\|alphaFrame]] → MLflow | Experiment tracking + model registry | ✅ |
| [[alphaKey\|alphaKey]] | JWT JWKS (token verification for all `/runs` routes) | ✅ |
| [[alphaKey\|alphaKey]] | Vault secrets (Polygon key, MinIO creds) | ⬜ (when SECRETS_SOURCE=alphakey) |
| yfinance (public API) | OHLCV historical data | ✅ (when provider=yfinance) |
| Polygon.io | OHLCV data (intraday) | ⬜ (when provider=polygon) |

---

## Downstream Consumers

| Service | What it consumes | Mechanism |
|---|---|---|
| [[alphaTrade\|alphaTrade]] | `model.ready` events | Redis pub/sub + SSE `/runs/events` |
| [[alphaTrade\|alphaTrade]] | model.onnx + manifest.json | MinIO S3 download |
| [[alphaLink\|alphaLink]] | All REST + SSE endpoints | HTTP via BFF |

---

## Error Outputs

| Condition | Output | Destination |
|---|---|---|
| Gate failure | `run.status=failed`, `error=gate:cat:decision:reason` | PostgreSQL + Redis `run:{id}:log` status event |
| Celery exception | `run.status=failed`, error stored | PostgreSQL |
| MLflow failure | Loud exception (no silent success) | Run marked failed |
| Training timeout (>30m) | Run auto-marked failed on worker restart | PostgreSQL |
