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
| `DATABASE_URL` | `postgresql://att:att@postgres:5432/att` | ✅ | Postgres connection for `runs` + `validation_settings` |
| `REDIS_URL` | `redis://redis:6379/0` | ✅ | Pub/sub + Celery broker |
| `CELERY_BROKER_URL` | `redis://redis:6379/0` | ✅ | Celery task broker |
| `CELERY_RESULT_BACKEND` | `redis://redis:6379/1` | ✅ | Celery task results |
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
| `ALPHAKEY_URL` | `http://alphakey-api:8000` | ⬜ | alphaKey service URL (when AUTH_MODE=alphakey) |
| `ALPHAKEY_SERVICE_TOKEN` | — | ⬜ | Service-to-service token for alphaKey vault API |
| `ALPHAKEY_USER_ID` | — | ⬜ | Default user UUID for multi-tenant MinIO namespace |
| `SECRETS_SOURCE` | `env` | ⬜ | `env` (vars) or `alphakey` (vault-fetched) |
| `AUTH_MODE` | `legacy` | ⬜ | `legacy` (no JWT) or `alphakey` (JWT required) |

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
    embargo_bars: 10

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

Gate failure error format: `gate:{category}:{decision}: {reason}` (e.g. `gate:backtest:reject: sharpe=0.12 < min_sharpe=0.50`)

---

## No Override Priority Chain

alphaGen has no runtime config overrides. Config is immutable per-run (stored in `run.config_yaml`). Validation gate thresholds are the only mutable config — via DB (PATCH /config/validation).

> [!note] Compare to alphaTrade
> [[services/alphaTrade/alphaTrade|alphaTrade]] has a full YAML → DB override priority chain. alphaGen does not — each run captures its full config at submission time.
