---
service: alphaGen
page: Data
tags:
  - service/alphaGen
  - data
---

# alphaGen — Data

[[services/alphaGen/alphaGen|alphaGen]] · [[services/alphaGen/Architecture|Architecture]] · [[services/alphaGen/Interactions|Interactions]] · [[services/alphaGen/API|API]] · [[services/alphaGen/Config|Config]]

---

## Datastores

| Store | Type | Purpose |
|---|---|---|
| `alphagen` (Postgres) | Relational | Job state machine + gate config |
| Redis db0 | Pub/sub | Log streaming + model.ready events |
| Redis db0 | Celery broker | Task routing |
| Redis db1 | Celery results | Task state + return values |
| MinIO `models` bucket | Object store | model.onnx, manifest.json, backtest.json, drift_reference.json |
| MLflow `mlflow` DB | Relational | Experiment params, metrics, model registry |

---

## PostgreSQL Tables

Database: `alphagen`  
Managed by: Alembic (runs `alembic upgrade head` on `alphagen-api` startup)  
Migrations: `alembic/versions/`

### `run`

Primary state machine for training jobs.

| Column | Type | Purpose |
|---|---|---|
| `id` | String PK | Nano-ID job identifier |
| `run_name` | String (indexed) | Model name (e.g. `aapl_daily_mlp`) |
| `config_yaml` | Text | Full YAML config used for this run |
| `status` | Enum | `queued \| running \| complete \| failed \| cancelled` |
| `celery_task_id` | String nullable | Celery task ID (for revoke) |
| `created_at` | DateTime | UTC |
| `started_at` | DateTime nullable | UTC |
| `finished_at` | DateTime nullable | UTC |
| `sharpe` | Float nullable | Backtest Sharpe ratio |
| `max_drawdown` | Float nullable | Backtest max drawdown |
| `hit_rate` | Float nullable | Backtest hit rate |
| `n_trades` | Int nullable | Backtest trade count |
| `total_return` | Float nullable | Backtest total return |
| `minio_version` | String nullable | Published version string (e.g. `v3`) |
| `artifact_prefix` | String nullable | MinIO path prefix |
| `error` | Text nullable | Gate failure reason or exception message |
| `per_class` | JSON nullable | Per-class backtest stats |
| `force_save` | Bool | Was gate bypassed? |
| `source_run_id` | String nullable | Original run ID if this is a force-save retry |
| `user_id` | String(36) (indexed) | Owner user UUID |
| `visibility` | String | `private \| public` |

### `validation_settings`

Singleton row (id=1) — validation gate thresholds.

| Column | Type | Default | Purpose |
|---|---|---|---|
| `id` | Int PK=1 | 1 | Singleton |
| `enabled` | Bool | `true` | Gate on/off |
| `min_sharpe` | Float | `0.5` | Minimum Sharpe ratio |
| `max_drawdown` | Float | `0.20` | Maximum drawdown (fraction) |
| `min_hit_rate` | Float | `0.45` | Minimum hit rate |
| `min_n_trades` | Int | `5` | Minimum trades in backtest |
| `max_val_loss` | Float nullable | — | Maximum validation loss |
| `min_val_accuracy` | Float nullable | — | Minimum validation accuracy |
| `min_f1_class1` | Float nullable | — | Minimum F1 for SELL class |

---

## DB Read/Write Count per Operation

| Operation | Reads | Writes | Tables |
|---|---|---|---|
| `POST /runs` — create | 0 | 1 | `run` |
| `GET /runs` — list | 1 | 0 | `run` |
| `GET /runs/{id}` — fetch | 1 | 0 | `run` |
| `DELETE /runs/{id}` — cancel | 1 | 1 | `run` |
| `POST /runs/{id}/force-save` | 1 | 2 | `run` (INSERT new + UPDATE original) |
| `POST /runs/{id}/publish` | 1 | 1 | `run` |
| `GET /runs/{id}/log` — SSE | 1 | 0 | `run` |
| `GET /config/validation` | 1 | 0 | `validation_settings` |
| `PATCH /config/validation` | 1 | 1 | `validation_settings` |
| `GET /models` | 0 | 0 | — (MinIO only) |
| `GET /health` | 1 | 0 | `SELECT 1` |
| Celery task: status transitions | 1–3 | 3–5 | `run` (running→complete/failed) |

---

## Migration History

| Migration | Revision | Change |
|---|---|---|
| `001_initial_schema` | 626178708997 | Create `run` table + indexes |
| `002_add_per_class` | a1b2c3d4e5f6 | Add `run.per_class` JSON column |
| `003_add_force_save` | b2c3d4e5f6a7 | Add `run.force_save`, `run.source_run_id` |
| `004_add_user_id_to_run` | c3d4e5f6a7b8 | Add `run.user_id` (multi-user) |
| `005_add_visibility_to_run` | d4e5f6a7b8c9 | Add `run.visibility` (public library) |

---

## MinIO Object Layout

Bucket: `models` (configurable via `MINIO_BUCKET`)

```
{MINIO_USER}/{MINIO_ACCOUNT}/{run_name}/
├── latest                        # JSON pointer → current version
├── v1/
│   ├── model.onnx                # ONNX model (required)
│   ├── manifest.json             # Inference contract: feature_names, window, norm_stats, input_shape, model_hash
│   ├── backtest.json             # Backtest report: sharpe, max_drawdown, hit_rate, n_trades, per_class
│   └── drift_reference.json      # Per-feature histogram bins for drift monitor (optional — non-fatal if absent)
├── v2/
│   └── ...
```

`drift_reference.json` is written during training (last-fold normalized windows), uploaded alongside required artifacts on publish. Used exclusively by the `check_drift_and_retrain` Beat task — not part of the inference contract (`manifest.json` is the inference contract).

---

## OHLCV Data Validation

`validate_ohlcv()` runs after every fetch (yfinance and Polygon) and raises `DataQualityError` if:

| Check | Threshold | Error |
|---|---|---|
| Duplicate timestamps | Any | `N duplicate timestamp(s)` |
| Zero/negative OHLC | Any bar | `N zero/negative value(s) in {col}` |
| Negative volume | Any bar | `N negative volume value(s)` |
| High < Low | Any bar | `N bar(s) with High < Low` |
| Extreme single-bar return | >90% | `N bar(s) with >90% single-bar price move` |
| Daily coverage | <70% of estimated trading days | `fetched N bars, expected ~M trading days (X% coverage)` |

Validation runs before feature engineering — a bad fetch fails the run immediately with a clear error message rather than silently poisoning model weights.

---

## OHLCV Feature Notes

| Feature | Behavior |
|---|---|
| **VWAP** (`include_vwap: true`) | Intraday (`1m`/`5m`/`15m`/`1h`): per-session (calendar-day) cumulative VWAP — resets each day. Daily/weekly: rolling 20-bar VWAP. Cumulative-from-dataset-start is non-stationary and diverges between training and live inference (different warmup offsets). |
| **Fundamentals earnings** | `earnings_dates` index normalized to tz-naive by `tz_localize(None)` if tz-aware; tz-naive index left as-is. Data shifted forward by `announcement_lag_days` (default 1) to prevent look-ahead from after-hours releases. Failure to fetch logs a warning — eps columns are absent but training continues. |
| **Multi-ticker union+ffill** | `alignment='union'` with `fill='ffill'` fabricates flat bars (identical OHLCV) for tickers missing on a given date. These synthetic bars feed labels and create false signals. Prefer `alignment='intersection'` or `fill='drop'`. A `UserWarning` is emitted when this combination is detected. |

---

## MinIO Object Layout

Bucket: `models`

```
{minio_user}/{minio_account}/{run_name}/{version}/
├── model.onnx          — ONNX model binary
├── manifest.json       — Feature contract for alphaTrade
└── backtest.json       — Backtest results summary

{minio_user}/{minio_account}/{run_name}/latest
  — JSON: {version, uploaded_at, run_name}  (polling pointer)
```

---

## Redis Pub/Sub

See [[reference/Event-Channels]] for full payload formats.

| Channel | DB | Publisher | Subscriber |
|---|---|---|---|
| `run:{id}:log` | db0 | Celery worker (RedisLogHandler) | `GET /runs/{id}/log` SSE endpoint |
| `model.ready` | db0 | Celery worker (on success) + `POST /runs/{id}/publish` | `GET /runs/events` SSE → [[services/alphaTrade/alphaTrade\|alphaTrade]] |

---

## MLflow Data

| What | Where | Written by | Read by |
|---|---|---|---|
| Experiments (per ticker) | MLflow `mlflow` DB | Celery worker | MLflow UI, alphaTrade |
| Run params (arch, lr, etc.) | MLflow `mlflow` DB | `att.mlflow_utils.log_and_register_run()` | MLflow UI |
| Run metrics (sharpe, drawdown, etc.) | MLflow `mlflow` DB | Same | MLflow UI, alphaTrade |
| Registered model | MLflow `mlflow` DB | Same | alphaTrade promote endpoint |
| Model alias (`staging` / `production`) | MLflow `mlflow` DB | `POST /models/{name}/promote` | [[services/alphaTrade/alphaTrade\|alphaTrade]] `ModelSyncDaemon` |
| model.onnx (MLflow artifact) | MinIO `mlflow` bucket | MLflow (via SDK) | MLflow UI |
