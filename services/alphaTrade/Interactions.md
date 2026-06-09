---
service: alphaTrade
page: Interactions
tags:
  - service/alphaTrade
  - interactions
---

# alphaTrade — Interactions

[[services/alphaTrade/alphaTrade|alphaTrade]] · [[services/alphaTrade/Architecture|Architecture]] · [[services/alphaTrade/API|API]] · [[services/alphaTrade/Data|Data]] · [[services/alphaTrade/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger | Handled by |
|---|---|---|---|---|
| [[services/alphaGen/alphaGen\|alphaGen]] | MinIO S3 download | `model.onnx`, `manifest.json` | ModelSyncDaemon poll (every 60s) | `adapter/model_sync.py` |
| [[services/alphaFrame/alphaFrame\|MLflow]] | SDK `search_registered_models` | Model registry metadata | ModelSyncDaemon poll | `adapter/model_sync.py` |
| [[services/alphaGen/alphaGen\|alphaGen]] | SSE `GET /runs/events` | `model.ready` JSON event | On alphaGen publish | `runs.py run_events()` → triggers sync |
| Trading 212 API | HTTP GET | OHLCV JSON | Per bar-close tick | `broker/t212_client.py` |
| yfinance | HTTP | OHLCV DataFrame | Per bar-close tick | `data/yfinance_provider.py` |
| Polygon.io | HTTP | OHLCV JSON | Per bar-close tick (if enabled) | `data/polygon_provider.py` |
| Trading 212 API | HTTP GET | Order fills, position state | OCO monitor poll (every 10s) | `broker/oco_monitor.py` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | REST `GET/POST/PUT/PATCH/DELETE /api/v1/*` | JSON | User action in UI | `api/routers/*.py` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `GET /api/v1/stream` SSE | — | Dashboard connection | `api/stream_bus.py` |
| [[services/alphaKey/alphaKey\|alphaKey]] | `POST /auth/introspect` | JWT introspection | Per authenticated API call | `security/deps.py` |
| `overrides.yaml` | File read | YAML | On startup | `config.py _load_overrides()` |
| SQLite/Postgres | SQL | DB records | Per tick + API calls | All repos |

---

## Outputs

| Destination | Mechanism | Format | Trigger | Sent from |
|---|---|---|---|---|
| Trading 212 API | HTTP POST | Market/stop/limit order JSON | Gate approved → BUY or SELL | `broker/t212_client.py place_market_order()` |
| Trading 212 API | HTTP DELETE | Cancel order | OCO leg filled | `broker/t212_client.py cancel_order()` |
| [[services/alphaLink/alphaLink\|alphaLink]] | REST responses | JSON | Per API call | All routers |
| [[services/alphaLink/alphaLink\|alphaLink]] | SSE `GET /stream` | `{data: JSON}` events | Tick results, fills, halts | `api/stream_bus.py` |
| Slack | HTTP POST webhook | `{text: "..."}` | Alert events | `notify/alerting.py` |
| Email (SMTP) | SMTP TLS | Text/HTML | Alert events | `notify/alerting.py` |
| Custom webhook | HTTP POST | `{content: "..."}` | Alert events | `notify/webhook.py` |
| MinIO `trades` bucket | S3 PUT | Trade logs / reports | Future use | Planned |
| OTel Collector | OTLP gRPC | Traces + metrics | Always | `telemetry` |
| Prometheus `:9090` | Pull endpoint | Prometheus metrics | Scraped every 15s | `prometheus_client` |

---

## Alert Events (notify module)

| Event | Level | Destination |
|---|---|---|
| Order filled | INFO | Webhook/Slack if level ≥ INFO |
| Order error / rejection | ERROR | All enabled channels |
| OCO leg executed (SL/TP) | INFO | Webhook/Slack |
| Daily loss halt triggered | CRITICAL | All enabled channels |
| Model retired automatically | WARNING | Webhook/Slack |
| Kill switch toggled | CRITICAL | All enabled channels |
| Health degraded (tick stale, T212 unreachable) | WARNING | Webhook/Slack |

Rate limit: 30s per-category (e.g. "fills", "oco") — prevents flood on rapid events.

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Postgres | All 23 tables | ✅ (or SQLite fallback) |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Redis | Token denylist | ⬜ (when AUTH_MODE=alphakey) |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → MinIO | Model artifacts | ✅ |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → MLflow | Model registry | ✅ |
| [[services/alphaGen/alphaGen\|alphaGen]] | Published models (ONNX + manifest) | ✅ (no models → no trading) |
| [[services/alphaKey/alphaKey\|alphaKey]] | JWT token introspection | ⬜ (when AUTH_MODE=alphakey) |
| Trading 212 | Order execution, account data | ✅ (trading halts if unreachable) |
| yfinance / Polygon | OHLCV price data | ✅ |

---

## Downstream Consumers

| Service | What it consumes | Mechanism |
|---|---|---|
| [[services/alphaLink/alphaLink\|alphaLink]] | All trading state (positions, orders, signals, P&L, etc.) | REST API + SSE `/stream` |
| Prometheus / Grafana | Metrics (signals, orders, P&L, latency) | Prometheus scrape port 9090 |
