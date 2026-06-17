---
service: alphaTrade
page: API
tags:
  - service/alphaTrade
  - api
---

# alphaTrade — API

[[alphaTrade|alphaTrade]] · [[alphaDocs/services/alphaTrade/Architecture|Architecture]] · [[alphaDocs/services/alphaTrade/Interactions|Interactions]] · [[alphaDocs/services/alphaTrade/Data|Data]] · [[alphaDocs/services/alphaTrade/Config|Config]]

---

## Inbound Endpoints

**Base URL:** `http://alphatrade:8081/api/v1` (internal)  
**Auth:** Bearer token (`Authorization: Bearer <jwt>`) or `X-API-Key` header — enforced via `make_jwt_dep`. **Fail-closed by default**: if `alphaTrade_API_KEY` is unset and `AUTH_MODE` is `legacy`, all requests return 401 unless `ALPHATRADE_INSECURE_NO_AUTH=true` (local dev only). JWT mode verifies ES256 tokens via alphaKey JWKS; X-API-Key accepted as legacy fallback only when a key is configured.  
**Health probe (no auth):** `http://alphatrade:8080`

### Trading State

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/positions` | List open positions | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/orders` | Order history (24h default; `since`, `limit` params) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/signals` | Recent signals (24h default) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/pnl` | Daily P&L snapshots (`since` YYYY-MM-DD, `limit`) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/trades` | Closed trade journal (30d default; `since`, `limit`, `model_id`) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/equity-curve` | Equity timeseries (30d default, 500 max) | [[alphaLink\|alphaLink]] | 1 | 0 |

### Model Management

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/models` | All models: loaded + registry + MLflow + inactive | [[alphaLink\|alphaLink]] | 3 | 0 (+MLflow call) |
| `DELETE` | `/models/{run_name}` | Delete model from disk + MLflow + MinIO + DB | [[alphaLink\|alphaLink]] | 1 | 2 (+MLflow+S3 delete) |
| `DELETE` | `/models` | Bulk-delete models (body: `{"run_names": [...]}`) — ownership-checked per model; parallel via `asyncio.gather` | [[alphaLink\|alphaLink]] | 1 | N×2 |
| `GET` | `/models/overrides` | All per-model config overrides | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/models/{run_name}/overrides` | Single model overrides (or defaults if not set) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `PUT` | `/models/{run_name}/overrides` | Update model override (enabled, sizing, retirement, `consensus_min_confidence`, `consensus_min_margin`, etc.) | [[alphaLink\|alphaLink]] | 1 | 1 (UPSERT) |
| `DELETE` | `/models/{run_name}/overrides` | Delete overrides (revert to defaults) | [[alphaLink\|alphaLink]] | 1 | 1 |
| `GET` | `/models/deployments` | Latest deployment per model. `failure_msg` redacted (null) unless caller role is `developer`/`admin`; legacy/API-key mode (no role) fails closed | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/models/registry` | MLflow registered models (name, versions, aliases) | [[alphaLink\|alphaLink]] | 0 | 0 (MLflow call) |
| `POST` | `/models/{model_name}/promote` | Move MLflow version to production | [[alphaLink\|alphaLink]] | 0 | 1 (MLflow + INSERT deployment) |
| `POST` | `/models/{model_name}/demote` | Move production → staging | [[alphaLink\|alphaLink]] | 0 | 0 (MLflow call) |
| `POST` | `/models/{model_name}/retry-deploy` | Retry failed deployment | [[alphaLink\|alphaLink]] | 0 | 1 (DELETE sync + INSERT deployment) |
| `GET` | `/models/public` | Public library (visibility=public) | [[alphaLink\|alphaLink]] | 2 | 0 |
| `POST` | `/models/{model_name}/fork` | Fork public model to user namespace | [[alphaLink\|alphaLink]] | 0 | 2 (MLflow + UPSERT override) |

### Backtest

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `POST` | `/backtest/trigger` | Queue backtest run (async) | [[alphaLink\|alphaLink]] | 0 | 1 (INSERT BacktestRun) |
| `GET` | `/backtest/runs` | All completed backtests | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/backtest/runs/{run_id}` | Single backtest summary | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/backtest/runs/{run_id}/trades` | Trades from backtest | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/backtest/runs/{run_id}/models` | Per-model backtest summary | [[alphaLink\|alphaLink]] | 1 | 0 |
| `GET` | `/backtest/schedule` | Global backtest schedule config | [[alphaLink\|alphaLink]] | 0 | 0 (in-memory) |
| `PATCH` | `/backtest/schedule` | Update global cron + lookback | [[alphaLink\|alphaLink]] | 1 | 1 |
| `GET` | `/backtest/schedule/models` | All per-model backtest schedules | [[alphaLink\|alphaLink]] | 0 | 0 |
| `GET` | `/backtest/schedule/{model_id}` | Single model schedule | [[alphaLink\|alphaLink]] | 0 | 0 |
| `PATCH` | `/backtest/schedule/{model_id}` | Update model-specific schedule | [[alphaLink\|alphaLink]] | 0 | 1 |

### Kill Switch

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/kill-switch` | Halt status (HALT file check) | [[alphaLink\|alphaLink]] | 0 | 0 (filesystem) |
| `POST` | `/halt` | Engage kill switch (create HALT file) | [[alphaLink\|alphaLink]] | 0 | 0 (filesystem) |
| `POST` | `/resume` | Disengage kill switch (delete HALT file) | [[alphaLink\|alphaLink]] | 0 | 0 (filesystem) |

### Settings & Config

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/settings` | Bot config (sensitive fields masked) | [[alphaLink\|alphaLink]] | 1 | 0 |
| `PUT/PATCH` | `/settings` | Update bot config (hot-reload) | [[alphaLink\|alphaLink]] | 1 | 1 (UPSERT + reinit T212/provider) |
| `GET` | `/retirement/config` | Global retirement thresholds | [[alphaLink\|alphaLink]] | 0 | 0 (in-memory) |
| `PATCH` | `/retirement/config` | Update global retirement | [[alphaLink\|alphaLink]] | 1 | 1 |
| `GET` | `/models/{run_name}/retirement` | Per-model + effective retirement config | [[alphaLink\|alphaLink]] | 1 | 0 |
| `PATCH` | `/models/{run_name}/retirement` | Override retirement for model | [[alphaLink\|alphaLink]] | 1 | 1 |
| `DELETE` | `/models/{run_name}/retirement` | Remove per-model retirement overrides | [[alphaLink\|alphaLink]] | 1 | 1 |
| `POST` | `/models/{run_name}/unretire` | Reset model retirement state | [[alphaLink\|alphaLink]] | 1 | 1 |

### Credential Verification

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `POST` | `/verify/t212` | Test T212 credentials | [[alphaLink\|alphaLink]] | 0 | 0 (outbound HTTP) |
| `POST` | `/verify/polygon` | Test Polygon API key | [[alphaLink\|alphaLink]] | 0 | 0 (outbound HTTP) |
| `POST` | `/verify/alphatrade-key` | Test alphaTrade API key | [[alphaLink\|alphaLink]] | 0 | 0 |
| `POST` | `/verify/provider` | Test active data provider | [[alphaLink\|alphaLink]] | 0 | 0 (outbound HTTP) |

### Health

| Method | Path | Port | Purpose | Auth |
|---|---|---|---|---|
| `GET` | `/health` | 8081 | Health state (models, T212, last tick) | API key |
| `GET` | `/healthz` | 8080 | Container liveness probe | None |
| `GET` | `/readyz` | 8080 | Container readiness probe | None |

### SSE Stream

| Method | Path | Purpose | Caller | Auth |
|---|---|---|---|---|
| `GET` | `/stream` | Live tick events, fills, halts | [[alphaLink\|alphaLink]] | API key (query param `?key=`) |

---

## Outbound Calls

| Target | Method | Endpoint | Trigger | Auth |
|---|---|---|---|---|
| Trading 212 (demo) | REST | `https://demo.trading212.com/api/v0/*` | Per tick + OCO poll | HTTP Basic or Bearer |
| Trading 212 (live) | REST | `https://live.trading212.com/api/v0/*` | Per tick + OCO poll | HTTP Basic or Bearer |
| yfinance | HTTP | Yahoo Finance | Per tick (default provider) | None |
| Polygon.io | HTTP | `https://api.polygon.io/v1/aggs/...` | Per tick (if provider=polygon) | Bearer |
| MinIO / S3 | S3 API | `models` bucket | ModelSyncDaemon poll | AWS credentials |
| [[alphaFrame\|MLflow]] | SDK | `http://mlflow:5000/api/2.0/...` | Model management endpoints | None |
| [[alphaKey\|alphaKey]] | `POST` | `/auth/introspect` | Per authenticated request | `X-Service-Token` |
| Slack | HTTP POST | Webhook URL | Alert events | Webhook URL |
| SMTP server | SMTP/TLS | `smtp_host:smtp_port` | Alert events | SMTP credentials |
| Custom webhook | HTTP POST | `WEBHOOK_URL` | Alert events | None |
| OTel Collector | OTLP gRPC | `:4317` | Always | None |
