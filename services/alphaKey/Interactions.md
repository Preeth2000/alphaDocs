---
service: alphaKey
page: Interactions
tags:
  - service/alphaKey
  - interactions
---

# alphaKey — Interactions

[[services/alphaKey/alphaKey|alphaKey]] · [[services/alphaKey/Architecture|Architecture]] · [[services/alphaKey/API|API]] · [[services/alphaKey/Data|Data]] · [[services/alphaKey/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger |
|---|---|---|---|
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /auth/login` | `{email, password}` | User login |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /auth/register` | `{email, password}` | User signup |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /auth/refresh` | `{refresh_token}` | Token refresh (from httpOnly cookie) |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `POST /auth/logout` | `{jti, exp, refresh_token}` | User logout |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | `PUT /auth/vault/{provider}/{account}/{name}` | `{value}` | User saves credential |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | Admin endpoints | Various | Developer user actions |
| [[services/alphaGen/alphaGen\|alphaGen]] | `GET /auth/.well-known/jwks.json` | — | JWT verification (cached 5min) |
| [[services/alphaGen/alphaGen\|alphaGen]] | `GET /auth/internal/secrets/{user_id}` | Query: provider, account | Vault secret fetch |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | `POST /auth/introspect` | `{token}` | Definitive token check |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | `GET /auth/.well-known/jwks.json` | — | JWT verification |
| Trading 212 API | HTTP response | Account summary | `POST /auth/vault/t212/verify` |
| Polygon.io API | HTTP response | Exchange list | `POST /auth/vault/polygon/verify` |

---

## Outputs

| Destination | Mechanism | Format | Trigger |
|---|---|---|---|
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | HTTP response | `{access_token, ...}` | Login / refresh |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | HTTP response | User object `{id, email, role, minio_user, ...}` | `GET /auth/me` |
| [[services/alphaLink/alphaLink\|alphaLink]] BFF | HTTP response | Masked credential list | `GET /auth/vault` |
| [[services/alphaGen/alphaGen\|alphaGen]] | HTTP response | JWKS JSON `{keys: [...]}` | `GET /auth/.well-known/jwks.json` |
| [[services/alphaGen/alphaGen\|alphaGen]] | HTTP response | Decrypted secrets `{secrets: [{provider, account, name, value}]}` | `GET /auth/internal/secrets/{uid}` |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | HTTP response | `{active, sub, role, jti}` | `POST /auth/introspect` |
| Redis | SET | JTI → expiry | `POST /auth/logout` |
| PostgreSQL | INSERT | AuditLog, CredentialAccessAudit records | Every auth event + vault access |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Postgres | `alphakey` database | ✅ |
| [[services/alphaFrame/alphaFrame\|alphaFrame]] → Redis | JTI denylist | ✅ (fail-closed on Redis unavailable) |
| Trading 212 API | Credential verification response | ⬜ (only for vault verify endpoint) |
| Polygon.io API | Credential verification response | ⬜ (only for vault verify endpoint) |

---

## Downstream Consumers

| Service | What it gets |
|---|---|
| [[services/alphaLink/alphaLink\|alphaLink]] | JWT access tokens, refresh tokens, user identity, vault read/write |
| [[services/alphaGen/alphaGen\|alphaGen]] | JWKS public keys, per-user decrypted secrets |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | JWKS public keys, token introspection results |

> [!warning] Partial integration
> alphaTrade and alphaGen only use alphaKey when `AUTH_MODE=alphakey` and `SECRETS_SOURCE=alphakey` respectively. Both default to `legacy` mode in current deployments.
