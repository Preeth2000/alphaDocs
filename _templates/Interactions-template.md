---
service: "{{SERVICE_NAME}}"
page: Interactions
tags:
  - service/{{service_slug}}
  - interactions
---

# {{SERVICE_NAME}} — Interactions

[[{{SERVICE_NAME}}]] · [[alphaDocs/services/alphaKey/Architecture]] · [[alphaDocs/services/alphaKey/API]] · [[alphaDocs/services/alphaKey/Data]] · [[alphaDocs/services/alphaKey/Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger | Handled by |
|---|---|---|---|---|
| [[services/UpstreamService]] | HTTP POST `/endpoint` | JSON | User action | `module.handler()` |
| User / Browser | HTTP GET `/endpoint` | — | On load | `api/routers/x.py` |
| Redis pub/sub | `channel:name` | JSON event | Event fired | `listener.py` |

---

## Outputs

| Destination | Mechanism | Format | Trigger | Sent from |
|---|---|---|---|---|
| [[services/DownstreamService]] | HTTP POST `/endpoint` | JSON | On event | `module.function()` |
| Redis pub/sub | `channel:name` | JSON event | On state change | `module.publish()` |
| File / Object store | MinIO PUT | Binary / JSON | On completion | `export.py` |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[services/alphaFrame]] | Redis, Postgres, MinIO, MLflow | Yes |
| [[services/alphaGen]] | model.ready events, model artifacts | Yes |

---

## Downstream Consumers

| Service | What it consumes from this service | Mechanism |
|---|---|---|
| [[services/alphaLink]] | REST API | HTTP |
| [[services/alphaTrade]] | model.ready SSE/events | SSE / Redis |

---

## Error / Edge Outputs

| Condition | Output | Destination |
|---|---|---|
| Validation failure | `422` response | Caller |
| Downstream unreachable | Webhook alert | [[services/alphaTrade]] notify module |
