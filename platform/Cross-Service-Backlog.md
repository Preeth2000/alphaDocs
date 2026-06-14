---
page: Cross-Service-Backlog
tags:
  - platform
  - backlog
  - cross-service
last-reviewed: 2026-06-14
---

# Cross-Service Backlog — Follow-On Work From 2026-06-06 → 2026-06-14 Doc Changes

[[README]] · [[platform/Code-Review-2026-06-13]] · [[platform/Overview]] · [[platform/Key-Decisions]]

> [!abstract] What this page is
> Between **2026-06-06** (vault initialisation) and **2026-06-14**, documentation across all
> seven services was updated to reflect the [[platform/Code-Review-2026-06-13|2026-06-13 platform code review]]
> and a wave of feature + security work. Many of those changes landed in **one** service but
> create **obligations in other services** that have not yet been implemented.
>
> This page is the authoritative cross-service work list. Each item states **why** it exists
> (the upstream change that triggered it), **what** must be built, the **exact contract**
> (endpoints, payloads, env vars, schemas) needed to build it, and **acceptance criteria**
> (linked to scenarios in [[services/alphaTest/Regression-Scenarios]] where one exists).
>
> An agent working in a single service repo should be able to read its own section here and
> execute without needing to re-derive the contract from other repos.

> [!info] Source of truth
> Everything below is derived from `git diff af11c67 HEAD` over `services/`, `reference/`,
> `platform/`, plus the uncommitted working-tree changes to `services/alphaKey/*`. When in
> doubt, re-run that diff and the alphaKey working-tree diff to confirm the current contract.

---

## How to read the priority labels

| Label | Meaning |
|---|---|
| **P0 — blocks a shipped feature** | An upstream feature is live/merged but unusable end-to-end until this work lands (e.g. password reset has no UI). |
| **P1 — correctness / contract** | A contract changed; consumers will break or silently misbehave until updated. |
| **P2 — hardening / completeness** | Improves security posture or operability; not blocking a user-visible flow. |

---

## 1. alphaLink (UI / BFF) — largest backlog

**Driver:** [[services/alphaKey/alphaKey|alphaKey]] shipped a large set of **user-facing** auth
features (password reset, MFA, session management, admin enable, login lockout) and
[[services/alphaTrade/alphaTrade|alphaTrade]] added two operator features (consensus gates,
bulk delete). alphaLink is the **only** UI in the platform. Today it has exactly three auth-adjacent
pages — `/login`, `/signup`, `/account` — plus a catch-all auth proxy. None of the new alphaKey
flows have a page or a dedicated BFF route, so they are unreachable by users.

> [!note] BFF pattern reminder
> Browser never calls backends directly. Every backend call goes through a Next.js route handler
> under `src/app/api/*`. Auth routes proxy to alphaKey; the server reads the `alphakey_session`
> httpOnly cookie via `src/lib/server-auth.ts` (`getSessionToken`) and forwards
> `Authorization: Bearer <token>` using `bearerHeader(token)`. Add new BFF routes the same way.
> See [[services/alphaLink/Architecture]] and [[services/alphaLink/API]].

### 1.1 Password reset flow — **P0**
**Why:** alphaKey added `POST /auth/forgot-password` and `POST /auth/reset-password`
(see [[services/alphaKey/API]]). No UI or BFF route exists, so a user who forgets their password
is permanently locked out.

**Build:**
- **Page `/forgot-password`** — email input form. On submit, calls BFF route below. Always shows
  the same neutral confirmation ("If an account exists for that email, a reset link has been sent")
  regardless of outcome — the backend deliberately never reveals whether the email exists.
- **Page `/reset-password`** — reads `?token=` from the query string, presents new-password +
  confirm fields, posts to BFF route below. On success, redirect to `/login` with a success banner.
  On `400`/expired/used token, show "This reset link is invalid or has expired" and a link back to
  `/forgot-password`.
- **BFF route `POST /api/auth/forgot-password`** → proxies `POST /auth/forgot-password`.
  Transparent body `{email}`. Pass through `204` as success. If alphaKey returns `501`
  (SMTP not configured), surface a clear admin-facing error ("Password reset is not available —
  email delivery is not configured").
- **BFF route `POST /api/auth/reset-password`** → proxies `POST /auth/reset-password`.
  Body `{token, new_password}`. On success alphaKey marks the token used, sets the new password,
  and **revokes all sessions** for that user — so after reset the user must log in fresh; do not
  attempt to auto-login with stale cookies.
- Add both `/forgot-password` and `/reset-password` to the middleware **public-route allowlist**
  (alongside `/login`, `/signup`, `/api/auth/*`) so the JWT guard does not redirect an
  unauthenticated user away from them. See [[services/alphaLink/Config]] middleware section.
- Add a "Forgot password?" link on `/login`.

**Contract (from alphaKey):**
| Endpoint | Auth | Body | Success | Notes |
|---|---|---|---|---|
| `POST /auth/forgot-password` | none | `{email}` | `204` always | `501` if `SMTP_HOST` unset. Writes `password_reset_token` + audit. |
| `POST /auth/reset-password` | none | `{token, new_password}` | `204`/`200` | Token is single-use, SHA-256 hashed server-side, TTL `PASSWORD_RESET_TTL` (default 3600s). Revokes all sessions on success. |

**Acceptance:** A user with no session can request a reset, receive a link (SMTP configured in
[[services/alphaFrame/alphaFrame|alphaFrame]] — see §4.1), set a new password, and log in with it;
old sessions are dead. The non-existent-email path is visually indistinguishable from the success path.

### 1.2 MFA / TOTP — **P0**
**Why:** alphaKey added `POST /auth/me/totp/setup`, `/verify`, `/disable`, the `user.totp_secret_encrypted`
+ `user.totp_enabled` columns, and a **login step-up**: when `totp_enabled=true`, `POST /auth/login`
requires a `totp_code` field; without it login returns `401` with header `X-TOTP-Required: true`.

**Build:**
- **Account-settings enrollment** (in `/account`): a "Two-factor authentication" section.
  1. Call BFF → `POST /auth/me/totp/setup`; response contains a **provisioning URI** (otpauth://…).
     Render it as a QR code (and show the raw secret for manual entry). MFA is **not yet active** at
     this point.
  2. Prompt the user for their first 6-digit code; call BFF → `POST /auth/me/totp/verify`. On success
     MFA is active — reflect `totp_enabled=true` in the UI.
  3. Provide a **Disable** action → BFF `POST /auth/me/totp/disable`, which requires the current TOTP
     code (collect it before calling).
- **Login step-up:** update the `/login` flow and the `POST /api/auth/login` BFF route. When the
  upstream login responds `401` with `X-TOTP-Required: true`, do **not** treat it as a failed
  password — instead reveal a "Authentication code" field and resubmit `POST /auth/login` with
  `{email, password, totp_code}`. Distinguish this from a genuine bad-password `401`.
- **BFF routes:** `POST /api/auth/me/totp/setup`, `POST /api/auth/me/totp/verify`,
  `POST /api/auth/me/totp/disable` — all Bearer-forwarded via `getSessionToken`/`bearerHeader`.

**Contract (from alphaKey):**
| Endpoint | Auth | Purpose |
|---|---|---|
| `POST /auth/me/totp/setup` | Bearer | Generates secret, returns provisioning URI. Stores encrypted secret. Does **not** enable MFA. |
| `POST /auth/me/totp/verify` | Bearer | Confirms first code → activates MFA. |
| `POST /auth/me/totp/disable` | Bearer | Requires current TOTP code. Deactivates MFA. |
| `POST /auth/login` | none | Accepts optional `totp_code`. If `totp_enabled` and code missing/invalid → `401` + `X-TOTP-Required: true`. |

**Acceptance:** A user can enrol MFA from `/account` (scan QR in an authenticator app, verify),
then on next login is prompted for a code and cannot log in without it; disabling MFA requires a
valid current code. New regression scenario required — see §6.

### 1.3 Active session management — **P0**
**Why:** alphaKey added `GET /auth/me/sessions` (list active refresh tokens) and
`DELETE /auth/me/sessions/{id}` (revoke one). No UI exists.

**Build:**
- In `/account`, a "Active sessions" list: one row per session showing whatever identifying fields
  alphaKey returns (created time, user-agent, IP, current-session marker). Each row has a "Revoke"
  button → BFF `DELETE /api/auth/me/sessions/{id}`.
- BFF routes `GET /api/auth/me/sessions` and `DELETE /api/auth/me/sessions/{id}`, Bearer-forwarded.
- If the user revokes the session they are currently using, handle the subsequent `401` gracefully
  (redirect to `/login?reason=session_expired`, reusing the existing session-expired UX).

**Acceptance:** A user can see their active sessions and revoke any of them; a revoked session's
access token is rejected platform-wide within ~1s (denylist). Aligns with
[[services/alphaTest/Regression-Scenarios|A6]] (kill-switch / session lifecycle).

### 1.4 Login lockout / rate-limit UX — **P1**
**Why:** alphaKey added login rate limiting: **20 attempts / IP / 5 min** and
**5 failures / account / 15 min**, with a 15-minute account lockout (Redis keys
`alphakey:rl:ip:*`, `alphakey:rl:acct:*`, `alphakey:lockout:*`). The login endpoint will now
return throttle/lockout responses (HTTP `429`).

**Build:** The `/login` page must detect the throttled/locked response and show a clear message
("Too many attempts. Try again in ~15 minutes.") instead of the generic "invalid credentials"
error. Do not retry automatically. Do not leak whether the email exists.

**Acceptance:** Repeated bad logins surface a lockout message, not a credentials error.
Aligns with [[services/alphaTest/Regression-Scenarios|A10]] (brute-force lockout).

### 1.5 Admin user management — enable / disable — **P2**
**Why:** alphaKey added `PATCH /auth/admin/users/{uid}/enable` and changed
`PATCH /auth/admin/users/{uid}/disable` to bump `token_version` + revoke all refresh tokens on
disable. There is no admin UI section in alphaLink today.

**Build:** An admin-only users view (gated on `role`) listing users (`GET /auth/admin/users`) with
**Enable**, **Disable**, role-change, and **Force logout** (`POST /auth/admin/users/{uid}/kill`)
actions, each via a Bearer-forwarded BFF route. Disable must visibly reflect that the user is
immediately logged out everywhere.

**Acceptance:** An admin can disable a user (who is then unable to log in or refresh) and re-enable
them. Aligns with [[services/alphaTest/Regression-Scenarios|A6]].

### 1.6 Consensus confidence gates in model-override editor — **P2**
**Why:** alphaTrade added `consensus_min_confidence` and `consensus_min_margin` to the per-model /
risk config (see [[services/alphaTrade/Config]]). The override editor UI does not expose them.

**Build:** Add two numeric inputs to the model-override form (`PUT /models/{run_name}/overrides`):
`consensus_min_confidence` (min winning-class probability, `0` = disabled) and
`consensus_min_margin` (min lead over runner-up, `0` = disabled). Document inline that low-conviction
signals below these thresholds are coerced to HOLD.

**Acceptance:** Operator can set both gates per model from the UI and see them persisted.

### 1.7 Bulk model delete — **P2**
**Why:** alphaTrade added `DELETE /models` (bulk) with body `{run_names: [...]}`, ownership-checked
per model, executed in parallel via `asyncio.gather`.

**Build:** On the models page(s) (`/trade/models/*`), add multi-select + "Delete selected" calling
a BFF route that proxies `DELETE /models` with the selected `run_names`. Surface per-model success/failure.

**Acceptance:** Operator can select several models and delete them in one action.

---

## 2. alphaTrade — contract + auth + infra dependencies

**Driver:** [[services/alphaGen/alphaGen|alphaGen]] changed the `model.ready` event schema and
[[services/alphaKey/alphaKey|alphaKey]] added `iss`/`aud` JWT claims; [[services/alphaFrame/alphaFrame|alphaFrame]]
now provisions per-service secrets that alphaTrade must consume.

### 2.1 Update the `model.ready` consumer — **P1 (contract change)**
**Why:** The event payload changed. alphaTrade consumes it in `runs.py run_events()` via the
SSE `GET /runs/events` stream (see [[services/alphaTrade/Interactions]] and [[reference/Event-Channels]]).

**Old payload (no longer sent):**
```json
{ "type": "model.ready", "run_name": "...", "version": "...",
  "minio_bucket": "models", "minio_path": ".../model.onnx", "manifest_path": ".../manifest.json" }
```
**New payload — consumers MUST tolerate BOTH emit paths:**
```json
{ "run_name": "aapl_daily_mlp", "version": "3",
  "published_at": "2026-06-14T10:00:00+00:00",
  "artifact_prefix": "user/account/aapl_daily_mlp/v3" }
```
| Emit path | `artifact_prefix` present? |
|---|---|
| Celery worker auto-promote (gate passed) | **No** |
| `POST /runs/{id}/publish` (explicit MinIO push) | **Yes** |

**Build:** Rewrite the consumer to:
- Stop reading `type`, `minio_bucket`, `minio_path`, `manifest_path` (gone).
- Use `artifact_prefix` to locate artifacts in the `models` bucket **when present**; when **absent**
  (auto-promote path), resolve the object location via the `latest` pointer / `{run_name}/v{version}/`
  layout documented in [[services/alphaGen/Data]] (MinIO Object Layout).
- Tolerate unknown/extra fields (forward-compatible parsing).
- Parse `published_at` as ISO-8601 with timezone.

**Acceptance:** A model published via **both** auto-promote and explicit `/runs/{id}/publish`
triggers `model_sync` correctly. Aligns with [[services/alphaTest/Regression-Scenarios|D3]]
(model.ready schema — single versioned shape, consumers tolerate unknown fields) and the upstream
contract test `tests/contract/test_model_ready_schema.py`.

### 2.2 JWT `iss` / `aud` validation — **P1**
**Why:** alphaKey now writes `iss: "alphakey"` and `aud: "alphakey"` into every JWT and verifies
them on decode (see [[services/alphaKey/Architecture]]). alphaTrade verifies tokens via
`make_jwt_dep` (ES256 against alphaKey JWKS).

**Build:** Confirm `make_jwt_dep` either validates `iss`/`aud` against the configured values or at
minimum does **not** reject tokens that now carry these claims. If the JWT library enforces audience
by default, set the expected `aud`/`iss` (`alphakey`, overridable via `JWT_AUDIENCE`/`JWT_ISSUER`).
`kid` remains in the JWT **header** (not payload) for JWKS lookup.

**Acceptance:** Valid alphaKey-issued tokens authenticate against alphaTrade `/api/v1/*`; tokens
with a wrong `aud`/`iss` are rejected. Aligns with [[services/alphaTest/Regression-Scenarios|L16]]
(API auth enforced).

### 2.3 Secrets-at-rest key from infra — **P1**
**Why:** alphaTrade added Fernet at-rest encryption for `BotSettings` secret columns (API keys,
SMTP password, Slack webhook), gated on `DB_SECRETS_KEY` (see [[services/alphaTrade/Config]] and
[[services/alphaTrade/Architecture]]). Without the key, secrets are stored in **plaintext**.

**Build (alphaTrade side):** Nothing new in code — but the deployment **must** receive
`DB_SECRETS_KEY` from compose (§4.2), or switch to `SECRETS_SOURCE=alphakey` to read from the
alphaKey vault at runtime. For production, [[platform/COMPLIANCE]] requires `SECRETS_SOURCE=alphakey`.
Verify the migration-safe fallback (plaintext rows written before encryption are returned as-is).

**Acceptance:** With `DB_SECRETS_KEY` set, secret columns are ciphertext at rest and transparently
decrypted on read. Aligns with [[services/alphaTest/Regression-Scenarios|V5]].

---

## 3. alphaGen — auth claim alignment + config docs

**Driver:** [[services/alphaKey/alphaKey|alphaKey]] `iss`/`aud` claims; [[services/alphaFrame/alphaFrame|alphaFrame]]
scoped MinIO accounts.

### 3.1 JWT `iss` / `aud` validation — **P1**
**Why:** alphaGen now requires Bearer JWT on all `/runs` routes via
`att.security.alphakey_auth.require_auth`, verifying ES256 against alphaKey JWKS
(see [[services/alphaGen/API]], [[services/alphaGen/Interactions]]). The documented verify path
checks signature + `exp` but predates the new `iss`/`aud` claims.

**Build:** Extend the verifier to validate `iss=alphakey` and `aud=alphakey` (configurable via
`JWT_ISSUER`/`JWT_AUDIENCE`), consistent with alphaKey and alphaTrade. Ensure tokens carrying these
claims are not rejected, and tokens with the wrong values are.

**Acceptance:** Valid tokens authenticate on `/runs/*`; mismatched `aud`/`iss` rejected.
Aligns with [[services/alphaTest/Regression-Scenarios|T12]] (ownership / auth) and A-suite.

### 3.2 Document scoped MinIO credentials — **P2 (doc gap)**
**Why:** [[services/alphaFrame/alphaFrame|alphaFrame]] now creates a per-service MinIO user
(`alphagen` → `models` bucket only) and injects `MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY` into
`alphagen-api`/`alphagen-worker`. [[services/alphaGen/Config]] does not document these variables.

**Build:** Add `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` (scoped account, not MinIO root) to
[[services/alphaGen/Config]]. Confirm the publish path uses them.

**Acceptance:** alphaGen config docs list the scoped MinIO credentials and match what compose injects.

---

## 4. alphaFrame — compose plumbing — ✅ CLOSED (2026-06-14)

> [!success] All items closed — alphaFrame session 2026-06-14
> All compose wiring, container additions, observability config, and doc updates landed in one
> session. See commit `8ae73c1` (alphaFrame) and `4650e95` / `7a74032` (alphaDocs).

**Driver:** New features in alphaKey (password reset, MFA, key rotation, CORS, JWT claims),
alphaGen (drift monitoring), and alphaTrade (at-rest secrets) required environment + container
wiring that alphaFrame owns.

### 4.1 SMTP env for alphaKey password reset — ✅ Done
`SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_TLS`,
`PASSWORD_RESET_TTL` threaded to `alphakey-api` service in compose.

### 4.2 `DB_SECRETS_KEY` for alphaTrade — ✅ Done
`DB_SECRETS_KEY: ${DB_SECRETS_KEY:-}` injected into `alphatrade`. Documented in
[[services/alphaFrame/Config]].

### 4.3 Celery Beat for alphaGen drift monitoring — ✅ Done
`alphagen-beat` container added (same image as worker, `celery -A att.api.worker beat`).
All drift env vars (`DRIFT_ENABLED`, `DRIFT_PSI_THRESHOLD`, `DRIFT_KS_THRESHOLD`,
`DRIFT_CHECK_INTERVAL_SECONDS`, `DRIFT_LOOKBACK_BARS`, `DRIFT_MIN_LIVE_BARS`,
`RETRAIN_MAX_ATTEMPTS`, `RETRAIN_RETRY_DELAY_SECONDS`) threaded. Capped 0.5 CPU / 512M.

### 4.4 Grafana panel + alert for drift gauges — ✅ Done
`AlphaGenDriftPSI` (threshold 0.20) and `AlphaGenDriftKS` (threshold 0.10) Prometheus alert
rules added (`severity: critical` → routes to Slack/PagerDuty). Drift timeseries panel added to
`observability/grafana/dashboards/services.json`.

### 4.5 Vault master-key rotation env naming — ✅ Done
`VAULT_MASTER_KEY_1: ${VAULT_MASTER_KEY_1:-}` threaded to `alphakey-api`. Rotation runbook
references `alphakey rotate-master-key` CLI per [[services/alphaKey/Config]].

### 4.6 JWT issuer/audience + CORS passthrough — ✅ Done
`JWT_ISSUER`, `JWT_AUDIENCE` threaded to `alphakey-api`, `alphagen-api`, `alphagen-worker`,
`alphagen-beat`, and `alphatrade`. `ALPHAKEY_CORS_ORIGINS` threaded to `alphakey-api` (defaults
empty — nginx fronts everything). `ALPHAKEY_URL`/`ALPHAKEY_SERVICE_TOKEN`/`SECRETS_SOURCE`
also threaded to all consumers.

### 4.7 Redis password consistency — ✅ Done
All service Redis URLs confirmed authenticated (`redis://:${REDIS_PASSWORD}@redis:6379/N`).
alphakey-api uses DB 2 to avoid collision with Celery broker/results (DBs 0/1).
[[services/alphaKey/Config]], [[services/alphaGen/Config]], [[services/alphaTrade/Config]] all
updated to show authenticated URL format.

---

## 5. alphaKey — outbound contract publication

**Driver:** alphaKey is the **source** of most changes; its remaining cross-service duty is to make
the contract unambiguous for consumers.

- **Publish the JWT claim + JWKS contract — P2.** Document the full claim set now issued
  (`iss`, `aud`, `sub`, `role`, `jti`, `tv`, `iat`, `exp`, plus `kid` in the header) in a place
  consumers reference, so alphaGen (§3.1) and alphaTrade (§2.2) validate identically. The verify
  path is: JWKS → find key by `kid` (header) → verify signature → check `exp`/`iss`/`aud` →
  Redis denylist by `jti` → `tv` matches `user.token_version`. See [[services/alphaKey/Architecture]].
- `alphakey rotate-master-key` CLI is already documented (idempotent, `--dry-run`); ensure the
  rotation runbook in [[services/alphaFrame/alphaFrame|alphaFrame]] (§4.5) references it.

---

## 6. alphaTest / alphaPerf — new scenarios for the new features — ✅ CLOSED (2026-06-14)

> [!success] Closed — documentation updated
> Both catalogues now cover the new alphaKey auth features. No further backlog action for these two
> services; the scenarios/benchmarks below are documented and await implementation in the alphaTest /
> alphaPerf repos (no code exists in either yet — these are spec pages).

**Driver:** The [[services/alphaTest/Regression-Scenarios|regression catalogue]] (2026-06-13) covered
brute-force lockout (A10) and the auth chain, but **predated** the MFA, password-reset, and
session-management features now in alphaKey.

**Done (alphaTest):** Added to [[services/alphaTest/Regression-Scenarios]] §1:
- **A12** — MFA/TOTP enrol (setup→verify) + login step-up (`401` + `X-TOTP-Required` without code;
  success with code; wrong code rejected).
- **A13** — MFA disable requires current code.
- **A14** — password-reset happy path (forgot→email→reset→login; old password dead).
- **A15** — reset-token security (single-use, TTL-bound, revokes all sessions, email non-enumeration,
  `501` when SMTP off).
- **A16** — active session list + revoke (platform-wide kill ~1s; current-session revoke degrades cleanly).
- Plus golden artifacts (RFC 6238 TOTP vectors, mock SMTP sink) and priority-order entries.

**Done (alphaPerf):** Added to [[services/alphaPerf/Performance-Test-Plan]]:
- MFA-login budget row + a login-storm MFA variant + a **TOTP-verify micro-benchmark** (P2.4).
- Explicit note that password-reset is not a perf target (SMTP-bound, low rps) — only guard that the
  SMTP send stays off the request path.

---

## Quick index — priority across services

| Priority | Items |
|---|---|
| **P0** | 1.1 password-reset UI · 1.2 MFA UI + step-up · 1.3 sessions UI |
| **P1** | 1.4 lockout UX · 2.1 model.ready consumer · 2.2 alphaTrade iss/aud (code) · 2.3 DB_SECRETS_KEY (verify migration-safe fallback) · 3.1 alphaGen iss/aud (code) |
| **P2** | 1.5 admin enable/disable · 1.6 consensus gates UI · 1.7 bulk delete · 3.2 MinIO creds docs · 5 JWT claim contract |
| **✅ Closed** | §4 all alphaFrame wiring (2026-06-14) · §6 alphaTest/alphaPerf scenarios (2026-06-14) |

---

*Compiled 2026-06-14 from the 2026-06-06 → 2026-06-14 documentation diff. Update or close items as
the owning service repos land the work. Cross-reference [[platform/Code-Review-2026-06-13]] for the
originating bead IDs.*
