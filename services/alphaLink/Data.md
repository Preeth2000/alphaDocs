---
service: alphaLink
page: Data
tags:
  - service/alphaLink
  - data
---

# alphaLink — Data

[[alphaLink|alphaLink]] · [[alphaDocs/services/alphaLink/Architecture|Architecture]] · [[alphaDocs/services/alphaLink/Interactions|Interactions]] · [[alphaDocs/services/alphaLink/API|API]] · [[alphaDocs/services/alphaLink/Config|Config]]

---

> [!note] No database
> alphaLink has no database. All persistent state lives in the backends it proxies. alphaLink's only local storage is the template files and artifact cache on the host filesystem.

---

## Local Filesystem Storage

| Path | Contents | Written by | Read by |
|---|---|---|---|
| `~/.alphalink/templates/*.json` | Saved RunConfig templates | `POST /api/templates` | `GET /api/templates` |
| `~/.alphalink/artifacts/{jobId}/model.onnx` | Downloaded ONNX model | `GET /api/jobs/[id]/artifacts/model.onnx` | Browser download |
| `~/.alphalink/artifacts/{jobId}/manifest.json` | Downloaded manifest | Same | Browser download |

---

## Client-Side State

### Zustand Stores (in-memory + localStorage)

| Store | Persistence | Contents |
|---|---|---|
| `auth-store` | localStorage (user only) | `user`, `role`, `status`. `accessToken` is **NOT** persisted. |
| `alphagen.session` | localStorage | Full `SessionConfig` (training config builder state) |

### React Query Cache (in-memory)

Server data cached per-query key. Invalidated on mutations.

| Data | Query key | Source |
|---|---|---|
| Training runs | `['runs']` | `GET /api/jobs` |
| Single run | `['run', id]` | `GET /api/jobs/[id]` |
| Positions | `['positions']` | `GET /api/trade/[...]/positions` |
| Orders | `['orders']` | `GET /api/trade/[...]/orders` |
| Models | `['models']` | `GET /api/trade/[...]/models` |
| P&L | `['pnl']` | `GET /api/trade/[...]/pnl` |

---

## Data Read/Write Count per BFF Call

| BFF endpoint | Reads from upstream | Writes to upstream |
|---|---|---|
| `POST /api/auth/login` | 1 (GET /auth/me after login) | 1 (POST /auth/login) |
| `GET /api/jobs` | 1 | 0 |
| `POST /api/jobs` | 0 | 1 |
| `GET /api/jobs/[id]/logs` | 1 (run state check) | 0 |
| `GET /api/templates` | 0 | 0 (local FS read) |
| `POST /api/templates` | 0 | 0 (local FS write) |
| `GET /api/trade/[...]/positions` | 1 | 0 |
| `GET /api/trade/events` (SSE) | 0 | 0 |

All data ultimately lives in [[alphaDocs/services/alphaGen/Data|alphaGen Data]], [[alphaDocs/services/alphaTrade/Data|alphaTrade Data]], or [[alphaDocs/services/alphaKey/Data|alphaKey Data]].
