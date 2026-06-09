---
service: alphaLink
page: Config
tags:
  - service/alphaLink
  - config
---

# alphaLink — Config

[[services/alphaLink/alphaLink|alphaLink]] · [[services/alphaLink/Architecture|Architecture]] · [[services/alphaLink/Interactions|Interactions]] · [[services/alphaLink/API|API]] · [[services/alphaLink/Data|Data]]

---

## Environment Variables

Source: `alphaLink/.env.local`

| Variable | Example Value | Required | Purpose |
|---|---|---|---|
| `ATT_BIN` | `/home/preeth/.pyenv/versions/3.11.14/bin/att` | ⬜ | Path to `att` CLI binary for config validation |
| `ALPHAGEN_API_URL` | `https://localhost/alphagen` | ✅ | alphaGen API base URL (via Nginx proxy) |
| `NODE_TLS_REJECT_UNAUTHORIZED` | `0` | ⬜ | Allow self-signed HTTPS (Nginx) in dev |
| `ALPHAGEN_ENV_FILE` | `/home/preeth/projects/alphaFrame/.env` | ⬜ | Path to alphaFrame `.env` (for Polygon key read/write) |
| `ALPHATRADE_API_URL` | `https://localhost/api/v1` | ✅ | alphaTrade API base URL (via Nginx proxy) |
| `ALPHAKEY_API_URL` | `http://alphakey-api:8000` | ✅ | alphaKey service URL (internal Docker network) |

> [!note] No client-side env vars
> All environment variables are server-side (Next.js BFF API routes). No `NEXT_PUBLIC_*` vars are exposed to the browser. Service URLs are never sent to the client.

---

## Next.js Config (`next.config.mjs`)

```js
export default {
  output: 'standalone'   // Minimal Docker image
}
```

No rewrites configured — all proxying handled by BFF API route handlers.

---

## Middleware (`src/middleware.ts`)

Guards all routes except public paths. Runs on every request.

**Public paths (no auth required):**
- `/login`, `/signup`
- `/api/auth/*`
- `/_next/*` (static assets)
- `/favicon.ico`

**All other paths:** Checks for `alphakey_session` httpOnly cookie. Missing → redirect to `/login`.

---

## Cookie: `alphakey_session`

| Property | Value |
|---|---|
| Name | `alphakey_session` |
| Contents | alphaKey refresh token (opaque, 96-char hex) |
| Flags | `httpOnly`, `Secure`, `SameSite=Strict` |
| TTL | 14 days (from alphaKey `REFRESH_TOKEN_TTL`) |
| Set by | `POST /api/auth/login` BFF handler |
| Cleared by | `POST /api/auth/logout` BFF handler |
| Read by | `POST /api/auth/refresh` BFF handler |

The access token (JWT, 10min) is stored in-memory only (Zustand `auth-store`). On page load, `AuthInitializer` calls `/api/auth/refresh` to restore it from the cookie.

---

## SessionConfig — Training Config Store

Persisted to localStorage as `alphagen.session`. Controls the multi-step config builder.

**Step groups:**
- **Step 0 (Data):** `tickers`, `interval`, `startDate`, `endDate`, `provider`, `includeVWAP`
- **Step 1 (Model):** `labelStrategy`, `features`, `indicators`, `modelArch`, `modelParams`
- **Step 2 (Train):** `epochs`, `batchSize`, `lr`, `optimizer`, `splits`, `backtest`

**Preserved across steps:** `tickers`, `startDate`, `endDate`, `provider`

Converted to RunConfig YAML via `src/lib/config-to-yaml.ts` before `POST /api/jobs`.

---

## No Override Chain

alphaLink has no runtime config overrides. All config is static `.env.local` + Next.js build-time constants.
