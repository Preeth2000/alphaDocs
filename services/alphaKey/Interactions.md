---
service: alphaKey
page: Interactions
tags:
  - service/alphaKey
  - interactions
---

# alphaKey — Interactions

[[alphaKey|alphaKey]] · [[alphaDocs/services/alphaKey/Architecture|Architecture]] · [[alphaDocs/services/alphaKey/API|API]] · [[alphaDocs/services/alphaKey/Data|Data]] · [[alphaDocs/services/alphaKey/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger |
|---|---|---|---|
| [[alphaLink\|alphaLink]] BFF | `POST /auth/login` | `{email, password}` | User login |
| [[alphaLink\|alphaLink]] BFF | `POST /auth/register` | `{email, password}` | User signup |
| [[alphaLink\|alphaLink]] BFF | `POST /auth/refresh` | `{refresh_token}` | Token refresh (from httpOnly cookie) |
| [[alphaLink\|alphaLink]] BFF | `POST /auth/logout` | `{jti, exp, refresh_token}` | User logout |
| [[alphaLink\|alphaLink]] BFF | `PUT /auth/vault/{provider}/{account}/{name}` | `{value}` | User saves credential |
| [[alphaLink\|alphaLink]] BFF | Admin endpoints | Various | Developer user actions |
| [[alphaGen\|alphaGen]] | `GET /auth/.well-known/jwks.json` | — | JWT verification (cached 5min) |
| [[alphaGen\|alphaGen]] | `GET /auth/internal/secrets/{user_id}` | Query: provider, account | Vault secret fetch |
| [[alphaTrade\|alphaTrade]] | `POST /auth/introspect` | `{token}` | Definitive token check |
| [[alphaTrade\|alphaTrade]] | `GET /auth/.well-known/jwks.json` | — | JWT verification |
| Trading 212 API | HTTP response | Account summary | `POST /auth/vault/t212/verify` |
| Polygon.io API | HTTP response | Exchange list | `POST /auth/vault/polygon/verify` |

---

## Outputs

| Destination | Mechanism | Format | Trigger |
|---|---|---|---|
| [[alphaLink\|alphaLink]] BFF | HTTP response | `{access_token, ...}` | Login / refresh |
| [[alphaLink\|alphaLink]] BFF | HTTP response | User object `{id, email, role, minio_user, ...}` | `GET /auth/me` |
| [[alphaLink\|alphaLink]] BFF | HTTP response | Masked credential list | `GET /auth/vault` |
| [[alphaGen\|alphaGen]] | HTTP response | JWKS JSON `{keys: [...]}` | `GET /auth/.well-known/jwks.json` |
| [[alphaGen\|alphaGen]] | HTTP response | Decrypted secrets `{secrets: [{provider, account, name, value}]}` | `GET /auth/internal/secrets/{uid}` |
| [[alphaTrade\|alphaTrade]] | HTTP response | `{active, sub, role, jti}` | `POST /auth/introspect` |
| Redis | SET | JTI → expiry | `POST /auth/logout` |
| PostgreSQL | INSERT | AuditLog, CredentialAccessAudit records | Every auth event + vault access |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[alphaFrame\|alphaFrame]] → Postgres | `alphakey` database | ✅ |
| [[alphaFrame\|alphaFrame]] → Redis | JTI denylist | ✅ (fail-closed on Redis unavailable) |
| Trading 212 API | Credential verification response | ⬜ (only for vault verify endpoint) |
| Polygon.io API | Credential verification response | ⬜ (only for vault verify endpoint) |

---

## Downstream Consumers

| Service | What it gets |
|---|---|
| [[alphaLink\|alphaLink]] | JWT access tokens, refresh tokens, user identity, vault read/write |
| [[alphaGen\|alphaGen]] | JWKS public keys, per-user decrypted secrets |
| [[alphaTrade\|alphaTrade]] | JWKS public keys, token introspection results |

> [!warning] Partial integration
> alphaTrade and alphaGen only use alphaKey when `AUTH_MODE=alphakey` and `SECRETS_SOURCE=alphakey` respectively. Both default to `legacy` mode in current deployments.
