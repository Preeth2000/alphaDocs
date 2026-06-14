---
service: alphaLink
page: API
tags:
  - service/alphaLink
  - api
---

# alphaLink — API

[[services/alphaLink/alphaLink|alphaLink]] · [[services/alphaLink/Architecture|Architecture]] · [[services/alphaLink/Interactions|Interactions]] · [[services/alphaLink/Data|Data]] · [[services/alphaLink/Config|Config]]

---

## BFF Route Handlers (`src/app/api/`)

> All routes are Next.js Route Handlers (App Router). Base: `http://localhost:3000`  
> All `/api/trade/*` and `/api/jobs/*` routes inject `Authorization: Bearer <access_token>`.

---

### Auth Routes (`/api/auth/`)

Proxies to [[services/alphaKey/alphaKey|alphaKey]] `http://alphakey-api:8000`

| Method | Path | Proxies to | Special handling |
|---|---|---|---|
| `POST` | `/api/auth/login` | `POST /auth/login` | Extract refresh_token → httpOnly cookie; fetch /auth/me; return `{access_token, user}`. Forwards `X-TOTP-Required: true` header when alphaKey signals step-up needed. Accepts optional `totp_code` in body. |
| `POST` | `/api/auth/logout` | `POST /auth/logout` | Clear `alphakey_session` cookie; body includes `{jti, exp, refresh_token}` |
| `POST` | `/api/auth/refresh` | `POST /auth/refresh` | Read httpOnly cookie; inject into body; return new `access_token`. On upstream 401/403: clear both cookies so middleware redirects to `/login`. |
| `POST` | `/api/auth/register` | `POST /auth/register` | Transparent proxy |
| `POST` | `/api/auth/forgot-password` | `POST /auth/forgot-password` | Transparent proxy; handles 501 (SMTP not configured) |
| `POST` | `/api/auth/reset-password` | `POST /auth/reset-password` | Transparent proxy; on success client redirects to `/login` (no auto-login — alphaKey revokes all sessions) |
| `GET` | `/api/auth/me/sessions` | `GET /auth/me/sessions` | Bearer-forwarded; returns active session list |
| `DELETE` | `/api/auth/me/sessions/{id}` | `DELETE /auth/me/sessions/{id}` | Bearer-forwarded; revoke session. Revoking current session → client redirects to `/login?reason=session_expired` |
| `POST` | `/api/auth/me/totp/setup` | `POST /auth/me/totp/setup` | Bearer-forwarded; returns `{qr_uri, secret}` |
| `POST` | `/api/auth/me/totp/verify` | `POST /auth/me/totp/verify` | Bearer-forwarded; activates TOTP |
| `POST` | `/api/auth/me/totp/disable` | `POST /auth/me/totp/disable` | Bearer-forwarded; requires current TOTP code |
| ALL | `/api/auth/[...path]` | `/auth/[...path]` | Transparent proxy (preserves Authorization header, merges refresh_token) |

---

### Training Jobs (`/api/jobs/`)

Proxies to [[services/alphaGen/alphaGen|alphaGen]] `ALPHAGEN_API_URL`

| Method | Path | Proxies to | Notes |
|---|---|---|---|
| `POST` | `/api/jobs` | `POST /runs` | Normalises run → Job type; adds `visibility` |
| `GET` | `/api/jobs` | `GET /runs` | Normalises list |
| `GET` | `/api/jobs/[id]` | `GET /runs/{id}` | Normalises run |
| `DELETE` | `/api/jobs/[id]` | `DELETE /runs/{id}` | Cancel job |
| `GET` | `/api/jobs/[id]/logs` | `GET /runs/{id}/log` (SSE) | Forwards `log`/`status` events; closes on `done: true` |
| `GET` | `/api/jobs/[id]/results` | `GET /runs/{id}` | Extracts metrics fields |
| `GET` | `/api/jobs/[id]/manifest` | `GET /models/{run_name}` | Model metadata |
| `GET` | `/api/jobs/[id]/artifacts/[filename]` | Local `~/.alphalink/artifacts/{id}/{filename}` | Download model.onnx / manifest.json |
| `POST` | `/api/jobs/[id]/force-save` | `POST /runs/{id}/force-save` | Re-train bypassing gate |
| `POST` | `/api/jobs/validate` | Local — runs `att validate --json` CLI | Returns `{valid: bool, errors: []}` |

---

### Trade Proxy (`/api/trade/`)

Proxies to [[services/alphaTrade/alphaTrade|alphaTrade]] `ALPHATRADE_API_URL` (`https://localhost/api/v1`)

| Method | Path | Proxies to | Notes |
|---|---|---|---|
| ALL | `/api/trade/[...path]` | `alphatrade /api/v1/[...path]` | Extracts Bearer token, forwards it |
| `GET` | `/api/trade/events` | `alphatrade /stream` (SSE) | Error handling; reconnect on disconnect |
| `GET` | `/api/trade/health/[...path]` | `alphatrade :8080/[...path]` | Health check proxy (no API key needed) |

---

### Config (`/api/config/`)

| Method | Path | Proxies to | Notes |
|---|---|---|---|
| `GET` | `/api/config/validation` | `GET /config/validation` (alphaGen) | Validation gate thresholds |
| `PATCH` | `/api/config/validation` | `PATCH /config/validation` (alphaGen) | Update gate thresholds |

---

### Templates (`/api/templates/`)

Local filesystem — no backend call.

| Method | Path | Storage | Notes |
|---|---|---|---|
| `GET` | `/api/templates` | `~/.alphalink/templates/*.json` | List saved templates |
| `POST` | `/api/templates` | `~/.alphalink/templates/{name}.json` | Save new template |
| `DELETE` | `/api/templates/[id]` | Filesystem delete | Remove template |
| `PATCH` | `/api/templates/[id]` | Filesystem rename | Rename template |

---

### Utilities

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | BFF liveness check — returns `{ok: true}` |
| `GET` | `/api/instruments?q=` | Ticker symbol search (via `lib/instruments.ts`) |
| `GET` | `/api/polygon-key` | Read `polygon/default/api_key` from alphaKey vault; returns `{status:'valid', hint:'****xxxx'}` or `{status:'missing'}` |
| `POST` | `/api/polygon-key` | Write `{value}` to alphaKey vault `polygon/default/api_key` via `PUT /auth/vault/polygon/default/api_key` |
| `GET` | `/api/fs/browse?path=` | Directory listing; requires auth; restricted to `$HOME` (or `FS_BROWSE_ROOT` env var) |
| `GET` | `/api/runs` | Legacy run summary endpoint |

---

## SSE Streams (BFF forwards)

| BFF Endpoint | Upstream | Events forwarded | Consumer |
|---|---|---|---|
| `GET /api/jobs/[id]/logs` | alphaGen `GET /runs/{id}/log` | `log`, `status`, closes on `done: true` | Training page — live log viewer |
| `GET /api/trade/events` | alphaTrade `GET /stream` | All alphaTrade SSE events | Dashboard — live positions/orders/fills |
| `GET /api/events/model-ready` | alphaGen `GET /runs/events` | Model-ready notifications | Trade models page |

All SSE proxies wire an `AbortController` to `req.signal`: when the browser disconnects the upstream fetch is cancelled, preventing orphaned server-side connections.

---

## Outbound Calls (server-side)

| Target | How | When |
|---|---|---|
| [[services/alphaKey/alphaKey\|alphaKey]] `:8000` | HTTP via fetch | All `/api/auth/*` calls; `/api/polygon-key` vault read/write |
| [[services/alphaGen/alphaGen\|alphaGen]] `ALPHAGEN_API_URL` | HTTP via fetch | All `/api/jobs/*` + `/api/config/*` (with `Authorization: Bearer` forwarded) |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] `ALPHATRADE_API_URL` | HTTP via fetch | All `/api/trade/*` |
| `att` binary (`ATT_BIN`) | Child process | `POST /api/jobs/validate` |
| Local filesystem `~/.alphalink/` | `fs` module | Templates read/write, artifacts |

---

*See [[reference/Ports-and-Endpoints]] for URL configuration.*
