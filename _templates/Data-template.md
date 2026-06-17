---
service: "{{SERVICE_NAME}}"
page: Data
tags:
  - service/{{service_slug}}
  - data
---

# {{SERVICE_NAME}} — Data

[[{{SERVICE_NAME}}]] · [[alphaDocs/services/alphaKey/Architecture]] · [[alphaDocs/services/alphaKey/Interactions]] · [[alphaDocs/services/alphaKey/API]] · [[alphaDocs/services/alphaKey/Config]]

---

## Datastores Used

| Store | Type | Purpose | Managed by |
|---|---|---|---|
| `{{db_name}}` | SQLite / Postgres / Redis | Primary state | Alembic migrations |
| MinIO | Object store | Artifact storage | Manual / SDK |
| Redis | In-memory | Pub/sub, caching | [[services/alphaFrame]] |

---

## Database Tables

| Table | Purpose | Key columns |
|---|---|---|
| `table_name` | Stores … | `id`, `status`, `created_at` |

---

## DB Operations — Read/Write Count per API Call

| Operation (endpoint/action) | Reads | Writes | Tables Touched |
|---|---|---|---|
| `POST /endpoint` — create | 0 | 1 | `table_name` |
| `GET /endpoint` — list | 1 | 0 | `table_name` |
| `GET /endpoint/{id}` — fetch | 1 | 0 | `table_name` |
| `DELETE /endpoint/{id}` | 1 | 1 | `table_name` |

---

## Migrations

| Location | Tool | Notes |
|---|---|---|
| `{{src}}/store/migrations/versions/` | Alembic | Run via `alembic upgrade head` |

### Migration History (key milestones)

| Migration | Description |
|---|---|
| `0001_initial` | Create `table_name` |

---

## Object Store (MinIO)

| Bucket | Contents | Written by | Read by |
|---|---|---|---|
| `models` | `model.onnx`, `manifest.json` | [[services/alphaGen]] | [[services/alphaTrade]] |

---

## Redis Usage

| Key pattern | Type | Purpose | TTL |
|---|---|---|---|
| `run:{id}:log` | pub/sub | Log stream per run | — |
| `model.ready` | pub/sub | Model publish events | — |
