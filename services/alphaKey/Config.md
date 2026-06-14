---
service: alphaKey
page: Config
tags:
  - service/alphaKey
  - config
---

# alphaKey — Config

[[services/alphaKey/alphaKey|alphaKey]] · [[services/alphaKey/Architecture|Architecture]] · [[services/alphaKey/Interactions|Interactions]] · [[services/alphaKey/API|API]] · [[services/alphaKey/Data|Data]]

---

## Environment Variables

Source: `alphaKey/.env.example`

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `DATABASE_URL` | `postgresql://platform:changeme@postgres:5432/alphakey` | ✅ | Postgres connection |
| `REDIS__URL` | `redis://:password@redis:6379/2` | ✅ | Redis for JTI denylist (DB 2; password required — see `REDIS_PASSWORD` in alphaFrame) |
| `VAULT_MASTER_KEY` | — | ✅ | 32-byte base64url KEK. **Service refuses to start if missing.** |
| `VAULT_MASTER_KEY_1`, `_2`, … | — | ⬜ | Previous key versions in rotation order (newest → oldest). `_1` = previous key, `_2` = key before that. |
| `ALPHAKEY_SERVICE_TOKENS` | — | ✅ | Comma-separated `token:principal_name` pairs for service-to-service auth |
| `ALPHAKEY_SERVICE_TOKEN_TRADE` | — | ⬜ | alphaTrade's service token (subset of SERVICE_TOKENS) |
| `ALPHAKEY_SERVICE_TOKEN_GEN` | — | ⬜ | alphaGen's service token |
| `ALPHAKEY_CORS_ORIGINS` | `""` | ⬜ | Comma-separated allowed CORS origins. Empty = no CORS headers sent (default). Example: `https://app.example.com` |
| `ACCESS_TOKEN_TTL` | `600` | ⬜ | JWT access token TTL (seconds, default 10 min) |
| `REFRESH_TOKEN_TTL` | `1209600` | ⬜ | Refresh token TTL (seconds, default 14 days) |
| `JWT_ISSUER` | `alphakey` | ⬜ | `iss` claim written into JWTs and verified on decode |
| `JWT_AUDIENCE` | `alphakey` | ⬜ | `aud` claim written into JWTs and verified on decode |
| `ALPHAKEY_PORT` | `8000` | ⬜ | API listen port |
| `OTEL_SERVICE_NAME` | `alphakey` | ⬜ | OTel service name |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector:4317` | ⬜ | OTel Collector endpoint |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | ⬜ | OTLP protocol |
| `SMTP_HOST` | `""` | ⬜ | SMTP relay host for password reset emails. Reset flow returns `501` if empty. |
| `SMTP_PORT` | `587` | ⬜ | SMTP port |
| `SMTP_FROM` | `noreply@alphakey.local` | ⬜ | From address for reset emails |
| `SMTP_USER` | `""` | ⬜ | SMTP auth username (leave empty for unauthenticated relay) |
| `SMTP_PASSWORD` | `""` | ⬜ | SMTP auth password |
| `SMTP_TLS` | `true` | ⬜ | Use STARTTLS |
| `PASSWORD_RESET_TTL` | `3600` | ⬜ | Reset token lifetime in seconds (default 1 hour) |

---

## Service Token Format

`ALPHAKEY_SERVICE_TOKENS` is a comma-separated list of `token:principal` pairs:

```
ALPHAKEY_SERVICE_TOKENS=abc123:alphatrade,def456:alphagen
```

Each token grants access to the `/auth/introspect` and `/auth/internal/secrets/*` endpoints. The principal name is used for `CredentialAccessAudit` logging (`svc:alphatrade`).

---

## Vault Key Rotation

Versions are **monotonically increasing**: `VAULT_MASTER_KEY` always holds the current (highest) version; `VAULT_MASTER_KEY_1` the previous, `_2` the one before that, etc.

To rotate:
1. Generate new 32-byte base64url key: `python -c "import base64,secrets; print(base64.urlsafe_b64encode(secrets.token_bytes(32)).decode())"`
2. Set `VAULT_MASTER_KEY_1` = current key, `VAULT_MASTER_KEY` = new key
3. Run `alphakey rotate-master-key` — re-wraps all credential DEKs and signing key PEMs from old to new
4. Once complete, remove `VAULT_MASTER_KEY_1` (and older `_N` vars if present)

> [!note] `rotate-master-key` is idempotent
> Rows already on the current version are skipped. Safe to re-run. Use `--dry-run` to preview.

---

## No Override Chain

alphaKey has no runtime config overrides. All config is static environment variables. Token TTLs and service tokens require restart to change.
