---
service: alphaTrade
page: Architecture
tags:
  - service/alphaTrade
  - architecture
---

# alphaTrade — Architecture

[[services/alphaTrade/alphaTrade|alphaTrade]] · [[services/alphaTrade/Interactions|Interactions]] · [[services/alphaTrade/API|API]] · [[services/alphaTrade/Data|Data]] · [[services/alphaTrade/Config|Config]]

---

## Purpose

alphaTrade is the live trading executor. It continuously polls for new ONNX models, runs inference at each bar close (NYSE-calendar aware), fuses signals from multiple models via softmax consensus, applies risk gates, and submits market + OCO bracket orders to Trading 212.

---

## Internal Modules

| Module | Path | Responsibility |
|---|---|---|
| `api` | `alphaTrade/api/` | FastAPI app, 40+ routers, SSE event bus, auth middleware |
| `broker` | `alphaTrade/broker/` | T212Client, order submission, OCO monitor, instrument map |
| `scheduler` | `alphaTrade/scheduler/` | NYSE bar-close scheduler + APScheduler backtest cron |
| `risk` | `alphaTrade/risk/` | Risk gates, position sizing (fixed/ATR/VIX), retirement, sector limits |
| `consensus` | `alphaTrade/consensus/` | Softmax-averaged multi-model signal fusion |
| `store` | `alphaTrade/store/` | SQLModel ORM, 23 tables, Alembic migrations, repos |
| `backtest` | `alphaTrade/backtest/` | Bar-by-bar simulation engine + reporter |
| `notify` | `alphaTrade/notify/` | Slack/Email/Webhook alerts with rate limiting |
| `data` | `alphaTrade/data/` | OHLCV provider abstraction (yfinance / Polygon) |
| `security` | `alphaTrade/security/` | API key auth, T212 credential management |
| `cache` | `alphaTrade/cache/` | VIX caching, sector caching |
| `adapter` | `alphaTrade/adapter/` | Model loading, ONNX runtime, manifest parsing |

---

## Scheduler Tick Sequence

The core trading loop executes at each NYSE bar close:

```mermaid
sequenceDiagram
    participant SCHED as BarCloseScheduler
    participant SYNC as ModelSyncDaemon
    participant FEAT as FeaturePipeline
    participant INF as OnnxModel
    participant CONS as Consensus
    participant RISK as RiskGates
    participant BROKER as T212Client
    participant DB as Database
    participant NOTIFY as AlertManager

    Note over SCHED: NYSE bar close fires
    SCHED->>FEAT: fetch OHLCV (ticker, interval, warmup bars)
    FEAT->>FEAT: compute TA-Lib indicators
    FEAT->>FEAT: normalize (zscore/minmax)
    FEAT->>FEAT: assemble window tensor
    FEAT->>INF: run inference (per model)
    INF-->>CONS: logits (shape 3 per model)
    CONS->>CONS: softmax(logits) → avg → argmax
    CONS-->>RISK: signal = BUY|SELL|HOLD
    RISK->>DB: check halt / cooldown / positions
    RISK->>RISK: gate checks (8 gates)
    alt Gate approved
        RISK->>BROKER: submit_market_order
        BROKER->>T212Client: place_market_order
        T212Client-->>DB: record Order
        BROKER->>BROKER: spawn OCO monitor task
        DB->>DB: record Signal, update Position
        NOTIFY->>NOTIFY: send fill alert
    else Gate rejected
        DB->>DB: record Signal (rejected)
        NOTIFY->>NOTIFY: send gate-reject alert (if configured)
    end
```

---

## Model Lifecycle in alphaTrade

```mermaid
stateDiagram-v2
    [*] --> launching: POST /models/{name}/promote (MLflow → Production)
    launching --> active: ModelSyncDaemon downloads + loads into registry
    active --> retired: AutoRetirement (win_rate/rolling_pnl below threshold)
    active --> disabled: PUT /models/{name}/overrides {enabled: false}
    active --> deleted: DELETE /models/{name}
    launching --> failed: Download fails (manifest hash mismatch, ONNX corrupt)
    failed --> launching: POST /models/{name}/retry-deploy
```

---

## Risk Gate Sequence (8 checks)

Applied in order before any order:

1. **Daily loss halt** — if daily PnL ≤ `daily_loss_halt_pct` and signal ≠ HOLD → REJECT
2. **HOLD signal** — no order
3. **Model retirement check** — if `model_performance.retired=true` → REJECT
4. **Cooldown check** — if `position.cooldown_until_ts > now` → REJECT
5. **Sector gate (BUY)** — portfolio mode balanced/unbalanced sector limit → REJECT if cap breached
6. **Max positions cap (BUY)** — if open_count ≥ `max_positions` → REJECT
7. **Pyramiding gate (BUY)** — `safe_mode=true` + existing position → REJECT; intraday intervals block pyramid unless `dangerously_allow_pyramid=true`
8. **Short-sell prevention (SELL)** — no existing long position → REJECT

---

## OCO Monitor (per-position async task)

After a BUY order fills, a dedicated async task polls both bracket legs every 10s:
- On SL fill → cancel TP, record `TradeJournal (exit_reason=OCO_SL)`, check retirement, set cooldown
- On TP fill → cancel SL, record `TradeJournal (exit_reason=OCO_TP)`, check retirement, set cooldown
- On non-fill terminal (CANCELLED/REJECTED) → cancel other leg, send warning webhook

---

## Model Auto-Retirement

`risk.performance.check_retirement()` runs after each trade record:
- Evaluates: `rolling_win_rate < min_win_rate` AND `rolling_pnl < min_rolling_pnl`
- Both age gates must pass: `trade_count >= min_trades_before_evaluation` AND age ≥ `min_evaluation_period`
- Sets `model_performance.retired=true` — model excluded from future ticks but NOT deleted

---

## Key Design Decisions

- **DB-first overrides**: YAML seeds Settings at startup; DB values overwrite on each tick/API update — enables hot-reload without restart. See [[services/alphaTrade/Config]] for full chain.
- **ONNX Runtime only**: No PyTorch dependency in alphaTrade. Manifest provides everything needed for inference.
- **Softmax consensus**: Multi-model averaging prevents single-model noise driving expensive orders.
- **Kill switch = HALT file**: Inference continues; only order submission is blocked. Allows diagnosis without losing market state.
- **Health on separate port 8080**: Container probe (`GET /healthz`) decoupled from API auth on 8081.
