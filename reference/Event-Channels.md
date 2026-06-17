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

Hosted by [[alphaFrame|alphaFrame]] Redis instance.

| Channel | Pattern | Producer | Consumer(s) | Payload | Purpose |
|---|---|---|---|---|---|
| `run:{id}:log` | Per-run | [[alphaGen\|alphaGen]] Celery worker | [[alphaGen\|alphaGen]] API (`GET /runs/{id}/log`) | `{"type": "log"|"status"|"done", "text": "…", "status": "…"}` | Stream live training log lines + status transitions to SSE subscribers |
| `model.ready` | Global | [[alphaGen\|alphaGen]] Celery worker (auto-promote) or `/runs/{id}/publish` endpoint | [[alphaTrade\|alphaTrade]], [[alphaLink\|alphaLink]] | See schema below | Notify when a new model is ready for consumption |

---

## SSE Endpoints

SSE streams exposed by [[alphaGen|alphaGen]] (`GET` endpoints that hold the connection open):

| Endpoint | Source channel | Emits to | Events |
|---|---|---|---|
| `GET /runs/{id}/log` | Redis `run:{id}:log` | [[alphaLink\|alphaLink]] BFF → browser | `log`, `status`, `done` |
| `GET /runs/events` | Redis `model.ready` | [[alphaTrade\|alphaTrade]] on startup | `model.ready` |

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

Emitted by two paths — consumers **must** tolerate both:

| Path | `artifact_prefix` present? |
|------|---------------------------|
| Celery worker auto-promote (gate passed) | No |
| `POST /runs/{id}/publish` (explicit MinIO push) | Yes |

```json
{
  "run_name": "aapl_daily_mlp",
  "version": "3",
  "published_at": "2026-06-14T10:00:00+00:00",
  "artifact_prefix": "user/account/aapl_daily_mlp/v3"
}
```

- `run_name` — MLflow registered model name
- `version` — MLflow model version string
- `published_at` — ISO 8601 with timezone
- `artifact_prefix` *(optional)* — S3/MinIO object key prefix; only present when published via `/runs/{id}/publish`

Contract tests: `tests/contract/test_model_ready_schema.py`

---

## Celery Task Queue (Redis)

| Redis DB | Purpose |
|---|---|
| `db 0` | Celery broker — task routing |
| `db 1` | Celery result backend — task state + results |

Producer: [[alphaGen|alphaGen]] FastAPI (`POST /runs` enqueues `att.train_and_export`)  
Consumer: [[alphaGen|alphaGen]] Celery worker

---

## Webhook Alerts

[[alphaTrade|alphaTrade]] notify module fires outbound HTTP webhooks on trading events. See [[alphaDocs/services/alphaTrade/Config]] for `WEBHOOK_URL` env var.

| Event | Trigger |
|---|---|
| Order placed | Broker confirms fill |
| Daily loss halt triggered | PnL crosses halt threshold |
| Model loaded | New model.ready received and model hot-swapped |
| Error / exception | Unhandled exception in scheduler tick |

---

*Update this page when new channels, SSE streams, or webhook events are added.*
