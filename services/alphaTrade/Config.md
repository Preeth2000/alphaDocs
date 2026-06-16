---
service: alphaTrade
page: Config
tags:
  - service/alphaTrade
  - config
---

# alphaTrade — Config

[[alphaTrade|alphaTrade]] · [[alphaDocs/services/alphaTrade/Architecture|Architecture]] · [[alphaDocs/services/alphaTrade/Interactions|Interactions]] · [[alphaDocs/services/alphaTrade/API|API]] · [[alphaDocs/services/alphaTrade/Data|Data]]

---

## Override Priority Chain

> [!important] DB-first overrides
> alphaTrade has the most complex config system on the platform. YAML seeds at startup; DB is authoritative at runtime.

```
Priority (highest → lowest):
1. Per-model ModelOverrideRecord (DB)   ← per-signal, per-model
2. BotSettings DB record (id=1)         ← hot-reloadable via PUT /settings
3. overrides.yaml                       ← seed at startup (never written at runtime)
4. Pydantic Settings defaults           ← code defaults
```

**Merge flow:**
1. On startup: `overrides.yaml` parsed → `Settings` object populated
2. `apply_bot_settings()` reads `BotSettings` DB → overwrites Settings fields
3. Per-tick: `_effective_config(global_cfg, per_model_override)` merges per-model DB record on top
4. On `PUT /settings`: BotSettings DB updated → `apply_bot_settings()` called → T212Client / DataProvider reinitialised if credentials changed

This means: changing `overrides.yaml` after startup has **no effect** without restart. All runtime changes must go via API `PUT /settings` (global) or `PUT /models/{name}/overrides` (per-model).

---

## Environment Variables

Source: `alphaTrade/.env.example`

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `T212_ACTIVE_ACCOUNT` | `demo` | ✅ | Which T212 account to trade: `demo` \| `invest` \| `isa` |
| `T212_DEMO_API_KEY` | — | ⬜ | T212 demo account API key |
| `T212_DEMO_SECRET_KEY` | — | ⬜ | T212 demo account secret (BasicAuth) |
| `T212_INVEST_API_KEY` | — | ⬜ | T212 invest account API key |
| `T212_INVEST_SECRET_KEY` | — | ⬜ | T212 invest account secret |
| `T212_ISA_API_KEY` | — | ⬜ | T212 ISA account API key |
| `T212_ISA_SECRET_KEY` | — | ⬜ | T212 ISA account secret |
| `DATA_PROVIDER` | `yfinance` | ⬜ | `yfinance` \| `polygon` |
| `POLYGON_API_KEY` | — | ⬜ | Polygon.io API key |
| `MODELS_DIR` | `./models` | ⬜ | Local path for downloaded ONNX models |
| `DATABASE_URL` | — | ⬜ | Postgres URL (overrides SQLite fallback) |
| `STATE_DB_PATH` | `./state.db` | ⬜ | SQLite DB path (used if DATABASE_URL not set) |
| `OVERRIDES_PATH` | `./overrides.yaml` | ⬜ | Path to overrides YAML |
| `LOG_FILE` | `./alphaTrade.log` | ⬜ | Log file path |
| `API_PORT` | `8081` | ⬜ | FastAPI listen port |
| `WEBHOOK_URL` | — | ⬜ | Generic webhook URL for alerts |
| `WEBHOOK_LEVEL` | `WARNING` | ⬜ | Minimum alert level for generic webhook |
| `AUTH_MODE` | `jwt` | ⬜ | `jwt` = require alphaKey Bearer JWT on all API endpoints (platform default); `legacy` = X-API-Key header auth |
| `PACT_VERIFICATION_MODE` | `false` | ⬜ | **Test-only.** When `true`, mounts the gated `POST /internal/pact-state` endpoint used by Pact provider verification (seeds/clears in-memory MLflow registry state and kill-switch sentinel). **Must be `false` in all non-CI environments** — the endpoint can flip the kill switch. Set to `true` only in the `pact-verify` CI job. |
| `ALPHATRADE_API_KEY` | — | ⬜ | API key used in `legacy` auth mode; if unset in legacy mode, requests are blocked (fail-closed) unless `ALPHATRADE_INSECURE_NO_AUTH=true` |
| `ALPHATRADE_INSECURE_NO_AUTH` | `false` | ⬜ | Set to `true` to allow unauthenticated access when no API key configured (local dev only — never in production) |
| `SECRETS_SOURCE` | `db` | ⬜ | `db` = secrets in DB columns (see `DB_SECRETS_KEY`); `alphakey` = fetch from alphaKey vault at runtime |
| `DB_SECRETS_KEY` | — | ⬜ | Fernet key for at-rest encryption of secret DB columns (API keys, SMTP password, webhook URLs). Generate: `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. **Strongly recommended in production.** Absent = plaintext storage. |
| `ALPHAKEY_USER_ID` | — | ⬜ | User ID for alphaKey vault lookups (required when `SECRETS_SOURCE=alphakey`) |
| `JWT_ISSUER` | `alphakey` | ⬜ | Expected `iss` claim in JWT; tokens with a different issuer are rejected |
| `JWT_AUDIENCE` | `alphakey` | ⬜ | Expected `aud` claim in JWT; tokens with a different audience are rejected |
| `REDIS__URL` | `redis://:password@redis:6379/0` | ⬜ | Redis for token denylist (password required — see `REDIS_PASSWORD` in alphaFrame) |
| `REDIS__ENABLED` | `true` | ⬜ | Enable Redis integration |
| `MINIO_ROOT_USER` | `minioadmin` | ✅ | MinIO credentials |
| `MINIO_ROOT_PASSWORD` | `minioadmin` | ✅ | MinIO credentials |
| `MINIO_BUCKET` | `models` | ✅ | Bucket for model artifacts |
| `MODEL_USER` | `default` | ⬜ | MinIO namespace user prefix |
| `MODEL_ACCOUNT` | `default` | ⬜ | MinIO namespace account prefix |
| `MODEL_SYNC_POLL_INTERVAL` | `60` | ⬜ | Seconds between ModelSyncDaemon polls |
| `MODEL_MAX_VERSIONS` | `5` | ⬜ | Max model versions to keep in local cache |
| `MODEL_VALIDATION_MIN_SHARPE` | `0.5` | ⬜ | Minimum Sharpe for accepting model |
| `MODEL_VALIDATION_MAX_DRAWDOWN` | `0.20` | ⬜ | Max drawdown for accepting model |
| `MODEL_VALIDATION_MIN_HIT_RATE` | `0.45` | ⬜ | Min hit rate for accepting model |
| `MLFLOW_TRACKING_URI` | `http://mlflow:5000` | ✅ | MLflow server URI |
| `ALERTS__SLACK__ENABLED` | `false` | ⬜ | Enable Slack alerts |
| `ALERTS__SLACK__WEBHOOK_URL` | — | ⬜ | Slack incoming webhook URL |
| `ALERTS__SLACK__MIN_LEVEL` | `WARNING` | ⬜ | Minimum level for Slack |
| `ALERTS__EMAIL__ENABLED` | `false` | ⬜ | Enable email alerts |
| `ALERTS__EMAIL__SMTP_HOST` | `smtp.gmail.com` | ⬜ | SMTP server host |
| `ALERTS__EMAIL__SMTP_PORT` | `587` | ⬜ | SMTP server port |
| `ALERTS__EMAIL__SMTP_USER` | — | ⬜ | SMTP login user |
| `ALERTS__EMAIL__SMTP_PASSWORD` | — | ⬜ | SMTP password |
| `ALERTS__EMAIL__FROM_ADDR` | — | ⬜ | Sender address |
| `ALERTS__EMAIL__TO_ADDRS` | `[]` | ⬜ | Recipient list (JSON array) |
| `ALERTS__EMAIL__MIN_LEVEL` | `WARNING` | ⬜ | Minimum level for email |

---

## overrides.yaml Structure

```yaml
defaults:
  size_pct: 0.10           # Fraction of account equity per BUY
  stop_loss_pct: 0.02      # SL: % below fill price
  take_profit_pct: 0.05    # TP: % above fill price
  cooldown_bars: 3         # Bars to wait after close before re-entry
  extended_hours: false    # Allow ticks outside NYSE regular hours

risk:
  max_positions: 5
  daily_loss_halt_pct: 0.05  # Halt new orders when daily PnL <= -5%
  consensus_min_confidence: 0.0  # Min winning-class probability (0 = disabled, e.g. 0.5 requires ≥50%)
  consensus_min_margin: 0.0      # Min lead over runner-up probability (0 = disabled, e.g. 0.1 requires 10pp margin)

models:
  <run_name>:              # Keyed by manifest run_name
    enabled: true
    t212_ticker: AAPL_US_EQ  # Optional T212 ticker (else auto-resolved)
    size_pct: 0.05           # Per-model override of defaults

alerts:
  slack:
    enabled: false
    webhook_url: ""
    min_level: WARNING
  email:
    enabled: false
    smtp_host: smtp.gmail.com
    smtp_port: 587
    smtp_user: ""
    smtp_password: ""
    from_addr: ""
    to_addrs: []
    min_level: CRITICAL

backtest:
  slippage_bps: 5
  commission_per_trade: 0.0
  initial_equity: 10000.0
  schedule_enabled: true
  cron: "0 2 * * *"     # Daily at 2am
  lookback_days: 30
```

> [!warning] YAML is seed only
> `overrides.yaml` is read-only at runtime. The bot never writes back to it. All changes to trading config after startup must use `PUT /settings` (global) or `PUT /models/{name}/overrides` (per-model) APIs. Changes are stored in DB and survive restarts.

---

## Position Sizing Modes

| Mode | Config | Formula |
|---|---|---|
| `fixed` | `size_pct` | `cash = equity × size_pct` |
| `atr` | `atr_risk_pct`, `atr_multiplier` | `dollar_risk = equity × atr_risk_pct; cash = (dollar_risk / (atr × atr_multiplier)) × price` |
| `vix` | `vix_scalar`, `vix_base_size_pct`, `vix_max_size_pct` | `multiplier = vix_scalar / current_vix; effective_pct = min(base × multiplier, max); cash = equity × effective_pct` |

---

## Portfolio Modes

| Mode | Behaviour |
|---|---|
| `balanced` | Cap total exposure per sector to `max_sector_pct` (e.g. 33%) |
| `unbalanced` | Cap positions per sector to `max_per_sector` (e.g. 3), with per-sector overrides |
