---
service: alphaTest
status: in-progress
stack: pytest + pytest-playwright (Python 3.11/3.12)
owner: ""
last-reviewed: 2026-06-15
tags:
  - service/alphaTest
  - status/in-progress
---

# alphaTest

> Cross-service regression suite — validates trading behaviour, signal quality, API contract compatibility, and auth security across all platform services.

**Status:** 🟢 Epic complete — all priority-1 tests implemented; dispatch wiring live; E2E not yet verified (Actions minutes exhausted, bead `alphaTest-7qi`)
**Repo path:** `projectAlpha/alphaTest/`
**Stack:** pytest + pytest-playwright. One runner covers REST/Redis/MinIO/JWT and full-stack browser journeys. Matches 3/4 backend services.
**UI scope:** Cross-service full-stack journeys only (U-suite). Isolated alphaLink component tests stay in alphaLink (Jest/Playwright).

---

## Implementation Status (2026-06-15)

| Bead | Deliverable | Status |
|---|---|---|
| `alphaTest-780.1` | Scaffold: pyproject.toml, conftest, Makefile, dir tree, CI workflows | ✅ Done |
| `alphaTest-780.2` | Golden artifacts: OHLCV parquet, ONNX models, schema snapshots, TOTP vectors | ✅ Done |
| `alphaTest-780.3` | `servers/mock_t212/` — stateful FastAPI broker mock for L-suite | ✅ Done |
| `alphaTest-780.4` | `[CON]` PR-gate contract tests: D3, T3, L16, token-claims, A4, A11, BFF-auth | ✅ Done |
| `alphaTest-780.5` | `[INT/P1]` L1: SELL closes actual position, never creates short | ✅ Done |
| `alphaTest-780.6` | `[INT/P1]` L2: OCO cancelled on signal exit, no orphaned orders | ✅ Done |
| `alphaTest-780.7` | `[INT/P1]` L12: Broker resilience — 429/5xx/401/event-loop jitter | ✅ Done |
| `alphaTest-780.8` | `[E2E/P1]` A1: Full login → refresh → logout flow | ✅ Done |
| `alphaTest-780.9` | `[E2E/P1]` V2: KEK rotation — strict xfail until alphaKey-5fv fixed | ✅ Done |

**Current test count:** 28 tests (20 contract + 8 INT/E2E). `make contract` passes with no running services. INT tests pass with mock_t212 in-process (no stack).

> **Pending verification (bead `alphaTest-7qi`):** GitHub Actions minutes exhausted 2026-06-15. The full `repository_dispatch` loop (sibling PR → notify-alphatest → alphaTest pr-gate) must be tested when minutes reset (monthly). See [[#CI Secrets & PAT]] for what to verify.

---

## Repository Layout

```
alphaTest/
  pyproject.toml            # hatchling build; pytest markers; ruff/mypy; coverage fail_under=70
  Makefile                  # contract / smoke / int / e2e / slow / all
  conftest.py               # opt-in flags (--run-integration/e2e/slow/golden); platform fixtures
  README.md                 # tier table; env vars; scenario→file map; header convention
  tests/
    contract/               # [CON] schema-snapshot tests — no stack; PR gate
      test_d3_t3_schemas.py # D3 (model.ready schema) + T3 (manifest=ONNX contract)
      test_l16_alphakey_contracts.py  # L16, token-claims, A4, A11, BFF-auth
    smoke/                  # post-deploy health (pending)
    integration/            # [INT] two-service tests (pending)
    e2e/                    # [E2E] full-stack incl. Playwright (pending)
    golden/                 # reproducibility baselines (pending)
    unit_r/                 # [UNIT-R] pinned-bug regressions (pending)
  fixtures/
    ohlcv/
      daily_AAPL.parquet    # 252 trading days 2023, rng seed=42
      intraday_AAPL_5m.parquet  # 78 bars 2023-01-03, rng seed=42
      generate.py           # regeneration script (do not change seed)
    models/
      mlp/manifest.json     # arch=mlp, features=[sma_5,sma_20,rsi_14,macd,volume_ratio]
      mlp/model.onnx        # tiny 2-layer MLP; fixed_input=[0.1..0.5]; expected_logits known
      lstm/manifest.json    # arch=lstm, same features, seq_len=5
      lstm/model.onnx       # tiny LSTM; same fixed_input; expected_logits known
    schemas/
      model_ready_event.json      # new-format model.ready payload (D3 source of truth)
      manifest_schema.json        # JSONSchema: feature_names/n_features/arch required
      alphakey_token_claims.json  # JWT claim keys + types (iss/aud/sub/role/jti/tv/kid/iat/exp)
      bff_auth_response.json      # BFF login shape: access_token in body; refresh_token httpOnly
    totp_vectors.json       # RFC 6238: secret JBSWY3DPEHPK3PXP; 3 codes pinned at t=1686816000
  servers/
    mock_t212/
      app.py                # FastAPI stateful broker mock (orders/positions/fills/errors)
      fixture.py            # pytest fixture: MockT212Client (in-process ASGI, no subprocess)
  .github/workflows/
    pr-gate.yml             # repository_dispatch 'alphatest-pr-gate': contract tests, Python 3.11+3.12
    post-merge.yml          # push to main: priority1 INT tests via alphaFrame compose
    nightly.yml             # cron 02:00: full INT+E2E
    weekly.yml              # cron Sun 03:00: slow + golden
    manual.yml              # workflow_dispatch: any tier, any Python version
```

---

## Running Tests

```bash
# Install (requires Python 3.11+)
make install

# Contract tests — no stack needed (PR gate)
make contract

# Integration tests (requires alphaFrame compose)
docker compose -f ../alphaFrame/docker-compose.yml up -d
PLATFORM_URL=http://localhost make int

# Full suite
PLATFORM_URL=http://localhost make all
```

### Required Environment Variables

| Variable | Default | Tier |
|---|---|---|
| `PLATFORM_BASE_URL` | `http://localhost` | smoke / int / e2e fixtures |
| `SERVICE_TOKEN_SECRET` | — | `alphakey_service_token` fixture |
| `REDIS_HOST` / `REDIS_PORT` | `redis` / `6379` | `redis_client` fixture |
| `MINIO_ENDPOINT` / `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | minio defaults | `minio_client` fixture |
| `PLATFORM_URL` | — | Makefile live targets |

---

## Pytest Markers

| Marker | Meaning | Opt-in flag |
|---|---|---|
| `con` | `[CON]` contract/schema-snapshot — no stack | always run |
| `int` | `[INT]` two-service integration | `--run-integration` |
| `e2e` / `ui` | `[E2E]` full stack / Playwright browser | `--run-e2e` |
| `smoke` | post-deploy health | always run on live tiers |
| `golden` | reproducibility baseline | `--run-golden` |
| `slow` | long sweeps — weekly only | `--run-slow` |
| `priority1` | money-losing / currently-failing (L1,L2,L12,L16,A1,V2) | subset flag |
| `unit_r` | `[UNIT-R]` pinned-bug unit regression | always run |

---

## Contract Tests (implemented)

### D3 — model.ready schema (`test_d3_t3_schemas.py`)
- Snapshot `fixtures/schemas/model_ready_event.json` has new-format fields (`run_name`, `version`, `published_at`)
- Old-format payload (`type`, `minio_bucket`, `minio_path`, `manifest_path`) is silently dropped by alphaTrade `ModelSyncDaemon`
- Unknown extra fields tolerated (forward compat)
- New-format payload triggers `_sync_once` / `_sync_from_minio` (not no-op)

### T3 — manifest = ONNX inference contract (`test_d3_t3_schemas.py`)
- `len(manifest["feature_names"]) == manifest["n_features"] == ONNX input shape[-1]` for MLP and LSTM
- `fixtures/schemas/manifest_schema.json` validates both golden manifests

### L16 — API auth enforced (`test_l16_alphakey_contracts.py`) `priority1`
- All 5 `/api/v1/*` routes (`kill_switch`, `settings`, `orders`, `positions`, `account`) return 401 with no credentials
- `ALPHATRADE_INSECURE_NO_AUTH=true` bypasses auth only when **explicitly** set; default is fail-closed

### alphaKey token-claims snapshot (`test_l16_alphakey_contracts.py`)
- Fixture declares all required claim keys with correct types
- JWT issued via `alphakey.security.tokens.issue_access_token` → verified JWT contains full snapshot key set

### A4 — fail_closed not client-controllable (`test_l16_alphakey_contracts.py`)
- `is_denied(fail_closed=True)` → denies when Redis down
- `is_denied(fail_closed=False)` → allows when Redis down
- `LogoutRequest` schema has no `fail_closed` field (client cannot inject)

### A11 — logout jti from token not body (`test_l16_alphakey_contracts.py`)
- `LogoutRequest` fields == `{"refresh_token"}` only; no `jti` or `exp`
- `logout()` source confirmed to use `current_user.jti`, not `body.jti`

### BFF auth response shape (`test_l16_alphakey_contracts.py`)
- `access_token` present in body; `refresh_token` absent from body (httpOnly only)
- `Set-Cookie` header for refresh_token has `HttpOnly` + `Secure` flags

---

## Pact Consumer-Driven Contract Testing

Separate from the schema-snapshot `[CON]` tests above — a parallel rollout of Pact consumer-driven contracts between sibling services. Each Pact contract is generated by running the consumer's real code against a mock provider, then verified against a live provider in CI.

**Handoff mechanism:** file-based (no persistent Pact Broker hosting yet — broker runs locally via alphaFrame compose but is not wired into CI). Pact files are committed to the provider repo's `tests/pacts/` directory.

### Rollout Status (2026-06-16)

| Sub-project | Consumer | Provider | Interactions | Status |
|---|---|---|---|---|
| 1 — Pact Broker infra | — | — | Pact Broker + Postgres in alphaFrame compose (`:9292`) | ✅ Done |
| 2 — alphaLink → alphaKey | alphaLink (TS) | alphaKey (Python) | Login/refresh/logout/me (9 interactions) | ✅ Done |
| 3 — alphaGen + alphaTrade → alphaKey | alphaGen (Python), alphaTrade (Python) | alphaKey (Python) | JWKS, token-version, secrets (5 interactions each) | ✅ Done |
| 4 — alphaLink → alphaTrade | alphaLink (TS) | alphaTrade (Python) | Kill-switch, halt, resume, promote×2, demote×2 (7 interactions) | ✅ Done |
| 5 — alphaLink → alphaGen | alphaLink (TS) | alphaGen (Python) | Jobs/runs endpoints | ✅ Done |

### Provider Verification Pattern

Each provider (alphaKey, alphaTrade) runs a `pact-verify` CI job that:
1. Starts a live `uvicorn` server with `PACT_VERIFICATION_MODE=true` (mounts gated `/internal/pact-state` endpoint)
2. Applies any required patches (JWKS mock, MlflowClient fake, etc.) to avoid needing live dependencies
3. Runs `pact-python`'s `Verifier` against the committed pact file
4. `notify-alphatest` depends on `pact-verify` — Pact failures block alphaTest PR gate dispatch

### Known Deferred Items

- `POST /api/v1/models/{name}/fork` (alphaTrade) — no alphaLink caller exists yet; tracked in [[ToDo/Cross-Service-Backlog]] §9.
- Automating pact file handoff to a persistent broker — blocked on hosting; tracked in alphaFrame.

---

## Golden Artifacts

### OHLCV Parquet (`fixtures/ohlcv/`)
Frozen synthetic data — deterministic, never regenerate from live feeds.

| File | Content | Seed |
|---|---|---|
| `daily_AAPL.parquet` | 252 trading days, 2023-01-03–2023-12-29, columns: open/high/low/close/volume | `np.random.default_rng(42)` |
| `intraday_AAPL_5m.parquet` | 78 bars, 2023-01-03 09:30–13:25, 5-min freq | same seed |

### ONNX Models (`fixtures/models/`)
Tiny models (functional, not accurate) for T3 contract and L-suite loading tests.

| Arch | Input shape | Output | Seed |
|---|---|---|---|
| `mlp` | `(batch, 5)` | `(batch, 2)` logits | `torch.manual_seed(0)` |
| `lstm` | `(batch, 5, 5)` | `(batch, 2)` logits | same |

Fixed input `[0.1, 0.2, 0.3, 0.4, 0.5]`; expected logits recorded in each `manifest.json`.

### Schema Snapshots (`fixtures/schemas/`)

| File | Source of truth for |
|---|---|
| `model_ready_event.json` | D3 — new-format model.ready payload |
| `manifest_schema.json` | T3 — JSONSchema for alphaGen manifest.json |
| `alphakey_token_claims.json` | alphaKey JWT claim keys + types |
| `bff_auth_response.json` | BFF login response shape + cookie flags |

### TOTP Vectors (`fixtures/totp_vectors.json`)
Secret: `JBSWY3DPEHPK3PXP` (RFC 6238 well-known test secret). Three 30-second windows pinned at `t=1686816000` (2023-06-15 12:00:00 UTC). Use `freezegun` to pin the clock in A12/A13 MFA tests.

---

## mock_t212 Broker Server (`servers/mock_t212/`)

Stateful FastAPI server — single source of truth for the entire L-suite (L1–L18).

### T212 API endpoints simulated

| Method | Path | Behaviour |
|---|---|---|
| `POST` | `/api/v0/equity/orders/market` | Immediate FILL, updates positions |
| `POST` | `/api/v0/equity/orders/stop` | Creates PLACED order (no fill) |
| `POST` | `/api/v0/equity/orders/limit` | Creates PLACED order (no fill) |
| `GET` | `/api/v0/equity/orders/{id}` | Returns order status |
| `DELETE` | `/api/v0/equity/orders/{id}` | Sets status=CANCELLED |
| `GET` | `/api/v0/portfolio/positions` | Returns in-memory position list |
| `GET` | `/api/v0/equity/account/summary` | Returns `{cash, totalValue}` |

### Control API (tests only)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/control/reset` | Clear all state between tests |
| `POST` | `/control/set-position` | Seed a position `{ticker, quantity, avgPrice}` |
| `POST` | `/control/inject-error` | Inject 429/5xx/401 `{status, after, retry_after}` |
| `POST` | `/control/clear-error` | Remove error injection |
| `POST` | `/control/set-latency` | Artificial latency `{latency_s}` for jitter tests |
| `POST` | `/control/fill-order` | Force-fill a PLACED order (OCO tests) |
| `GET` | `/control/orders-placed` | Audit log of all calls + current order state |

### Usage in tests

```python
from servers.mock_t212.fixture import mock_t212  # pytest fixture

async def test_sell_closes_position(mock_t212):
    await mock_t212.set_position("AAPL_EQ", quantity=10.0, avg_price=155.0)
    # ... drive alphaTrade ...
    positions = await mock_t212.get_positions()
    assert positions == []
```

The fixture uses `httpx.ASGITransport` — no subprocess, no ports, instant startup.

---

## CI Workflows

| Workflow | Trigger | Runs | Stack |
|---|---|---|---|
| `pr-gate.yml` | `repository_dispatch: alphatest-pr-gate` | contract tests, Python 3.11 + 3.12 | none |
| `post-merge.yml` | push to `main` | `priority1` INT subset | alphaFrame compose |
| `nightly.yml` | cron 02:00 daily | full INT + E2E | alphaFrame compose |
| `weekly.yml` | cron 03:00 Sunday | slow + golden | alphaFrame compose |
| `manual.yml` | `workflow_dispatch` | any tier, any Python version | optional |

**Sibling repo trigger:** sibling repos dispatch `alphatest-pr-gate` via `ALPHATEST_DISPATCH_TOKEN` PAT. alphaTest branch protection lists `alphatest-pr-gate` as a required check.

---

## CI Secrets & PAT

### Required secrets per repo

| Repo | Secret | Value |
|---|---|---|
| alphaTrade | `ALPHATEST_DISPATCH_TOKEN` | Fine-grained PAT (see below) |
| alphaKey | `ALPHATEST_DISPATCH_TOKEN` | Fine-grained PAT |
| alphaLink | `ALPHATEST_DISPATCH_TOKEN` | Fine-grained PAT |
| alphaTest | `ALPHATEST_DISPATCH_TOKEN` | Fine-grained PAT |
| alphaTest | `SERVICE_TOKEN_SECRET` | alphaKey service token for alphatest principal (see alphaFrame/.env `ALPHAKEY_SERVICE_TOKEN_TEST`) |

### PAT (`ALPHATEST_DISPATCH_TOKEN`) — ⚠️ EXPIRES 2027-01-01

**Scopes required:**
- Read access on `alphaTrade` and `alphaKey` repos (so alphaTest CI can `pip install --no-deps` them via `git+https`)
- Actions write on `alphaTest` repo (so sibling CIs can POST to `/dispatches`)
- alphaLink does NOT need to be in the PAT's repo access list

**Renewal — do this before 2027-01-01:**
1. GitHub → Settings → Developer settings → Fine-grained personal access tokens
2. Find `ALPHATEST_DISPATCH_TOKEN` → Regenerate (or create new with same scopes)
3. Update the secret in all four repos: alphaTrade, alphaKey, alphaLink, alphaTest (Settings → Secrets and variables → Actions)

### SERVICE_TOKEN_SECRET

Token value lives in `alphaFrame/.env` as `ALPHAKEY_SERVICE_TOKEN_TEST`. alphaKey reads it via `ALPHAKEY_SERVICE_TOKENS` env var (format `token:principal`). Tests pass it as `X-Service-Token` header — alphaKey grants `alphatest` principal permissions. Only needed for E2E tests (A1, V2) that call alphaKey admin API.

---

## Architecture Decisions

### Mock Broker Strategy

**Decision (2026-06-15):** Broker-specific mocks, not a generic interface.

**Rationale:** alphaTrade couples directly to the T212 API — no broker abstraction layer exists. A generic mock would invent a hypothetical interface. Each broker gets its own mock that matches its actual API response shapes exactly.

**Future:** When a `BrokerClient` ABC is introduced in alphaTrade, add `servers/mock_<broker>/` per new broker. Fixture injection stays via env var (`T212_API_URL`) — zero rework on existing tests.

### Sibling Service Imports

**Decision (2026-06-15):** Replaced `sys.path` filesystem coupling with explicit PEP 508 git-reference dependencies. See [[ToDo/Cross-Service-Backlog]] for Phase 2 (private registry) backlog.

Siblings declared in `pyproject.toml [siblings]` as `alphaTrade @ git+https://github.com/Preeth2000/alphaTrade.git@main`. CI installs with `--no-deps` to skip `torch`/`ta-lib`/`onnxruntime`; runtime libs needed at import time (`fastapi`, `sqlmodel`, etc.) declared explicitly in `[dev]` extra.

**pr-gate SHA semantics:** the `repository_dispatch` `client_payload` carries `repo` and `sha`. `pr-gate.yml` force-reinstalls the dispatching sibling at the exact PR commit — guarantees tests run against the code under review, not `main`.

---

## Test Catalogue

> Full scenario catalogue: [[Regression-Scenarios|Regression Scenarios]]

---

*Beads epic: `alphaTest-780` — see `bd show alphaTest-780` for full issue tree.*
