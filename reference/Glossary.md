---
page: Glossary
tags:
  - reference
  - glossary
---

# Glossary

[[README]] · [[reference/Ports-and-Endpoints]] · [[reference/Event-Channels]]

Domain terms used across projectAlpha documentation.

---

## A

**ATR (Average True Range)**
Volatility indicator used by [[services/alphaTrade/alphaTrade|alphaTrade]] for dynamic position sizing and stop-loss calculations.

**att**
The Python library package inside [[services/alphaGen/alphaGen|alphaGen]] (`src/att/`). Contains all model-training logic. Can run as a CLI (`att train`, `att validate`, `att backtest`) or be invoked by the Celery worker via the FastAPI layer.

---

## B

**Backtest**
Replay of a trained model's signals against historical OHLCV data to measure expected performance before live deployment. Executed by [[services/alphaGen/alphaGen|alphaGen]] (`att backtest`) and [[services/alphaTrade/alphaTrade|alphaTrade]] (live backtester).

**BFF (Backend for Frontend)**
Pattern used by [[services/alphaLink/alphaLink|alphaLink]]: Next.js API routes (`/app/api/*`) proxy browser requests to backend services (alphaGen, alphaTrade, alphaKey), avoiding direct browser-to-service calls and hiding service URLs from the client.

**Beads**
GasTownHall Beads — state/data management library. Configured in each service repo (`.beads/`). Used as the primary pattern for state management where active.

**Broker**
External trading platform integration. Currently: Trading 212. Managed by [[services/alphaTrade/alphaTrade|alphaTrade]] broker module.

---

## C

**Celery**
Distributed task queue used by [[services/alphaGen/alphaGen|alphaGen]] to run `att.train_and_export` jobs asynchronously. Broker: Redis db0. Result backend: Redis db1.

**Consensus**
Module in [[services/alphaTrade/alphaTrade|alphaTrade]] that aggregates signals from multiple models before deciding whether to place an order. Prevents single-model noise driving trades.

**Cooldown bars**
After closing a position, the number of price bars that must pass before the scheduler can re-enter the same instrument. Configured via `overrides.yaml` or DB override.

---

## D

**Daily loss halt**
Risk circuit-breaker in [[services/alphaTrade/alphaTrade|alphaTrade]]. When intraday PnL drops below `daily_loss_halt_pct` of equity, the bot halts new orders until the next trading day. Configured in `overrides.yaml` → `risk.daily_loss_halt_pct`.

---

## E

**Extended hours**
Flag allowing the scheduler in [[services/alphaTrade/alphaTrade|alphaTrade]] to fire outside NYSE regular session hours. Default: `false`.

---

## F

**Force-save**
[[services/alphaGen/alphaGen|alphaGen]] feature: when a training run fails the validation gate, an operator can re-queue it with `force_save=True` to bypass the gate and publish anyway. Endpoint: `POST /runs/{id}/force-save`.

---

## G

**Gate (Validation gate)**
Quality threshold checked by [[services/alphaGen/alphaGen|alphaGen]] after training. If metrics (Sharpe, drawdown, accuracy, etc.) don't meet thresholds, the run is flagged `gate_failed` and not published. Thresholds in `validation_settings` DB table.

---

## M

**Manifest**
`manifest.json` stored alongside `model.onnx` in MinIO. Contains model metadata: run_name, version, thresholds, feature list, training params. Read by [[services/alphaTrade/alphaTrade|alphaTrade]] when loading models.

**MinIO**
S3-compatible object store in [[services/alphaFrame/alphaFrame|alphaFrame]]. Primary artifact storage for trained models (`model.onnx`, `manifest.json`).

**MLflow**
ML experiment tracker in [[services/alphaFrame/alphaFrame|alphaFrame]]. [[services/alphaGen/alphaGen|alphaGen]] logs runs, params, metrics, and registers model versions here.

**model.ready**
Redis pub/sub channel (and SSE endpoint in alphaGen) that fires when a model is published to MinIO. [[services/alphaTrade/alphaTrade|alphaTrade]] subscribes to load new models automatically.

---

## O

**OCO (One-Cancels-Other)**
Order type: a paired stop-loss and take-profit bracket. When one leg fills, the other is cancelled. Implemented in [[services/alphaTrade/alphaTrade|alphaTrade]].

**OHLCV**
Open / High / Low / Close / Volume — standard candlestick price data format. Used by [[services/alphaGen/alphaGen|alphaGen]] for training features and [[services/alphaTrade/alphaTrade|alphaTrade]] for live price ticks.

**OTel (OpenTelemetry)**
Distributed tracing / metrics collection standard. [[services/alphaFrame/alphaFrame|alphaFrame]] runs OTel Collector to receive traces from all services.

---

## R

**Run**
A single model-training job in [[services/alphaGen/alphaGen|alphaGen]]. Created via `POST /runs`, executed by Celery worker, tracked in `runs` Postgres table. Has lifecycle states: `queued → running → gate_passed | gate_failed → published`.

**run_name**
Unique identifier for a model configuration (e.g. `aapl_daily_mlp`). Used as key in `overrides.yaml` per-model section and in MinIO bucket paths.

---

## S

**Scheduler**
[[services/alphaTrade/alphaTrade|alphaTrade]] component that fires model-inference ticks on a time schedule, applies risk checks, and submits orders to the broker.

**SSE (Server-Sent Events)**
One-way HTTP streaming. Used in [[services/alphaGen/alphaGen|alphaGen]] to stream training log lines (`GET /runs/{id}/log`) and broadcast `model.ready` events (`GET /runs/events`).

**Sweep**
Hyperparameter sweep in [[services/alphaGen/alphaGen|alphaGen]] — runs multiple training configurations to find optimal params (e.g. using Optuna or custom grid).

---

## T

**T212 / Trading 212**
Primary broker integration in [[services/alphaTrade/alphaTrade|alphaTrade]]. Accessed via REST API. Supports multi-account configuration.

---

## V

**VIX**
Volatility Index. Used by [[services/alphaTrade/alphaTrade|alphaTrade]] risk module as a market-regime signal to scale position sizes or halt trading.

---

*Add new terms alphabetically. Link to the owning service page.*
