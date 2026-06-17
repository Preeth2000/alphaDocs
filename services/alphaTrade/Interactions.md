---
service: alphaTrade
page: Interactions
tags:
  - service/alphaTrade
  - interactions
---

# alphaTrade — Interactions

[[alphaTrade|alphaTrade]] · [[alphaDocs/services/alphaTrade/Architecture|Architecture]] · [[alphaDocs/services/alphaTrade/API|API]] · [[alphaDocs/services/alphaTrade/Data|Data]] · [[alphaDocs/services/alphaTrade/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger | Handled by |
|---|---|---|---|---|
| [[alphaGen\|alphaGen]] | MinIO S3 download | `model.onnx`, `manifest.json` | ModelSyncDaemon poll (every 60s) | `adapter/model_sync.py` |
| [[alphaFrame\|MLflow]] | SDK `search_registered_models` | Model registry metadata | ModelSyncDaemon poll | `adapter/model_sync.py` |
| [[alphaGen\|alphaGen]] | Redis pub/sub `model.ready` | `{run_name, version, published_at, artifact_prefix?}` | On alphaGen publish | `store/model_sync.py ModelSyncDaemon._listen_model_ready()` — artifact_prefix present → MinIO direct; absent → MLflow poll |
| Trading 212 API | HTTP GET | OHLCV JSON | Per bar-close tick | `broker/t212_client.py` |
| yfinance | HTTP | OHLCV DataFrame | Per bar-close tick | `data/yfinance_provider.py` |
| Polygon.io | HTTP | OHLCV JSON | Per bar-close tick (if enabled) | `data/polygon_provider.py` |
| Trading 212 API | HTTP GET | Order fills, position state | OCO monitor poll (every 10s) | `broker/oco_monitor.py` |
| [[alphaLink\|alphaLink]] BFF | REST `GET/POST/PUT/PATCH/DELETE /api/v1/*` | JSON | User action in UI | `api/routers/*.py` |
| [[alphaLink\|alphaLink]] BFF | `GET /api/v1/stream` SSE | — | Dashboard connection | `api/stream_bus.py` |
| [[alphaKey\|alphaKey]] | `POST /auth/introspect` | JWT introspection | Per authenticated API call | `security/deps.py` |
| `overrides.yaml` | File read | YAML | On startup | `config.py _load_overrides()` |
| SQLite/Postgres | SQL | DB records | Per tick + API calls | All repos |

---

## Outputs

| Destination | Mechanism | Format | Trigger | Sent from |
|---|---|---|---|---|
| Trading 212 API | HTTP POST | Market/stop/limit order JSON | Gate approved → BUY or SELL | `broker/t212_client.py place_market_order()` |
| Trading 212 API | HTTP DELETE | Cancel order | OCO leg filled | `broker/t212_client.py cancel_order()` |
| [[alphaLink\|alphaLink]] | REST responses | JSON | Per API call | All routers |
| [[alphaLink\|alphaLink]] | SSE `GET /stream` | `{data: JSON}` events | Tick results, fills, halts | `api/stream_bus.py` |
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
| [[alphaFrame\|alphaFrame]] → Postgres | All 23 tables | ✅ (or SQLite fallback) |
| [[alphaFrame\|alphaFrame]] → Redis | Token denylist | ⬜ (when AUTH_MODE=alphakey) |
| [[alphaFrame\|alphaFrame]] → MinIO | Model artifacts | ✅ |
| [[alphaFrame\|alphaFrame]] → MLflow | Model registry | ✅ |
| [[alphaGen\|alphaGen]] | Published models (ONNX + manifest) | ✅ (no models → no trading) |
| [[alphaKey\|alphaKey]] | JWT token introspection | ⬜ (when AUTH_MODE=alphakey) |
| Trading 212 | Order execution, account data | ✅ (trading halts if unreachable) |
| yfinance / Polygon | OHLCV price data | ✅ |

---

## Downstream Consumers

| Service | What it consumes | Mechanism |
|---|---|---|
| [[alphaLink\|alphaLink]] | All trading state (positions, orders, signals, P&L, etc.) | REST API + SSE `/stream` |
| Prometheus / Grafana | Metrics (signals, orders, P&L, latency) | Prometheus scrape port 9090 |
