---
service: alphaFrame
page: Architecture
tags:
  - service/alphaFrame
  - architecture
---

# alphaFrame — Architecture

[[services/alphaFrame/alphaFrame|alphaFrame]] · [[services/alphaFrame/Interactions|Interactions]] · [[services/alphaFrame/API|API]] · [[services/alphaFrame/Data|Data]] · [[services/alphaFrame/Config|Config]]

---

## Purpose

alphaFrame eliminates per-project infrastructure duplication. Previously alphaGen and alphaTrade each ran their own Postgres/Redis/MinIO — causing port conflicts and model artifacts not being visible cross-project. alphaFrame provides one shared `platform` Docker network with all stateful services, one MLflow instance for experiment tracking, and a full LGTM observability stack.

---

## Service Groups

### Core Stateful Infrastructure

| Container | Image | Purpose |
|---|---|---|
| `postgres` | `postgres:16-alpine` | Primary SQL store for all services |
| `redis` | `redis:7-alpine` | Celery broker/results, pub/sub, token denylist |
| `minio` | `minio/minio` | S3-compatible object store for model artifacts |
| `minio-init` | `minio/mc` | One-shot bucket creation (models, trades, mlflow) |
| `mlflow` | custom (`mlflow:v3.13.0` + psycopg2 + boto3) | Experiment tracking + model registry |
| `nginx` | custom (nginx:alpine + self-signed TLS) | TLS termination + rate-limited reverse proxy |

### Application Services (joined to platform network)

| Container | Source | Purpose |
|---|---|---|
| `alphakey-api` | `../alphaKey` | Auth / vault service |
| `alphagen-api` | `../alphaGen` | Training API (runs Alembic upgrade at startup) |
| `alphagen-worker` | `../alphaGen` | Celery worker (concurrency=1) |
| `alphagen-flower` | `mher/flower` | Celery monitoring UI |
| `alphatrade` | `../alphaTrade` | Trading executor |
| `alphalink` | `../alphaLink` | Next.js frontend |

### Observability Stack (LGTM)

| Container | Image | Purpose |
|---|---|---|
| `otel-collector` | otel/opentelemetry-collector-contrib:0.103.0 | Receives OTLP traces/metrics/logs from apps; routes to backends |
| `prometheus` | prom/prometheus:v2.52.0 | Metrics storage (15d), alert evaluation |
| `loki` | grafana/loki:3.1.0 | Log aggregation (7d) |
| `tempo` | grafana/tempo:2.5.0 | Trace storage (7d), service graph |
| `alertmanager` | prom/alertmanager:v0.27.0 | Alert routing (webhook stub → Slack/PagerDuty ready) |
| `grafana` | grafana/grafana:11.1.0 | Unified UI for all telemetry |
| `node-exporter` | prom/node-exporter:v1.8.1 | Host CPU/memory/disk metrics |
| `cadvisor` | gcr.io/cadvisor/cadvisor:v0.49.1 | Per-container resource metrics |
| `postgres-exporter` | prometheuscommunity/postgres-exporter | PostgreSQL metrics |
| `redis-exporter` | oliver006/redis_exporter | Redis metrics |

---

## Startup Sequence

```mermaid
sequenceDiagram
    participant DC as Docker Compose
    participant PG as postgres
    participant RD as redis
    participant MN as minio
    participant MI as minio-init
    participant ML as mlflow
    participant AK as alphakey-api
    participant AG as alphagen-api
    participant AT as alphatrade
    participant OC as otel-collector

    DC->>PG: Start (healthcheck: pg_isready)
    DC->>RD: Start (healthcheck: redis-cli ping)
    DC->>MN: Start (healthcheck: mc ready local)
    DC->>OC: Start (always-on)
    PG-->>DC: healthy
    MN-->>DC: healthy
    DC->>MI: Run init.sh (creates buckets)
    MI-->>DC: service_completed_successfully
    PG-->>DC: healthy
    DC->>ML: Start (needs postgres + minio-init)
    ML-->>DC: healthy
    DC->>AK: Start (needs postgres + redis + minio-init + otel-collector)
    DC->>AG: Start (needs postgres + redis + mlflow + minio-init)
    DC->>AT: Start (needs postgres + redis + mlflow + minio)
```

---

## Observability Pipeline

```mermaid
flowchart LR
    subgraph Apps
        AK[alphaKey]
        AG_A[alphaGen API]
        AG_W[alphaGen Worker]
        AT[alphaTrade]
    end

    subgraph OTelCol["OTel Collector"]
        R_OTLP["otlp receiver :4317/:4318"]
        R_FILE["filelog receiver (Docker container logs)"]
        P_BATCH["batch processor"]
        P_RES["resource processor (add env=local)"]
        E_TEMPO["otlp/tempo exporter :4317"]
        E_PROM["prometheusremotewrite :9090/api/v1/write"]
        E_LOKI["otlphttp/loki :3100/otlp"]
    end

    Apps -->|OTLP gRPC :4317| R_OTLP
    R_FILE -->|Docker logs| P_RES
    R_OTLP --> P_RES --> P_BATCH
    P_BATCH --> E_TEMPO & E_PROM & E_LOKI

    E_TEMPO --> Tempo[Tempo :3200]
    E_PROM --> Prometheus[Prometheus :9090]
    E_LOKI --> Loki[Loki :3100]

    Prometheus -->|eval alerts| Alertmanager[Alertmanager :9093]
    Alertmanager -->|webhook| Stub["localhost:9999 (stub)"]
    Grafana[Grafana :3001] --> Prometheus & Loki & Tempo
```

---

## Nginx Reverse Proxy Architecture

All external traffic enters via Nginx on ports 80/443. Port 80 redirects to HTTPS.

| Route | Upstream | SSE-safe? | Rate-limited? |
|---|---|---|---|
| `/auth/` | `alphakey-api:8000/auth/` | No | Yes |
| `/alphagen/` | `alphagen-api:8000/` | No | Yes |
| `/alphagen/runs/events` | `alphagen-api:8000/runs/events` | ✅ (proxy_buffering off, 3600s) | No |
| `/stream` | `alphatrade:8081/stream` | ✅ (proxy_buffering off, 3600s) | No |
| `/` (default) | `alphatrade:8081` | No | Yes |

> [!note] Internal network only
> `/auth/internal/*` is NOT exposed through Nginx — service-to-service only.

---

## Key Design Decisions

- **Named `platform` network**: All services join `platform` bridge; app projects declare `networks: { platform: { external: true } }` — service names resolve as DNS hostnames.
- **Single Postgres instance, per-project databases**: One `platform` user owns `alphagen`, `alphatrade`, `mlflow`, `alphakey` databases — isolation without separate instances.
- **MLflow with no artifact serving** (`--no-serve-artifacts`): Clients fetch model files directly from MinIO S3 — reduces MLflow load, preserves MinIO as artifact source of truth.
- **OTel Collector pattern**: Single collector aggregates all telemetry; only exporter config changes for cloud migration (Grafana Cloud, Datadog, etc.) — zero app instrumentation changes needed.
- **Exemplars end-to-end**: `TraceBasedExemplarFilter` in apps → OTel Collector preserves exemplar metadata → Prometheus → Grafana links metric data points to traces.

---

*See [[platform/Key-Decisions]] for full ADRs on shared infra and observability.*
