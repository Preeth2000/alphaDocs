---
page: Features
tags:
  - platform
  - features
last-reviewed: 2026-06-06
---

# Platform Features

[[README]] · [[platform/Overview]] · [[platform/Tech-Stack]] · [[platform/Key-Decisions]]

All features across the alphaPlatform, grouped by domain.

---

## Model Training ([[services/alphaGen/alphaGen|alphaGen]])

| Feature | Description |
|---|---|
| Multi-architecture training | MLP, LSTM, CNN, Transformer models — all exportable to ONNX |
| Walk-forward cross-validation | N-fold walk-forward splits with configurable embargo bars (prevents look-ahead) |
| Multi-class classification | BUY / SELL / HOLD signal generation |
| TA-Lib feature engineering | RSI, MACD, Bollinger Bands, ATR, EMA, SMA, ADX, Stochastic, OBV |
| Feature normalization | Z-score, min-max, or none — per-column stats stored in manifest |
| Multiple label strategies | ForwardReturn (horizon_bars + threshold) and TripleBarrier |
| Class weight balancing | Auto-computed or manual class weights for imbalanced data |
| Early stopping | Configurable patience on validation loss |
| Multi-ticker support | Align multiple tickers as features |
| Fundamentals integration | EPS, EPS surprise, sector one-hot encoding (optional) |
| Async job management | FastAPI + Celery — submit, cancel, list, monitor training jobs |
| Real-time log streaming | SSE-based training log stream from worker to browser (Redis pub/sub) |
| Force-save bypass | Re-queue gate-failed runs to publish without validation (operator escape hatch) |
| Run visibility | `private` or `public` — public models shared in platform library |
| Hyperparameter sweeps | Optuna-based sweep (CLI) with configurable search space |
| Stale job cleanup | Jobs stuck `running` > 30min or `queued` > 5min auto-marked failed |

---

## Validation Gate ([[services/alphaGen/alphaGen|alphaGen]])

| Feature | Description |
|---|---|
| Configurable gate thresholds | Min Sharpe, max drawdown, min hit rate, min trades, max val loss, min accuracy, min F1 (class 1) |
| DB-stored thresholds | PATCH /config/validation to update at runtime without restart |
| Gate bypass | Force-save creates a new run with `force_save=True` (audit trail preserved) |
| Backtest validation | Gate evaluates backtest metrics, not just training metrics |
| RetrainManager | Budget-based retrain limiting (tracks retrain count) |

---

## Model Publishing & Registry ([[services/alphaGen/alphaGen|alphaGen]] + [[services/alphaFrame/alphaFrame|alphaFrame]])

| Feature | Description |
|---|---|
| ONNX export | Framework-agnostic export (opset configurable); parity verification between PyTorch and ORT |
| Manifest contract | `manifest.json` captures feature names, window, norm stats, input shape, model hash — alphaTrade needs nothing else |
| MinIO artifact storage | Versioned paths: `{user}/{account}/{run_name}/{version}/` |
| Latest pointer | `{run_name}/latest` JSON blob for polling consumers |
| MLflow model registry | Staging → Production lifecycle; human-review checkpoint |
| model.ready events | Redis pub/sub + SSE endpoint notify alphaTrade on publish |
| Public model library | Public runs visible to all users; fork/adopt workflow |
| MLflow model promotion | POST /models/{name}/promote — set Production alias |
| MLflow model demotion | POST /models/{name}/demote — revert Production → Staging |

---

## Trading Execution ([[services/alphaTrade/alphaTrade|alphaTrade]])

| Feature | Description |
|---|---|
| ONNX model loading | onnxruntime inference; manifest-driven feature pipeline replication |
| Auto model discovery | ModelSyncDaemon polls MinIO + MLflow every 60s; hot-swaps new models |
| Multi-model consensus | Softmax-averaged logit fusion for same-ticker models → single signal |
| Model retirement | Auto-retires underperformers (rolling win rate + P&L thresholds) |
| NYSE bar-close scheduler | Exchange-calendar aware; 1m/5m/15m/1h/1d/1wk intervals |
| Extended hours mode | Optional tick firing outside NYSE regular session |
| Kill switch | HALT file pauses order submission while inference continues |
| Multi-account T212 | Demo / Invest / ISA accounts; switchable via `t212_active_account` |
| OCO bracket orders | Stop-loss + take-profit pair per position; async monitor with 10s poll |
| Order idempotency | Deterministic `client_order_id` (SHA-256 of ticker+bar+side) prevents duplicate orders |
| T212 rate limiting | Exponential backoff; 429 handling with Retry-After; per-type min gap throttle |
| T212 credential verification | POST /verify/t212 tests credentials before saving |
| Data provider abstraction | yfinance (default) or Polygon.io; switchable at runtime |
| Hot-reload settings | PUT /settings reinitialises T212Client + DataProvider without restart |

---

## Risk Management ([[services/alphaTrade/alphaTrade|alphaTrade]])

| Feature | Description |
|---|---|
| Daily loss halt | Halt new orders when intraday PnL ≤ `daily_loss_halt_pct` |
| Max positions cap | Reject BUY when open positions ≥ `max_positions` |
| Cooldown bars | Post-close wait before re-entering same instrument |
| Pyramiding control | `safe_mode` blocks new BUY if position already open; per-interval intraday pyramid prevention |
| Sector limits (balanced mode) | Cap portfolio exposure per sector (`max_sector_pct`) |
| Sector limits (unbalanced mode) | Cap positions per sector (`max_per_sector`), with per-sector overrides |
| Position sizing — fixed | `equity × size_pct` |
| Position sizing — ATR | Dollar-risk-based sizing using ATR volatility |
| Position sizing — VIX | VIX-scaled sizing (higher volatility → smaller size) |
| Short-sell prevention | Reject SELL with no existing long position |

---

## Backtesting ([[services/alphaTrade/alphaTrade|alphaTrade]])

| Feature | Description |
|---|---|
| Bar-by-bar simulation | Full OCO simulation with SL/TP triggering via intraday high/low |
| Slippage + commission | Configurable `slippage_bps` and `commission_per_trade` |
| OCO lag simulation | Realistic fill gap simulation for stop/limit order execution |
| Per-model results | Trade count, win rate, rolling P&L per model |
| Scheduled backtests | APScheduler cron runs backtest automatically (default: 02:00 daily) |
| Per-model schedule overrides | Disable or customise cron + lookback per model |
| Backtest report | Sharpe, max drawdown, win rate, profit factor, avg win/loss, equity curve |

---

## Alerting ([[services/alphaTrade/alphaTrade|alphaTrade]])

| Feature | Description |
|---|---|
| Slack alerts | Configurable webhook, per-level threshold (INFO/WARNING/ERROR/CRITICAL) |
| Email alerts (SMTP) | TLS SMTP, configurable recipients and min level |
| Generic webhook | HTTP POST for custom integrations |
| Rate limiting | 30s per-category window prevents flood |
| Async delivery | Non-blocking queue (64-item) — alert delivery doesn't block trading loop |
| Alert events | Fills, order errors, OCO executions, daily loss halt, model retirement, kill switch, health degradation |

---

## Authentication & Security ([[services/alphaKey/alphaKey|alphaKey]])

| Feature | Description |
|---|---|
| ES256 JWT access tokens | Short-lived (10min), signed with ECDSA P-256 |
| Opaque refresh tokens | 14-day, cryptographically random, hash-stored |
| Token rotation | Every refresh rotates the refresh token |
| Reuse detection | Presenting revoked refresh token wipes all sessions for user |
| Instant token revocation | JTI denylist in Redis — logout takes effect immediately |
| Offline revocation (token_version) | Bump `token_version` on logout-all/password-change; JWT carries `tv` claim |
| Argon2 password hashing | With transparent rehash on login (algorithm upgrade path) |
| Credential vault | Per-user encrypted secret storage (T212, Polygon, SMTP, Slack, etc.) |
| Envelope encryption | Per-secret DEK encrypted by KEK; supports KEK rotation without re-encrypting secrets |
| Credential masking | Vault list returns last 4 chars only — plaintext never in API responses |
| Credential connectivity verify | POST /vault/{provider}/verify tests live API connectivity with stored key |
| Role-based access | admin / developer / standard; first user auto-promoted to developer |
| Admin kill-switch | Force logout all sessions for any user |
| Audit logs | Append-only AuditLog + CredentialAccessAudit tables |
| JWKS endpoint | Public signing keys for offline JWT verification by other services |
| Service-to-service auth | `X-Service-Token` header for introspect + secrets endpoints |

---

## Frontend / UX ([[services/alphaLink/alphaLink|alphaLink]])

| Feature | Description |
|---|---|
| Multi-step config builder | Guided training config (Data → Model → Train) |
| Config templates | Save, load, rename, delete RunConfig templates |
| Live training log viewer | SSE-streamed training logs in browser during training |
| Model comparison | Side-by-side comparison of training run metrics |
| Real-time trading dashboard | Live positions, orders, signals, equity curve via SSE |
| Kill switch UI | One-click halt/resume from dashboard |
| Settings page | Full BotSettings editor (T212 credentials, risk config, alerts) |
| Model management | Browse/promote/demote/delete/fork models |
| Backtest UI | Trigger, view, and analyse backtests |
| Public model library | Browse and adopt models published by other users |
| Account vault UI | Encrypted credential management |
| Polygon key management | Set/test Polygon.io API key from UI |
| BFF pattern | All backend calls proxied server-side — service URLs never exposed to browser |
| httpOnly refresh cookie | Refresh token inaccessible to JavaScript |

---

## Observability ([[services/alphaFrame/alphaFrame|alphaFrame]])

| Feature | Description |
|---|---|
| OpenTelemetry instrumentation | All Python services emit traces + metrics + logs via OTLP |
| Distributed tracing | Tempo stores traces; Grafana trace explorer with TraceQL |
| Metrics | Prometheus scrapes all services + infra exporters (postgres, redis, node, cadvisor, Celery) |
| Log aggregation | Loki stores all container logs (Docker filelog receiver) |
| Grafana dashboards | Pre-built: Platform Services (request rate, error rate, p50/p99 latency) + Celery (task success/failure, duration) |
| Exemplars | Metric data points link directly to traces in Grafana |
| Alerting | Prometheus → Alertmanager → webhook (Slack/PagerDuty-ready) |
| 7 alert rules | ServiceDown, HighErrorRate, HighLatencyP99, CeleryTaskFailures, PostgresDown, HighMemoryUsage, DiskPressure |
| alphaTrade Prometheus metrics | Signals, orders, inference latency, T212 request latency, positions, equity, daily P&L |
