---
service: alphaKey
page: API
tags:
  - service/alphaKey
  - api
---

# alphaKey — API

[[services/alphaKey/alphaKey|alphaKey]] · [[services/alphaKey/Architecture|Architecture]] · [[services/alphaKey/Interactions|Interactions]] · [[services/alphaKey/Data|Data]] · [[services/alphaKey/Config|Config]]

---

## Inbound Endpoints

**Base URL:** `http://alphakey-api:8000` (internal) | `https://localhost/auth/` (via Nginx)

> [!warning] `/auth/internal/*` not exposed via Nginx
> Internal service routes (secrets, introspect) are service-to-service only on the `platform` network.

### User Auth (`/auth/`)

| Method | Path | Purpose | Caller | DB R | DB W | Auth |
|---|---|---|---|---|---|---|
| `POST` | `/auth/register` | Create user account | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (User + AuditLog) | None |
| `POST` | `/auth/login` | Authenticate → issue access + refresh tokens | [[services/alphaLink/alphaLink\|alphaLink]] | 2 | 2 (RefreshToken + AuditLog) | None |
| `POST` | `/auth/refresh` | Rotate refresh token, issue new access JWT | [[services/alphaLink/alphaLink\|alphaLink]] | 2 | 3 (revoke old + INSERT new RefreshToken + AuditLog) | Refresh token in body |
| `POST` | `/auth/logout` | Revoke refresh token + add JTI to denylist | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (RefreshToken + AuditLog) + Redis write | Bearer JWT |
| `POST` | `/auth/logout-all` | Bump token_version, revoke all sessions | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 3 (User version + RevokAll RefreshTokens + AuditLog) | Bearer JWT |

### Current User (`/auth/me`)

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/auth/me` | Get current user identity | [[services/alphaLink/alphaLink\|alphaLink]] BFF (login flow) | 1 | 0 |
| `PATCH` | `/auth/me/password` | Change password (bumps token_version) | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 3 (User + revoke all + AuditLog) |

### Vault (`/auth/vault/`)

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/auth/vault` | List secrets (masked — last 4 chars only) | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 0 |
| `PUT` | `/auth/vault/{provider}/{account}/{name}` | Create or update secret (encrypted) | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (Credential + CredentialAccessAudit) |
| `DELETE` | `/auth/vault/{provider}/{account}/{name}` | Delete secret | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (delete + CredentialAccessAudit) |
| `POST` | `/auth/vault/{provider}/verify` | Connectivity check using stored credentials | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 1 (CredentialAccessAudit) + outbound HTTP |

Allowed providers: `t212`, `polygon`, `smtp`, `slack`, `minio`, `alphatrade`

**Verify behaviour:**
- `t212`: GET `{demo|live}.trading212.com/api/v0/equity/account/summary` with stored key
- `polygon`: GET Polygon `/meta/exchanges` with stored key

### Admin (`/auth/admin/`) — developer role required

| Method | Path | Purpose | Caller | DB R | DB W |
|---|---|---|---|---|---|
| `GET` | `/auth/admin/users` | List all users | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 0 |
| `PATCH` | `/auth/admin/users/{uid}/role` | Change user role | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (User + AuditLog) |
| `PATCH` | `/auth/admin/users/{uid}/disable` | Deactivate user | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 2 (User + AuditLog) |
| `POST` | `/auth/admin/users/{uid}/kill` | Force logout all sessions | [[services/alphaLink/alphaLink\|alphaLink]] | 1 | 3 (version bump + revoke all + AuditLog) |

### Service-to-Service (internal network only)

| Method | Path | Purpose | Caller | DB R | DB W | Auth |
|---|---|---|---|---|---|---|
| `GET` | `/auth/.well-known/jwks.json` | Public signing keys | [[services/alphaGen/alphaGen\|alphaGen]], [[services/alphaTrade/alphaTrade\|alphaTrade]] | 1 | 0 | None (Cache-Control: 5min) |
| `POST` | `/auth/introspect` | Definitive token validity check | [[services/alphaTrade/alphaTrade\|alphaTrade]] | 2 (User + RedisCheck) | 0 | `X-Service-Token` |
| `GET` | `/auth/internal/secrets/{user_id}` | Return decrypted secrets for user | [[services/alphaGen/alphaGen\|alphaGen]] | 2 | 1 (CredentialAccessAudit) | `X-Service-Token` |

### Health

| Method | Path | Port | Purpose |
|---|---|---|---|
| `GET` | `/healthz` | 8000 | Liveness — returns `{status: ok}` |
| `GET` | `/readyz` | 8000 | Readiness — checks DB + Redis |

---

## Outbound Calls

| Target | Purpose | When |
|---|---|---|
| Trading 212 API | Credential verification | `POST /auth/vault/t212/verify` |
| Polygon.io API | Credential verification | `POST /auth/vault/polygon/verify` |
| OTel Collector :4317 | Traces + metrics | Always (async) |
