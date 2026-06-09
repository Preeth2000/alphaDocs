---
service: alphaKey
status: partial
stack: Python 3.11, FastAPI, SQLModel, Alembic, PostgreSQL, Redis, JWT (ES256), Argon2, Fernet (envelope encryption)
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaKey
  - status/partial
  - auth
---

# alphaKey

> Authentication, account management, and encrypted credential vault for the alphaPlatform.

**Status:** 🟡 Partial — core auth implemented; integration with alphaTrade/alphaGen in progress (depends on `AUTH_MODE` and `SECRETS_SOURCE` env vars)  
**Port:** `8000`  
**Repo path:** `projectAlpha/alphaKey/`

---

## Contents

| Page | Description |
|---|---|
| [[services/alphaKey/Architecture\|Architecture]] | Auth flow, vault encryption, token lifecycle |
| [[services/alphaKey/Interactions\|Interactions]] | Inputs/outputs, callers, service-to-service auth |
| [[services/alphaKey/API\|API]] | All endpoints + JWKS + introspection |
| [[services/alphaKey/Data\|Data]] | 6 DB tables, read/write counts |
| [[services/alphaKey/Config\|Config]] | Env vars, token TTLs, vault key config |

---

## Mermaid Flow

```mermaid
flowchart TD
    subgraph Callers
        AL[alphaLink BFF]
        AG[alphaGen worker]
        AT[alphaTrade API]
    end

    subgraph alphaKey[:8000]
        AUTH[/auth/* - user auth]
        VAULT[/auth/vault/* - credential storage]
        ADMIN[/auth/admin/* - user mgmt]
        JWKS[/auth/.well-known/jwks.json]
        INTROSPECT[/auth/introspect - service-to-service]
        SECRETS[/auth/internal/secrets/{uid} - service-to-service]
        DB[(PostgreSQL\nalphakey DB)]
        RD[(Redis\ntoken denylist)]
    end

    AL -->|login/logout/refresh/register| AUTH
    AL -->|read/write vault| VAULT
    AL -->|admin actions| ADMIN
    AG -->|JWKS fetch (JWT verify)| JWKS
    AG -->|fetch user secrets| SECRETS
    AT -->|introspect token| INTROSPECT
    AT -->|JWKS fetch| JWKS

    AUTH --> DB & RD
    VAULT --> DB
    INTROSPECT --> DB & RD
    SECRETS --> DB
    ADMIN --> DB
```

---

## Related

- [[platform/Overview]]
- [[services/alphaLink/alphaLink|alphaLink]] — primary user-facing caller
- [[services/alphaGen/alphaGen|alphaGen]] — consumes JWKS + vault secrets
- [[services/alphaTrade/alphaTrade|alphaTrade]] — consumes JWKS + introspection
- [[reference/Glossary]] — JWT, vault, envelope encryption
