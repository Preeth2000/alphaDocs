---
service: alphaGen
page: Config
tags:
  - service/alphaGen
  - config
---

# alphaGen — Config

[[services/alphaGen/alphaGen|alphaGen]] · [[services/alphaGen/Architecture|Architecture]] · [[services/alphaGen/Interactions|Interactions]] · [[services/alphaGen/API|API]] · [[services/alphaGen/Data|Data]]

---

## Environment Variables

Source: `alphaGen/.env.example`

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `DATABASE_URL` | — | ✅ | Postgres connection for `runs` + `validation_settings`. **Required in production** — unset outside `ATT_ENV=test/dev` raises `RuntimeError` at startup |
| `ATT_ENV` | — | ⬜ | Set to `test` or `dev` to allow in-memory SQLite fallback (local dev only). Unset in production |
| `REDIS_URL` | `redis://:password@redis:6379/0` | ✅ | Pub/sub + Celery broker (password required — see `REDIS_PASSWORD` in alphaFrame) |
| `CELERY_BROKER_URL` | `redis://:password@redis:6379/0` | ✅ | Celery task broker |
| `CELERY_RESULT_BACKEND` | `redis://:password@redis:6379/1` | ✅ | Celery task results |
| `MINIO_ENDPOINT` | `http://minio:9000` | ✅ | MinIO S3 endpoint for artifact storage |
| `MINIO_ACCESS_KEY` | `minioadmin` | ✅ | MinIO credentials |
| `MINIO_SECRET_KEY` | `minioadmin` | ✅ | MinIO credentials |
| `MINIO_BUCKET` | `models` | ✅ | Bucket for published model artifacts |
| `MINIO_USER` | `prod` | ⬜ | Namespace prefix (user level) in MinIO paths |
| `MINIO_ACCOUNT` | `isa` | ⬜ | Namespace prefix (account level) in MinIO paths |
| `MLFLOW_TRACKING_URI` | `http://mlflow:5000` | ✅ | MLflow experiment tracking + registry |
| `MLFLOW_S3_ENDPOINT_URL` | `http://minio:9000` | ✅ | S3 endpoint used by MLflow SDK |
| `POLYGON_API_KEY` | — | ⬜ | Polygon.io data provider API key (if provider=polygon) |
| `OTEL_SERVICE_NAME` | `alphagen-api` / `alphagen-worker` | ⬜ | OTel service name for traces/metrics |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | ⬜ | OTel Collector OTLP endpoint |
| `ALPHAKEY_URL` | `http://alphakey-api:8000` | ✅ | alphaKey service URL — JWKS fetched for JWT verification on all `/runs` routes |
| `ALPHAKEY_SERVICE_TOKEN` | — | ⬜ | Service-to-service token for alphaKey vault API |
| `JWT_ISSUER` | `alphakey` | ⬜ | Expected `iss` claim in Bearer JWTs; tokens with a different issuer are rejected |
| `JWT_AUDIENCE` | `alphakey` | ⬜ | Expected `aud` claim in Bearer JWTs; tokens with a different audience are rejected |
| `MINIO_ACCESS_KEY` | — | ✅ | Scoped MinIO service account for alphaGen (models bucket only); injected by alphaFrame |
| `MINIO_SECRET_KEY` | — | ✅ | Secret for the scoped MinIO service account above |
| `SECRETS_SOURCE` | `env` | ⬜ | `env` (vars) or `alphakey` (vault-fetched) |
| `RETRAIN_MAX_ATTEMPTS` | `3` | ⬜ | Max automatic retrains before escalating to `flag_human` |
| `RETRAIN_RETRY_DELAY_SECONDS` | `300` | ⬜ | Countdown (seconds) before the rescheduled Celery retrain task is dispatched |
| `DRIFT_ENABLED` | `true` | ⬜ | Enable/disable drift monitoring Beat task |
| `DRIFT_PSI_THRESHOLD` | `0.20` | ⬜ | PSI score above which a retrain is triggered (PSI > 0.20 = significant drift) |
| `DRIFT_KS_THRESHOLD` | `0.10` | ⬜ | KS statistic above which a retrain is triggered |
| `DRIFT_CHECK_INTERVAL_SECONDS` | `86400` | ⬜ | Celery Beat drift check period (default: daily) |
| `DRIFT_LOOKBACK_BARS` | `252` | ⬜ | Number of recent live bars compared against reference distribution |
| `DRIFT_MIN_LIVE_BARS` | `50` | ⬜ | Minimum live bars required to run drift check (skip otherwise) |

---

## RunConfig YAML Schema

Every training run is configured by a YAML string stored in `run.config_yaml`. Structure:

```yaml
run_name: aapl_daily_mlp        # Unique model identifier (keys MinIO paths + overrides)

data:
  ticker: AAPL                  # Ticker symbol (yfinance format)
  interval: 1d                  # 1m | 5m | 15m | 1h | 1d | 1wk
  start: "2018-01-01"           # Training start date
  end: "2024-12-31"             # Training end date
  cache: true                   # Cache fetched OHLCV to parquet
  provider: yfinance            # yfinance | polygon
  include_vwap: false
  multi: []                     # Multi-ticker alignment (list of extra tickers)

features:
  indicators:
    - name: RSI
      args: {timeperiod: 14}
    - name: MACD
      args: {fastperiod: 12, slowperiod: 26, signalperiod: 9}
    - name: BBANDS
    - name: ATR
    - name: EMA
    - name: SMA
    - name: ADX
    - name: STOCH
    - name: OBV
  include_ohlcv: true
  normalize: zscore             # zscore | minmax | none
  window: 30                    # Lookback window size (bars)
  fundamentals: false

label:
  strategy: forward_return      # forward_return | triple_barrier
  horizon_bars: 5
  params:
    threshold: 0.01

model:
  arch: mlp                     # mlp | lstm | cnn | transformer
  params:
    hidden_dims: [128, 64]
    dropout: 0.3

train:
  backend: torch
  device: auto                  # auto | cpu | cuda
  epochs: 50
  batch_size: 64
  lr: 0.001
  optimizer: adam
  loss: cross_entropy
  class_weights: balanced
  early_stop_patience: 10
  splits:
    type: walk_forward
    n_folds: 5
    embargo_bars: 10   # Must be >= label.horizon_bars (validated at load time)

export:
  opset: 17
  dynamic_batch: true
  verify_parity: true

backtest:
  fee_bps: 10
  slippage_bps: 5
  position_sizing: fixed
  fraction: 0.1

sweep:                          # Optional hyperparameter sweep
  enabled: false
  trials: 50
  seed: 42                      # TPE sampler seed for reproducibility
  search_space: {}
```

---

## Validation Gate Config

Stored in `validation_settings` DB table (id=1). Accessible via `GET/PATCH /config/validation`.

| Setting | Default | Effect when breached |
|---|---|---|
| `enabled` | `true` | If false, gate always passes |
| `min_sharpe` | `0.5` | Run marked `gate_failed` |
| `max_drawdown` | `0.20` | Run marked `gate_failed` |
| `min_hit_rate` | `0.45` | Run marked `gate_failed` |
| `min_n_trades` | `5` | Run marked `gate_failed` |
| `max_val_loss` | `null` | Optional — checked if set |
| `min_val_accuracy` | `null` | Optional — checked if set |
| `min_f1_class1` | `null` | Optional — checked if set |

Gate failure decisions:

| Decision | Trigger | Outcome |
|---|---|---|
| `retrain` | `metric`/`training` category + budget not exhausted | Run reset to `queued`; new Celery task dispatched after `RETRAIN_RETRY_DELAY_SECONDS` |
| `flag_human` | `structural`/`unknown` category, or retrain budget exhausted (`RETRAIN_MAX_ATTEMPTS`) | Run set to `failed`; error = `gate:{category}:flag_human: {reason}` |
| `retry_later` | `infra` category (transient failure) | Run set to `failed`; error = `gate:infra:retry_later: {reason}` |

---

## Config Validation Constraints

| Constraint | Enforced by | Error |
|---|---|---|
| `train.splits.embargo_bars >= label.horizon_bars` | `RunConfig` `model_validator` at load | `ValueError` — labels look forward `horizon_bars` bars; embargo must cover this gap to prevent train labels near the fold boundary from reading prices inside the val window |
| `data.interval` within yfinance history limit | `RunConfig` `model_validator` at load | `UserWarning` (not hard error) — intraday intervals have yfinance history caps |

---

## No Override Priority Chain

alphaGen has no runtime config overrides. Config is immutable per-run (stored in `run.config_yaml`). Validation gate thresholds are the only mutable config — via DB (PATCH /config/validation).

> [!note] Compare to alphaTrade
> [[services/alphaTrade/alphaTrade|alphaTrade]] has a full YAML → DB override priority chain. alphaGen does not — each run captures its full config at submission time.
