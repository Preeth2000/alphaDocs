---
page: Event-Channels
tags:
  - reference
  - events
  - redis
  - sse
---

# Event Channels

[[README]] · [[reference/Glossary]] · [[reference/Ports-and-Endpoints]]

All asynchronous event mechanisms used across projectAlpha — Redis pub/sub channels and SSE streams.

---

## Redis Pub/Sub Channels

Hosted by [[services/alphaFrame/alphaFrame|alphaFrame]] Redis instance.

| Channel | Pattern | Producer | Consumer(s) | Payload | Purpose |
|---|---|---|---|---|---|
| `run:{id}:log` | Per-run | [[services/alphaGen/alphaGen\|alphaGen]] Celery worker | [[services/alphaGen/alphaGen\|alphaGen]] API (`GET /runs/{id}/log`) | `{"type": "log"|"status"|"done", "text": "…", "status": "…"}` | Stream live training log lines + status transitions to SSE subscribers |
| `model.ready` | Global | [[services/alphaGen/alphaGen\|alphaGen]] publish endpoint | [[services/alphaTrade/alphaTrade\|alphaTrade]] | `{"run_name": "…", "version": "…", "bucket": "…"}` | Notify when a new model is published to MinIO |

---

## SSE Endpoints

SSE streams exposed by [[services/alphaGen/alphaGen|alphaGen]] (`GET` endpoints that hold the connection open):

| Endpoint | Source channel | Emits to | Events |
|---|---|---|---|
| `GET /runs/{id}/log` | Redis `run:{id}:log` | [[services/alphaLink/alphaLink\|alphaLink]] BFF → browser | `log`, `status`, `done` |
| `GET /runs/events` | Redis `model.ready` | [[services/alphaTrade/alphaTrade\|alphaTrade]] on startup | `model.ready` |

### SSE Event Formats

#### `run:{id}:log` events

```json
// Log line
{ "type": "log", "text": "Epoch 3/50 — loss=0.043" }

// Status transition
{ "type": "status", "status": "gate_passed" }

// Terminal — stream closes
{ "type": "done", "status": "published" }
```

#### `model.ready` event

```json
{
  "type": "model.ready",
  "run_name": "aapl_daily_mlp",
  "version": "3",
  "minio_bucket": "models",
  "minio_path": "aapl_daily_mlp/v3/model.onnx",
  "manifest_path": "aapl_daily_mlp/v3/manifest.json"
}
```

---

## Celery Task Queue (Redis)

| Redis DB | Purpose |
|---|---|
| `db 0` | Celery broker — task routing |
| `db 1` | Celery result backend — task state + results |

Producer: [[services/alphaGen/alphaGen|alphaGen]] FastAPI (`POST /runs` enqueues `att.train_and_export`)  
Consumer: [[services/alphaGen/alphaGen|alphaGen]] Celery worker

---

## Webhook Alerts

[[services/alphaTrade/alphaTrade|alphaTrade]] notify module fires outbound HTTP webhooks on trading events. See [[services/alphaTrade/Config]] for `WEBHOOK_URL` env var.

| Event | Trigger |
|---|---|
| Order placed | Broker confirms fill |
| Daily loss halt triggered | PnL crosses halt threshold |
| Model loaded | New model.ready received and model hot-swapped |
| Error / exception | Unhandled exception in scheduler tick |

---

*Update this page when new channels, SSE streams, or webhook events are added.*
