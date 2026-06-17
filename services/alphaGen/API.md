---
service: alphaGen
page: API
tags:
  - service/alphaGen
  - api
---

# alphaGen — API

[[alphaGen|alphaGen]] · [[alphaDocs/services/alphaGen/Architecture|Architecture]] · [[alphaDocs/services/alphaGen/Interactions|Interactions]] · [[alphaDocs/services/alphaGen/Data|Data]] · [[alphaDocs/services/alphaGen/Config|Config]]

---

## Inbound Endpoints

**Base URL:** `http://alphagen-api:8000` (internal) | `https://localhost/alphagen/` (via Nginx)  
**Auth:** Bearer JWT required on all `/runs` and `/runs/{id}/*` endpoints. Missing or invalid token → `401`. Valid token for wrong owner → `403`. JWKS fetched from [[alphaKey|alphaKey]] `/auth/.well-known/jwks.json` (cached 10 min). `/health`, `/config/*`, `/models/*`, `/runs/events` are unauthenticated.

### Run Management

| Method | Path | Purpose | Caller | DB Reads | DB Writes | Auth |
|---|---|---|---|---|---|---|
| `POST` | `/runs` | Create training job, enqueue Celery task | [[alphaLink\|alphaLink]] | 0 | 1 (INSERT Run) | 🔒 JWT |
| `GET` | `/runs` | List runs scoped to caller's `user_id` (filter by status, run_name, limit, offset) | [[alphaLink\|alphaLink]] | 1 (SELECT filtered) | 0 | 🔒 JWT |
| `GET` | `/runs/{run_id}` | Get full run state (owner only) | [[alphaLink\|alphaLink]] | 1 (SELECT by id) | 0 | 🔒 JWT + 403 |
| `DELETE` | `/runs/{run_id}` | Cancel: revoke Celery task, publish cancelled event (owner only) | [[alphaLink\|alphaLink]] | 1 | 1 (UPDATE status→cancelled) | 🔒 JWT + 403 |
| `POST` | `/runs/{run_id}/force-save` | Re-queue gate-failed run with `force_save=True` (owner only) | [[alphaLink\|alphaLink]] | 1 | 2 (INSERT new run, UPDATE original) | 🔒 JWT + 403 |
| `POST` | `/runs/{run_id}/publish` | Push artifacts to MinIO, fire model.ready (owner only) | [[alphaLink\|alphaLink]] | 1 | 1 (UPDATE run: minio_version, artifact_prefix) | 🔒 JWT + 403 |
| `GET` | `/runs/{run_id}/log` | **SSE** — stream log lines + status events (owner only) | [[alphaLink\|alphaLink]] | 1 (SELECT run state) | 0 | 🔒 JWT + 403 |
| `GET` | `/runs/events` | **SSE** — global model.ready channel | [[alphaTrade\|alphaTrade]] | 0 | 0 | Open |

### Config

| Method | Path | Purpose | Caller | DB Reads | DB Writes |
|---|---|---|---|---|---|
| `GET` | `/config/validation` | Get validation gate thresholds | [[alphaLink\|alphaLink]] | 1 (SELECT id=1) | 0 |
| `PATCH` | `/config/validation` | Update validation gate thresholds | [[alphaLink\|alphaLink]] | 1 | 1 (UPDATE id=1) |

### Models (MinIO-backed)

| Method | Path | Purpose | Caller | DB Reads | DB Writes |
|---|---|---|---|---|---|
| `GET` | `/models` | List published models from MinIO (by user/account prefix) | [[alphaLink\|alphaLink]] | 0 | 0 |
| `GET` | `/models/{run_name}` | Get latest version for model from MinIO | [[alphaLink\|alphaLink]] | 0 | 0 |
| `POST` | `/models/{run_name}/promote` | Set model version alias to "production" in MLflow | [[alphaLink\|alphaLink]] | 0 | 0 (MLflow write) |

### Health

| Method | Path | Purpose | Caller | DB Reads | DB Writes |
|---|---|---|---|---|---|
| `GET` | `/health` | Check db ping + worker connectivity | CI, [[alphaFrame\|Nginx]] | 1 (SELECT 1) | 0 |

---

## Request / Response Detail

### `POST /runs`

**Request:**
```json
{
  "config_yaml": "data:\n  ticker: AAPL\n  interval: 1d\n...",
  "visibility": "private"
}
```

**Response `202`:**
```json
{
  "id": "run_abc123",
  "run_name": "aapl_daily_mlp",
  "status": "queued",
  "created_at": "2026-06-06T10:00:00Z",
  "user_id": "uuid..."
}
```

### `GET /runs/{id}/log` — SSE events

```
event: data
data: {"type": "log", "line": "Epoch 3/50 — loss=0.043", "ts": "2026-06-06T10:01:00Z"}

event: data
data: {"type": "status", "status": "gate_passed", "ts": "2026-06-06T10:05:00Z"}

event: data
data: {"done": true}
```

### `GET /runs/events` — SSE model.ready

```
event: data
data: {"run_name": "aapl_daily_mlp", "version": "v3", "artifact_prefix": "prod/isa/aapl_daily_mlp/v3", "published_at": "2026-06-06T10:06:00Z"}
```

---

## SSE Streams

| Endpoint | Source | Events | Consumer |
|---|---|---|---|
| `GET /runs/{id}/log` | Redis `run:{id}:log` (live) or log file replay (terminal) | `log`, `status`, `done` | [[alphaLink\|alphaLink]] BFF → browser |
| `GET /runs/events` | Redis `model.ready` | `model.ready` | [[alphaTrade\|alphaTrade]] |

**SSE behaviour for terminal runs:** If run is already `complete/failed/cancelled` when `GET /runs/{id}/log` is called, replays the log file then sends a synthesized final status event and closes.

---

## Outbound Calls

| Target | Method | Endpoint | Trigger | Auth |
|---|---|---|---|---|
| yfinance (public) | HTTP GET | Yahoo Finance API | Inside Celery task (data fetch) | None |
| Polygon.io | HTTP GET | `/v2/aggs/ticker/{ticker}/range/...` | When `SECRETS_SOURCE` provides `POLYGON_API_KEY` | Bearer token |
| [[alphaKey\|alphaKey]] | `GET` | `/auth/.well-known/jwks.json` | JWT verification (cached 5min) | None |
| [[alphaKey\|alphaKey]] | `GET` | `/auth/internal/secrets/{user_id}` | When `SECRETS_SOURCE=alphakey` | `X-Service-Token` header |
| MinIO S3 | PUT | `/{bucket}/{prefix}/model.onnx` etc. | POST /runs/{id}/publish | AWS credentials |
| MLflow | SDK | Various | After training success | None (internal) |
| OTel Collector | OTLP gRPC | `:4317` | Always (async) | None |

---

*See [[reference/Ports-and-Endpoints]] for host/port details.*
