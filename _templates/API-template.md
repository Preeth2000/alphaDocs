---
service: "{{SERVICE_NAME}}"
page: API
tags:
  - service/{{service_slug}}
  - api
---

# {{SERVICE_NAME}} — API

[[{{SERVICE_NAME}}]] · [[Architecture]] · [[Interactions]] · [[Data]] · [[Config]]

---

## Inbound Endpoints

> Base URL: `http://{{host}}:{{PORT}}`

| Method | Path | Purpose | Caller(s) | Auth | DB Reads | DB Writes | Notes |
|---|---|---|---|---|---|---|---|
| `GET` | `/health` | Health check | [[services/alphaFrame]] Nginx, CI | None | 0 | 0 | |
| `POST` | `/endpoint` | … | [[services/alphaLink]] | JWT | 1 | 1 | |

### Request / Response Detail

#### `POST /endpoint`

**Request body:**
```json
{
  "field": "value"
}
```

**Response `200`:**
```json
{
  "id": "uuid",
  "status": "created"
}
```

---

## Outbound Calls

> Calls this service makes to other services.

| Target Service | Method | Endpoint | Trigger | Notes |
|---|---|---|---|---|
| [[services/alphaGen]] | `GET` | `/runs/events` (SSE) | On startup | Listen for model.ready |
| [[services/alphaKey]] | `GET` | `/auth/verify` | Per request | Token validation |

---

## SSE Streams (if applicable)

| Endpoint | Channel | Events emitted | Consumer |
|---|---|---|---|
| `GET /runs/{id}/log` | `run:{id}:log` | `log`, `status`, `done` | [[services/alphaLink]] |
| `GET /runs/events` | `model.ready` | `model.ready` | [[services/alphaTrade]] |

---

## WebSocket Streams (if applicable)

*None / describe if present.*

---

*See [[reference/Ports-and-Endpoints]] for the global port map.*
