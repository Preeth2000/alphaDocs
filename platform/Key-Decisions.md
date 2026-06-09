---
page: Key-Decisions
tags:
  - platform
  - architecture
  - decisions
last-reviewed: 2026-06-06
---

# Key Architecture Decisions

[[README]] · [[platform/Overview]] · [[platform/Features]] · [[platform/Tech-Stack]]

Architecture Decision Records (ADRs) for the major design choices across projectAlpha.

---

## Communication Patterns

### ADR-001: REST + SSE over WebSocket

**Context:** alphaLink needs real-time training log streaming and live trading updates. Options were WebSocket, polling, or SSE.

**Decision:** Use HTTP REST for all synchronous operations; SSE for real-time push from server to client.

**Rationale:**
- SSE is unidirectional (server→client only) — sufficient for all real-time use cases (logs, tick events, model.ready notifications)
- SSE works over standard HTTP/1.1 — no upgrade handshake, simpler Nginx proxying (just disable buffering + extend timeout)
- REST + SSE fits natural request/response + stream separation; no bidirectional needs
- Simpler client implementation (EventSource API) vs WebSocket

**Consequences:** alphaTrade's SSE `/stream` endpoint and alphaGen's `/runs/{id}/log` both use Redis pub/sub as the backing channel — allows multiple API instances to serve the same streams.

---

### ADR-002: Celery + Redis for Async Training Jobs

**Context:** ML training jobs take minutes to hours. The API must return immediately while work continues.

**Decision:** FastAPI enqueues a Celery task on `POST /runs`; Celery worker (`concurrency=1`) runs `att.train_and_export`; results stream back via Redis pub/sub.

**Rationale:**
- `concurrency=1` prevents GPU/CPU contention — training jobs queue, not compete
- Redis dual-purpose: Celery broker (db0) + pub/sub (same db0) — reduces infrastructure
- Celery revoke on `DELETE /runs/{id}` enables clean cancellation
- Stale job detection (30min running / 5min queued → auto-fail) prevents zombie jobs

**Consequences:** Only one training job runs at a time. Horizontal scaling requires increasing `concurrency` or running multiple workers.

---

### ADR-003: MinIO as Artifact Transport (not MLflow)

**Context:** alphaTrade needs to download trained models. Options: serve from MLflow, fetch from MinIO directly, or use a registry API.

**Decision:** alphaGen publishes `model.onnx` + `manifest.json` to MinIO. alphaTrade downloads directly from MinIO. MLflow tracks metadata and aliases but does NOT serve artifacts (`--no-serve-artifacts`).

**Rationale:**
- MinIO is a persistent S3-compatible store — artifacts available indefinitely at known paths
- MLflow's `no-serve-artifacts` keeps it lightweight (no streaming load)
- Binary downloads are fast from S3; MLflow API is for metadata only
- Versioned paths + `latest` pointer enable both explicit-version and polling consumers
- S3 protocol is universal — easy to swap MinIO for AWS S3 in production

**Consequences:** alphaTrade must know MinIO credentials. Model manifest is the contract — alphaTrade reads it to reconstruct the exact feature pipeline used during training.

---

### ADR-004: Redis pub/sub for model.ready (not direct SSE call)

**Context:** alphaGen needs to notify alphaTrade when a new model is published. Options: alphaTrade polls, direct HTTP push, or event-based.

**Decision:** alphaGen publishes `model.ready` to Redis pub/sub channel. alphaTrade subscribes (via `GET /runs/events` SSE) on startup. alphaLink browser also subscribes via the same SSE endpoint.

**Rationale:**
- Decoupled — alphaGen does not need to know alphaTrade's address
- Multiple consumers possible (alphaTrade + future services) with no code change
- Redis is already in use; no new infrastructure
- SSE endpoint on alphaGen serves as the bridge: Redis pub/sub → HTTP SSE

**Consequences:** alphaTrade must be running and connected to the SSE endpoint to receive events. ModelSyncDaemon also polls MinIO periodically (60s) as a fallback.

---

## Security

### ADR-005: ES256 JWT with JWKS (not HMAC)

**Context:** alphaKey issues tokens consumed by alphaGen + alphaTrade for offline verification. HMAC-SHA256 (HS256) requires shared secret; ES256 (ECDSA P-256) uses asymmetric keys.

**Decision:** ES256 JWT with JWKS endpoint for public key distribution.

**Rationale:**
- Asymmetric keys: services only need the public key to verify — no shared secret to protect
- JWKS standard endpoint (`/.well-known/jwks.json`) enables key rotation without service restarts
- `kid` claim in JWT allows multi-key JWKS (retiring old key gracefully)
- ES256 signatures are small (64 bytes) and fast

**Consequences:** alphaGen + alphaTrade cache JWKS with 5-minute TTL. Key rotation requires alphaKey to serve both old and new keys during transition. Private key encrypted at rest (Fernet + KEK).

---

### ADR-006: httpOnly Refresh Cookie (not localStorage)

**Context:** alphaLink needs to persist the refresh token so users don't have to log in on every page load.

**Decision:** Refresh token stored in httpOnly cookie; access token stored in-memory only (Zustand, not localStorage).

**Rationale:**
- httpOnly prevents JavaScript access → XSS cannot steal the refresh token
- Access token in-memory means it's lost on page refresh (intentional) → forces use of refresh cookie which is safe
- `SameSite=Strict` prevents CSRF on the cookie
- BFF pattern means the cookie is only ever read server-side (Next.js API routes)

**Consequences:** Every page load that finds a user in localStorage but no access token calls `POST /api/auth/refresh`. This is the `AuthInitializer` component.

---

### ADR-007: DB-First Overrides in alphaTrade

**Context:** alphaTrade needs runtime config changes (T212 credentials, sizing, risk thresholds) without restarting. `overrides.yaml` is a seed file, not an edit target.

**Decision:** YAML seeds Pydantic `Settings` at startup. `BotSettings` DB record (id=1) is applied on top via `apply_bot_settings()`. Per-model `ModelOverrideRecord` is merged per-tick. DB is authoritative — never write back to YAML at runtime.

**Rationale:**
- DB changes survive restarts; YAML changes do not (without re-deploy)
- API (`PUT /settings`, `PUT /models/{name}/overrides`) provides hot-reload path
- YAML as seed = convenient defaults in version control; DB = operational state
- Prior incident: writing to YAML at runtime (retirement router) caused config drift and test failures → removed

**Consequences:** YAML changes after startup have no effect until restart. All operational changes must go through the API. This is the expected behaviour.

---

## Scaling

### ADR-008: Single-Host Docker Compose

**Context:** projectAlpha is a single-user / small-team platform at current scale. Options: Kubernetes, multi-host Swarm, or single-host Compose.

**Decision:** Docker Compose v2 on a single host with named `platform` network.

**Rationale:**
- Operational simplicity — one `docker compose up` starts everything
- Named network provides service discovery without DNS or service mesh
- ML training (GPU-bound) and live trading (low-latency local) both benefit from co-location
- Per-project DBs and buckets on shared infra (alphaFrame) avoid port conflicts and make MinIO artifacts visible cross-service

**Consequences:** Not horizontally scalable without migration to Compose v3 + Swarm or k8s. Celery worker is `concurrency=1` — acceptable for current training volume.

---

### ADR-009: Shared alphaFrame Infrastructure

**Context:** Before alphaFrame, alphaGen and alphaTrade each ran their own Postgres/Redis/MinIO — causing port conflicts (5432, 6379, 9000) and model artifacts not visible cross-project.

**Decision:** Dedicated alphaFrame project runs all stateful infra on `platform` network. App projects declare it as an external network.

**Rationale:**
- Single Postgres → separate databases per service → no port conflict, shared host, clean isolation
- Single MinIO → shared buckets (`models`, `trades`, `mlflow`) → cross-project artifact visibility
- Single Redis → multiple logical databases (db0 broker, db1 results, shared pub/sub)
- Init scripts idempotent: databases and buckets created once, survive restarts

**Consequences:** alphaFrame must be started before any app service. App repos reference infra by service name on the `platform` network.

---

## Observability

### ADR-010: LGTM Stack with OTel Collector

**Context:** Need unified observability (traces, metrics, logs) without vendor lock-in. Options: Datadog/New Relic (vendor), Grafana Cloud (managed), self-hosted LGTM.

**Decision:** Self-hosted LGTM (Loki + Grafana + Tempo + Prometheus/Mimir) with OTel Collector as the single ingestion point.

**Rationale:**
- OTel SDK in apps is vendor-neutral — only exporter config changes for cloud migration
- Single OTel Collector aggregates all signals: OTLP from apps + Docker filelog scrape
- Exemplar links (metric → trace) enabled end-to-end via TraceBasedExemplarFilter
- Grafana provisioned from config files (GitOps) — no manual dashboard setup
- Data retention: 15d metrics, 7d logs, 7d traces — appropriate for dev/staging

**Consequences:** Local resource usage is significant (10+ containers for observability). For production, replace OTel Collector exporters to point at managed endpoints (Grafana Cloud, etc.) with zero app changes.

---

### ADR-011: alphaTrade Prometheus Metrics on Port 9090

**Context:** alphaTrade needs business-level Prometheus metrics (signals, orders, P&L, T212 latency) scraped by the platform Prometheus.

**Decision:** alphaTrade exposes `prometheus_client` metrics on a dedicated port `9090` — separate from the API port `8081`.

**Rationale:**
- Separate port means metrics scraping doesn't share auth or rate limiting with the API
- Decoupled — Prometheus scrapes `:9090`; users and other services use `:8081`
- Health probe on `:8080` for container readiness — also separate

**Consequences:** alphaTrade opens 3 ports: `8080` (health), `8081` (API), `9090` (metrics).

---

## Model Integrity

### ADR-012: Manifest as Deployment Contract

**Context:** alphaTrade needs to replicate exactly the feature pipeline used during training (same indicators, same normalization, same window size). Options: carry training code, store pipeline config in DB, or use a manifest file.

**Decision:** `manifest.json` published alongside `model.onnx` contains the complete inference contract: `feature_names`, `window`, `normalize`, `norm_stats` (per-column mean/std/min/max), `input_shape`, `model_hash` (SHA-256 of ONNX binary).

**Rationale:**
- Self-contained: alphaTrade needs no training code, no alphaGen running
- `model_hash` verification prevents corrupted download or tampered model from being used
- `norm_stats` enables exact feature normalization replication at inference time
- Version-stable: manifest is immutable per published version

**Consequences:** If alphaGen changes the feature pipeline format, the manifest schema must be versioned. alphaTrade must support reading manifest schema version 1.0.0.
