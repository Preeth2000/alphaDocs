# README Template — projectAlpha services

<!-- GitHub root README template.
     Fill placeholders ({{...}}), drop sections that don't apply.
     Source content from alphaDocs/alphaDocs/services/<name>/ and reference/.
     Style reference: alphaGen/README.md (table density, fenced-block convention). -->

---

# {{ServiceName}}

> {{One-line purpose — what this service is and what it does.}}

**Stack:** {{e.g. Python 3.11, FastAPI, PostgreSQL}} · **Port:** {{port or "no app port"}} · **Repo:** `projectAlpha/{{ServiceName}}/`

---

## What it is

2–4 sentences describing:
- Role in the platform (upstream/downstream)
- What problem it solves
- Key peers — link siblings with relative paths e.g. [alphaGen](../alphaGen)

---

## How it works

Core mechanism described concisely. Include a pipeline or architecture snippet:

```
ComponentA → ComponentB → ComponentC → Output
```

Call out important design invariants (e.g. "fail-closed auth", "walk-forward splits only", "kill-switch blocks orders not inference").

---

## Prerequisites

System-level or language-version requirements before install:

```bash
# Example system dep
sudo apt-get install -y libta-lib-dev
```

- Python ≥ {{version}} / Node ≥ {{version}}
- Docker + Docker Compose (if applicable)

---

## Install

```bash
# Python services
pip install -e ".[dev]"

# Node services
npm install
```

---

## Usage

Quick-start: get the service running in the fewest steps.

```bash
cp .env.example .env
# edit .env
docker compose up -d --build
```

For CLI-driven services, show the most common commands:

```bash
{{cli-name}} {{command}} --flag value
```

---

## Configuration

Copy `.env.example` → `.env`. Required vars must be set before first start.

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `VAR_NAME` | — | yes | Description |
| `VAR_NAME` | `value` | no | Description |

{{Note any config-file override chain if one exists, e.g.:}}
> Override priority: per-record DB override → BotSettings DB → `overrides.yaml` → Pydantic defaults

---

## API

> All routes require `Authorization: Bearer <jwt>` unless noted.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/health` | none | Liveness check |
| `POST` | `/example` | JWT | Description |

---

## Architecture & Data

**Databases / stores:**

| Store | Name/Path | Key tables / buckets |
|---|---|---|
| PostgreSQL | `{{db_name}}` | `table_a`, `table_b` |
| Redis | db{{n}} | Purpose |
| MinIO | `{{bucket}}` | Contents |

**Events / channels:**

| Channel | Direction | Payload |
|---|---|---|
| `model.ready` | subscribe | `{run_name, version}` |

---

## Tests

```bash
# Unit
pytest tests/unit -v

# Contract (no stack needed)
pytest tests/contract -v

# Integration
pytest tests/integration -v
```

### Contract tests (Pact)

{{ServiceName}} is a **{{consumer|provider}}** of {{OtherService}} ({{what for}}).

```bash
# Consumer / provider verification command
pytest tests/contract/test_{{name}}_pact.py -v
```

> `PACT_VERIFICATION_MODE` must be `false` outside of Pact verification runs.

---

## Related

| Service | Role |
|---|---|
| [alphaFrame](../alphaFrame) | Shared infra (Postgres, Redis, MinIO, Nginx) |
| [alphaKey](../alphaKey) | Auth provider |
| [alphaGen](../alphaGen) | Model trainer |
| [alphaTrade](../alphaTrade) | Trading executor |
| [alphaLink](../alphaLink) | Web UI |
| [alphaTest](../alphaTest) | Cross-service test suite |

Full documentation: [`alphaDocs`](../alphaDocs)
