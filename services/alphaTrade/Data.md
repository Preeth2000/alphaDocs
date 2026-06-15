---
service: alphaTrade
page: Data
tags:
  - service/alphaTrade
  - data
---

# alphaTrade — Data

[[services/alphaTrade/alphaTrade|alphaTrade]] · [[services/alphaTrade/Architecture|Architecture]] · [[services/alphaTrade/Interactions|Interactions]] · [[services/alphaTrade/API|API]] · [[services/alphaTrade/Config|Config]]

---

## Datastores

| Store | Type | Purpose |
|---|---|---|
| `alphatrade` Postgres (or SQLite fallback) | Relational | All trading state — 23 tables |
| MinIO `models` bucket | Object store | ONNX model + manifest download |
| MLflow `mlflow` DB | Relational | Model registry (read + write) |
| Redis (optional) | Cache | Token denylist (when AUTH_MODE=alphakey) |

---

## Database Tables (23)

Database: `alphatrade` (Postgres) or `state.db` (SQLite fallback)  
Managed by: Alembic (run on startup via `run_migrations()`)  
Migrations: `alphaTrade/store/migrations/versions/` (migrations 0001–0023)

| Table | Purpose | Key Columns |
|---|---|---|
| `signal` | Generated signals (per tick, per model) | `ts`, `run_name`, `ticker`, `signal` (BUY/SELL/HOLD), `model_count`, `raw_json`, `user_id` |
| `order` | Order lifecycle | `ts`, `signal_id`, `t212_ticker`, `side`, `quantity`, `status` (pending/filled/rejected/error), `t212_order_id`, `fill_price`, `client_order_id` (indexed), `user_id` |
| `position` | Open positions + OCO state + cooldown | `t212_ticker` (unique), `quantity`, `avg_entry`, `opened_at`, `cooldown_until_ts`, `stop_order_id`, `limit_order_id`, `user_id` |
| `equitycurve` | Daily equity snapshots | `ts`, `equity`, `user_id` |
| `instrumentcache` | yfinance → T212 ticker resolution | `yf_ticker` (unique), `t212_ticker`, `resolved_at` |
| `tradejournal` | Closed trades for P&L analysis | `ts`, `model_id`, `ticker`, `entry/exit_price`, `quantity`, `hold_bars`, `exit_reason` (OCO_SL/OCO_TP/SIGNAL_SELL/HALT/MANUAL), `sl/tp_price`, `realized_pnl`, `pnl_pct`, `user_id` |
| `pnlsnapshot` | Daily P&L aggregation | `date` (unique), `total_equity`, `day_pnl`, `day_pnl_pct`, `realized_pnl`, `unrealized_pnl`, `positions_json`, `open_positions`, `trade_count`, `user_id` |
| `model_performance` | Auto-retirement tracking | `model_id` (unique), `trade_count`, `win_count`, `rolling_pnl`, `rolling_trades_json`, `retired`, `retired_at`, `first_trade_at` |
| `sectorcache` | yfinance sector lookups | `yf_ticker` (unique), `sector`, `resolved_at` |
| `backtest_run` | Backtest job metadata | `ts`, `start_date`, `end_date`, `config_json`, `status` (done/queued/running/failed), `error_msg`, `user_id` |
| `backtest_trade` | Simulated trades per backtest | `run_id` (indexed), `model_id`, `side`, `entry/exit_time`, `entry/exit_price`, `quantity`, `exit_reason`, `realized_pnl`, `sl/tp_price` |
| `backtest_model_run` | Per-model backtest summary | `run_id` (indexed), `model_id`, `ticker`, `interval`, `trade_count`, `status` (ran/no_data/failed), `error_msg` |
| `bot_settings` | Global bot config singleton (id=1) | T212 credentials (3 accounts), data_provider, sizing, risk, retirement, alerts, backtest config, `user_id` |
| `model_override` | Per-model config overrides | `run_name` (PK), `enabled`, `broker_ticker`, sizing/SL/TP/cooldown overrides, `safe_mode`, `pyramid`, retirement overrides, backtest overrides, `consensus_min_confidence`, `consensus_min_margin` (per-model consensus gates), `visibility` (private/public), `updated_at` |
| `model_deployments` | MLflow deployment lifecycle | `run_name` (indexed), `user_id`, `status` (launching/active/failed), `promoted_at`, `activated_at`, `failed_at`, `failure_msg` |
| `model_adoptions` | Public model adoption tracking | `user_id` (indexed), `model_name` (indexed), `source_user_id`, `artifact_prefix`, `adopted_at` |

---

## DB Read/Write Count per Operation

| Operation | Reads | Writes | Tables |
|---|---|---|---|
| Per-tick signal generation | 2–4 | 1–3 | `signal`, `position`, `model_performance` |
| Order placement | 1 | 2 | `order`, `position` |
| OCO fill → trade closed | 2 | 3 | `order`, `position`, `tradejournal`, `model_performance` |
| `GET /positions` | 1 | 0 | `position` |
| `GET /orders` | 1 | 0 | `order` |
| `GET /signals` | 1 | 0 | `signal` |
| `GET /pnl` | 1 | 0 | `pnlsnapshot` |
| `GET /trades` | 1 | 0 | `tradejournal` |
| `GET /equity-curve` | 1 | 0 | `equitycurve` |
| `GET /models` | 3 | 0 | `model_performance`, `model_override`, `model_deployments` (+MLflow) |
| `PUT /models/{name}/overrides` | 1 | 1 | `model_override` |
| `POST /backtest/trigger` | 0 | 1 | `backtest_run` |
| `GET /settings` | 1 | 0 | `bot_settings` |
| `PUT /settings` | 1 | 1 | `bot_settings` |
| `POST /models/{name}/promote` | 0 | 1 | `model_deployments` (+MLflow write) |

---

## Migration History (27 migrations)

| Migration | Change |
|---|---|
| 0001 | Create `signal`, `order`, `position`, `equitycurve`, `instrumentcache` |
| 0002 | Create `tradejournal`, `pnlsnapshot`, `model_performance`, `sectorcache`, `backtest_*` |
| 0003 | Create `bot_settings` (id=1 singleton) |
| 0004 | Add `stop_order_id`, `limit_order_id` to `position` |
| 0005 | Add `t212_*_secret_key` columns to `bot_settings` |
| 0006 | Create `model_override` (run_name PK) |
| 0007 | Add `status`, `error_msg` to `backtest_run` |
| 0008 | Add `t212_{demo\|invest\|isa}_*`, `t212_active_account` to `bot_settings` |
| 0009 | Add `retirement_*` columns to `bot_settings` |
| 0010 | Create `backtest_model_run` |
| 0011 | Add `safe_mode` to `bot_settings`, `model_override` |
| 0012 | Add `dangerously_allow_pyramid` to `bot_settings`, `model_override` |
| 0013 | Add `retirement_*`, `backtest_*` override cols to `model_override` |
| 0014 | Add sizing, portfolio, ATR, VIX, backtest settings to `bot_settings` |
| 0015 | Create `model_deployments` |
| 0016 | Convert raw_json, config_json, positions_json, rolling_trades_json to JSON type |
| 0017 | Add `error_msg` to `backtest_run` |
| 0018 | Add `user_id` (multi-tenant) to all major tables |
| 0019 | Ensure `user_id` on `tradejournal`, `pnlsnapshot`, `backtest_run` |
| 0020 | Drop `alphaTrade_api_key` column (moved to env var) |
| 0021 | Add `user_id` to `model_deployments` |
| 0022 | Create `model_adoptions` |
| 0023 | Add `visibility` to `model_override` |
| 0024 | Add `ts` index to `equitycurve` |
| 0025 | Add `oco_metadata` to `position` |
| 0026 | Add `consensus_min_confidence`, `consensus_min_margin` to `bot_settings` (global gates) |
| 0027 | Add `consensus_min_confidence`, `consensus_min_margin` to `model_override` (per-model gates) |

---

## MinIO Usage

| Action | Path | Trigger |
|---|---|---|
| Download `model.onnx` | `{user}/{account}/{run_name}/{version}/model.onnx` | ModelSyncDaemon — new Production alias found |
| Download `manifest.json` | `{user}/{account}/{run_name}/{version}/manifest.json` | Same |
| Read `latest` pointer | `{user}/{account}/{run_name}/latest` | ModelSyncDaemon poll |
| Delete model artifacts | `{user}/{account}/{run_name}/` (prefix) | `DELETE /models/{name}` |
