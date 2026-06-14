---
service: alphaKey
page: Data
tags:
  - service/alphaKey
  - data
---

# alphaKey — Data

[[services/alphaKey/alphaKey|alphaKey]] · [[services/alphaKey/Architecture|Architecture]] · [[services/alphaKey/Interactions|Interactions]] · [[services/alphaKey/API|API]] · [[services/alphaKey/Config|Config]]

---

## Datastores

| Store | Type | Purpose |
|---|---|---|
| `alphakey` Postgres | Relational | All identity, token, vault, audit data |
| Redis db0 | Cache | JWT JTI denylist (instant token revocation) |

---

## Database Tables (8)

Database: `alphakey`  
Managed by: Alembic (`alphakey upgrade head` on startup)  
Migrations: `alphakey/store/migrations/versions/`

### `user`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID string PK | User identifier |
| `email` | String unique (indexed) | Login identifier |
| `password_hash` | String | Argon2 hash |
| `role` | String | `admin` \| `developer` \| `standard` |
| `minio_user` | String nullable | MinIO namespace prefix for user |
| `minio_account` | String nullable | MinIO namespace account |
| `is_active` | Bool (default true) | Login allowed? |
| `token_version` | Int (default 0) | Bumped on logout-all/password-change → offline invalidation |
| `totp_secret_encrypted` | Text nullable | Fernet-encrypted TOTP base32 secret; null if MFA never set up |
| `totp_enabled` | Bool (default false) | Whether TOTP MFA is active |
| `created_at`, `updated_at` | DateTime UTC | Timestamps |

### `refresh_token`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID string PK | Token record ID |
| `user_id` | String (indexed) | Owner |
| `token_hash` | String | SHA-256 hex of raw opaque token |
| `expires_at` | DateTime UTC | When token expires |
| `revoked_at` | DateTime nullable | When revoked |
| `rotated_to` | String nullable | ID of replacement token (rotation chain) |
| `user_agent`, `ip` | String | Audit |
| `created_at` | DateTime UTC | |

### `signing_key`

| Column | Type | Purpose |
|---|---|---|
| `kid` | String PK | Key ID (referenced in JWT `kid` claim) |
| `algorithm` | String | `ES256` |
| `public_jwk` | JSON | Public key as JWK dict |
| `private_pem_encrypted` | String | Fernet-encrypted private PEM (using KEK) |
| `status` | String | `active` \| `retiring` \| `retired` |
| `created_at` | DateTime | |
| `retire_after` | DateTime nullable | When to transition to `retiring` |

### `credential`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID string PK | |
| `user_id` | String (indexed) | Owner |
| `provider` | String | `t212` \| `polygon` \| `smtp` \| `slack` \| `minio` \| `alphatrade` |
| `account` | String | `demo` \| `invest` \| `isa` \| `default` |
| `name` | String | `api_key` \| `secret_key` \| `webhook_url` \| `smtp_password` \| … |
| `ciphertext` | String | Fernet token of encrypted secret |
| `dek_wrapped` | String | Fernet token of DEK, wrapped by KEK |
| `key_version` | Int | KEK version that wrapped DEK |
| `secret_last4` | String | Last 4 plaintext chars; stored at write time to avoid decryption on list |
| `created_at`, `updated_at` | DateTime | |

Unique constraint: `(user_id, provider, account, name)`

### `audit_log` (append-only)

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID PK | |
| `ts` | DateTime (indexed) | Event time |
| `actor_user_id` | String nullable | Who performed action |
| `target_user_id` | String nullable | Who was affected |
| `event` | String | login \| login_fail \| refresh \| logout \| logout_all \| logout_all \| register \| role_change \| kill_switch \| password_change \| password_reset_requested \| password_reset \| totp_enabled \| totp_disabled \| session_revoke |
| `ip`, `user_agent` | String | Request context |
| `detail` | JSON string | Additional context |

### `password_reset_token`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID PK | |
| `user_id` | String (indexed) | Owner |
| `token_hash` | String (indexed) | SHA-256 of raw URL-safe token |
| `expires_at` | DateTime UTC | Expiry (default 1 hour from creation) |
| `used_at` | DateTime nullable | Set on first use (single-use) |
| `created_at` | DateTime UTC | |

### `credential_access_audit` (append-only)

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID PK | |
| `ts` | DateTime (indexed) | Access time |
| `accessor` | String | user_id or `svc:service_name` |
| `user_id` | String (indexed) | Whose secret was accessed |
| `provider`, `account`, `name` | String | Which secret |
| `action` | String | `read` \| `write` \| `delete` |
| `ip` | String | |

---

## DB Read/Write Count per Operation

| Operation | Reads | Writes | Tables |
|---|---|---|---|
| `POST /auth/register` | 1 (email check) | 2 | `user`, `audit_log` |
| `POST /auth/login` | 2 | 2 | `user` (verify), `refresh_token` (INSERT), `audit_log` |
| `POST /auth/refresh` | 2 | 3 | `refresh_token` (verify+revoke+insert), `audit_log` |
| `POST /auth/logout` | 1 | 2 + Redis | `refresh_token` (revoke), `audit_log`, Redis (JTI denylist) |
| `GET /auth/me` | 1 | 0 | `user` |
| `GET /auth/vault` | 1 | 0 | `credential` (list, masked) |
| `PUT /auth/vault/*` | 1 | 2 | `credential` (upsert), `credential_access_audit` |
| `GET /auth/internal/secrets/{uid}` | 2 | 1 | `credential` (decrypt), `credential_access_audit` |
| `POST /auth/introspect` | 2 | 0 | `user` (token_version), Redis (denylist check) |
| `GET /auth/.well-known/jwks.json` | 1 | 0 | `signing_key` |

---

## Migration History

| Migration | Change |
|---|---|
| `0001_initial_identity` | Create `user`, `refresh_token`, `signing_key`, `audit_log` |
| `0002_credential_vault` | Create `credential`, `credential_access_audit` |
| `0003_refresh_token_hash_index` | Index `refresh_token.token_hash` for O(1) lookup |
| `0004_credential_last4` | Add `credential.secret_last4` column |
| `0005_password_reset_token` | Create `password_reset_token` table |
| `0006_user_totp` | Add `user.totp_secret_encrypted`, `user.totp_enabled` columns |

---

## Redis Key Space

| Key pattern | Purpose | TTL |
|---|---|---|
| `alphakey:denylist:{jti}` | JTI denylist entry (instant revocation) | Token `exp - now` |
| `alphakey:rl:ip:{ip}` | Login rate limit counter per IP | 300s (5 min window) |
| `alphakey:rl:acct:{sha256}` | Login failure counter per account | 900s (15 min window) |
| `alphakey:lockout:{sha256}` | Account lockout flag | 900s (15 min lockout) |
