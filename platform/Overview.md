---
page: Overview
tags:
  - platform
  - overview
last-reviewed: 2026-06-06
---

# Platform Overview

[[README]] · [[platform/Features]] · [[platform/Tech-Stack]] · [[platform/Key-Decisions]]

> projectAlpha is an automated algorithmic trading platform — ML models trained on price data are validated, published, and executed live against Trading 212.

---

## System Architecture

```mermaid
flowchart TD
    subgraph Browser["Browser"]
        UI[alphaLink :3000]
    end

    subgraph Frame["alphaFrame (shared infra)"]
        NGINX[Nginx :443]
        PG[(PostgreSQL :5432)]
        RD[(Redis :6379)]
        MN[(MinIO :9000)]
        ML[MLflow :5000]
        OBS[Observability\nGrafana/Prometheus/Loki/Tempo]
    end

    subgraph Gen["alphaGen :8000"]
        GEN_API[FastAPI]
        GEN_WORK[Celery Worker]
        ATT["att library\n(train/validate/export/publish)"]
    end

    subgraph Trade["alphaTrade :8081"]
        TR_API[FastAPI]
        TR_SCHED[BarClose Scheduler]
        TR_INF[ONNX Inference]
        TR_RISK[Risk Gates]
        TR_BROKER[T212Client]
    end

    subgraph Key["alphaKey :8000"]
        AK_AUTH[Auth / JWT]
        AK_VAULT[Credential Vault]
    end

    subgraph External["External"]
        T212[Trading 212 API]
        YF[yfinance / Polygon.io]
    end

    Browser --> NGINX
    NGINX --> Gen & Trade & Key
    UI -->|BFF| Gen & Trade & Key

    Gen & Trade & Key --> PG & RD
    Gen & Trade --> MN & ML
    Gen & Trade & Key --> OBS

    GEN_API --> GEN_WORK --> ATT
    ATT -->|model.onnx + manifest.json| MN
    MN -->|model sync| TR_SCHED
    RD -->|model.ready pub/sub| TR_API

    TR_SCHED --> TR_INF --> TR_RISK --> TR_BROKER
    TR_BROKER --> T212
    TR_SCHED --> YF
```

---

## Services

| Service | Status | Role | Port |
|---|---|---|---|
| [[services/alphaFrame/alphaFrame\|alphaFrame]] | 🟢 Full | Shared infrastructure — Postgres, Redis, MinIO, MLflow, Nginx, Observability | multiple |
| [[services/alphaGen/alphaGen\|alphaGen]] | 🟢 Full | ML model generation — train, validate, backtest, publish | 8000 |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | 🟢 Full | Trading executor — scheduler, inference, risk, orders | 8081/8080/9090 |
| [[services/alphaLink/alphaLink\|alphaLink]] | 🟢 Full | Frontend + BFF — Next.js UI + proxy | 3000 |
| [[services/alphaKey/alphaKey\|alphaKey]] | 🟡 Partial | Auth + credential vault — JWT, Argon2, Fernet | 8000 |
| [[services/alphaTest/alphaTest\|alphaTest]] | ⬜ Planned | Regression testing | TBD |
| [[services/alphaPerf/alphaPerf\|alphaPerf]] | ⬜ Planned | Performance testing | TBD |

---

## Primary Data Flows

### 1. Model Training → Publishing

```mermaid
sequenceDiagram
    participant U as User (alphaLink)
    participant AG as alphaGen
    participant RD as Redis
    participant ML as MLflow
    participant MN as MinIO
    participant AT as alphaTrade

    U->>AG: POST /runs {config_yaml}
    AG->>RD: enqueue Celery task
    AG-->>U: 202 {run_id}
    U->>AG: GET /runs/{id}/log (SSE)
    RD-->>AG: stream log lines
    AG-->>U: SSE log events
    AG->>ML: log params + metrics
    AG->>MN: model.onnx + manifest.json
    AG->>RD: publish model.ready
    RD-->>AT: model.ready event
    AT->>MN: download model.onnx + manifest.json
    Note over AT: model hot-swapped into registry
```

### 2. Live Trading Tick

```mermaid
sequenceDiagram
    participant SCHED as BarCloseScheduler
    participant DATA as Data Provider
    participant INF as ONNX Inference
    participant RISK as Risk Gates
    participant T212 as Trading 212
    participant DB as Database

    Note over SCHED: NYSE bar close fires
    SCHED->>DATA: fetch OHLCV (warmup bars)
    DATA->>SCHED: DataFrame
    SCHED->>INF: compute features → window → inference
    INF->>RISK: logits per model → consensus signal
    RISK->>DB: check halt / cooldown / positions
    RISK->>T212: place market order (if approved)
    T212-->>DB: order fill → TradeJournal
```

---

## Communication Patterns

| Pattern | Used between | Purpose |
|---|---|---|
| REST (HTTP/JSON) | alphaLink ↔ alphaGen, alphaTrade, alphaKey | Synchronous operations |
| SSE (Server-Sent Events) | alphaGen → alphaLink (logs + model.ready) | Real-time streaming |
| SSE | alphaTrade → alphaLink (tick events, fills) | Live dashboard |
| Redis pub/sub | alphaGen worker → alphaTrade | model.ready notification |
| Celery (Redis broker) | alphaGen API → Celery worker | Async training job dispatch |
| S3 API (MinIO) | alphaGen write, alphaTrade read | Model artifact transfer |
| MLflow SDK | alphaGen write, alphaTrade read | Model registry |
| OTLP gRPC | All services → OTel Collector | Distributed traces + metrics |

See [[platform/Key-Decisions]] for why these patterns were chosen.

---

## Deployment

All services run as Docker containers on a single host via Docker Compose. The `platform` named bridge network provides service discovery by container name (e.g. `postgres`, `redis`, `alphagen-api`).

**Start order:** alphaFrame infra first → alphaKey → alphaGen + alphaTrade + alphaLink (depend on infra being healthy).

**Local URLs:**
- UI: `http://localhost:3000`
- Grafana: `http://localhost:3001`
- MLflow: `http://localhost:5000`
- MinIO Console: `http://localhost:9001`
