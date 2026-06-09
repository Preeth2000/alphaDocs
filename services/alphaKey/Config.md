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
| `REDIS__URL` | `redis://redis:6379/0` | ✅ | Redis for JTI denylist |
| `VAULT_MASTER_KEY` | — | ✅ | 32-byte base64 KEK. **Service refuses to start if missing.** |
| `VAULT_MASTER_KEY_2`, `_3`, … | — | ⬜ | Rotated-out key versions (for DEK re-wrapping) |
| `ALPHAKEY_SERVICE_TOKENS` | — | ✅ | Comma-separated `token:principal_name` pairs for service-to-service auth |
| `ALPHAKEY_SERVICE_TOKEN_TRADE` | — | ⬜ | alphaTrade's service token (subset of SERVICE_TOKENS) |
| `ALPHAKEY_SERVICE_TOKEN_GEN` | — | ⬜ | alphaGen's service token |
| `ACCESS_TOKEN_TTL` | `600` | ⬜ | JWT access token TTL (seconds, default 10 min) |
| `REFRESH_TOKEN_TTL` | `1209600` | ⬜ | Refresh token TTL (seconds, default 14 days) |
| `ALPHAKEY_PORT` | `8000` | ⬜ | API listen port |
| `OTEL_SERVICE_NAME` | `alphakey` | ⬜ | OTel service name |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://otel-collector:4317` | ⬜ | OTel Collector endpoint |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | ⬜ | OTLP protocol |

---

## Service Token Format

`ALPHAKEY_SERVICE_TOKENS` is a comma-separated list of `token:principal` pairs:

```
ALPHAKEY_SERVICE_TOKENS=abc123:alphatrade,def456:alphagen
```

Each token grants access to the `/auth/introspect` and `/auth/internal/secrets/*` endpoints. The principal name is used for `CredentialAccessAudit` logging (`svc:alphatrade`).

---

## Vault Key Rotation

To rotate the KEK:
1. Generate new 32-byte base64 key
2. Set `VAULT_MASTER_KEY_2` = old key, `VAULT_MASTER_KEY` = new key
3. Run re-wrapping migration to update `credential.dek_wrapped` and `credential.key_version`
4. Once all DEKs re-wrapped, remove `VAULT_MASTER_KEY_2`

> [!warning] Key rotation not automated yet
> As of 2026-06-06, vault key rotation is manual. No automated rotation tooling exists.

---

## No Override Chain

alphaKey has no runtime config overrides. All config is static environment variables. Token TTLs and service tokens require restart to change.
