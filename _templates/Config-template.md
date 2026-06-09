---
service: "{{SERVICE_NAME}}"
page: Config
tags:
  - service/{{service_slug}}
  - config
---

# {{SERVICE_NAME}} — Config

[[{{SERVICE_NAME}}]] · [[Architecture]] · [[Interactions]] · [[API]] · [[Data]]

---

## Environment Variables

> Source: `.env.example` — copy to `.env` to run locally.

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `VAR_NAME` | `default` | ✅ / ⬜ | Description |

---

## Config Files

| File | Format | Purpose | Loaded by |
|---|---|---|---|
| `overrides.yaml` | YAML | Runtime trading config | Main app startup |
| `configs/validation.yaml` | YAML | Validation gate thresholds | Config loader |

### `overrides.yaml` structure

```yaml
# describe structure here
defaults:
  key: value
```

---

## Override Priority Chain

> [!info] Priority (highest → lowest)
> 1. **DB record** (authoritative at runtime — survives restart)
> 2. **Environment variable** (`.env`)
> 3. **Config file** (YAML seed — applied at startup)
> 4. **Code default** (Pydantic field default)

Describe the merge logic: when YAML seeds DB, when DB overrides YAML, what triggers re-merge.

---

## Feature Flags / Runtime Toggles

| Flag / Setting | Default | Effect | Changed via |
|---|---|---|---|
| `enabled` per model | `true` | Skip model in scheduler tick | DB / API |

---

*See [[platform/Key-Decisions]] for why DB-first override was chosen.*
