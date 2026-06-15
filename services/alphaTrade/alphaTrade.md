---
service: alphaTrade
status: full
stack: Python 3.11, FastAPI, SQLModel, Alembic, SQLite/PostgreSQL, Redis, MinIO, MLflow, ONNX Runtime, APScheduler, TA-Lib
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaTrade
  - status/full
  - trading
---

# alphaTrade

> Live trading executor — loads ONNX models published by [[services/alphaGen/alphaGen|alphaGen]], runs inference on bar-close, applies risk gates, and submits orders to Trading 212.

**Status:** 🟢 Full  
**Ports:** `8081` (REST API), `8080` (health probe), `9090` (Prometheus metrics)  
**Repo path:** `projectAlpha/alphaTrade/`

---

## Contents

| Page | Description |
|---|---|
| [[services/alphaTrade/Architecture\|Architecture]] | Internal modules, model lifecycle, scheduler tick |
| [[services/alphaTrade/Interactions\|Interactions]] | All inputs/outputs, upstream/downstream |
| [[services/alphaTrade/API\|API]] | 40+ endpoints + SSE stream + outbound calls |
| [[services/alphaTrade/Data\|Data]] | 23 DB tables, read/write counts, MinIO, Redis |
| [[services/alphaTrade/Config\|Config]] | Env vars, overrides.yaml, DB-first override chain |

---

## Mermaid Flow

```mermaid
flowchart TD
    subgraph Inputs
        MN[(MinIO models bucket)]
        ML[MLflow registry]
        T212[Trading 212 API]
        YF[yfinance / Polygon]
        AK[alphaKey JWT]
        AL[alphaLink BFF]
    end

    subgraph alphaTrade
        SYNC[ModelSyncDaemon\npolls MinIO / MLflow every 60s]
        REG[ModelRegistry\nin-memory ONNX models]
        SCHED[BarCloseScheduler\nNYSE-calendar aware]
        FEAT[FeaturePipeline\nTA-Lib indicators]
        INF[ONNX Inference\nper model]
        CONS[Consensus\nsoftmax-avg multi-model]
        RISK[RiskGates\nhalt/cooldown/sizing/sector]
        BROKER[T212Client\nmarket + OCO orders]
        OCO[OCO Monitor\nasync task per position]
        API[FastAPI :8081]
        HEALTH[Health :8080]
        PROM[Prometheus :9090]
        DB[(SQLite/Postgres\n23 tables)]
        STREAM[SSE /stream\nevent bus]
    end

    subgraph Outputs
        ORDER[Orders → T212]
        ALERTS[Slack/Email/Webhook]
        ALPHALINK[alphaLink dashboard]
    end

    MN -->|model.onnx + manifest.json| SYNC
    ML -->|Production-stage models| SYNC
    SYNC --> REG
    SCHED -->|bar close tick| FEAT
    YF & T212 -->|OHLCV + price| FEAT
    FEAT --> INF
    REG --> INF
    INF --> CONS
    CONS --> RISK
    RISK -->|approved| BROKER
    BROKER -->|place market order| ORDER
    BROKER -->|place SL/TP bracket| OCO
    OCO -->|poll fills| T212
    OCO -->|exit| DB
    RISK & BROKER & OCO --> DB
    DB & STREAM --> ALPHALINK
    AL -->|REST| API
    AK -->|JWT verify| API
    API --> DB
    BROKER & OCO --> ALERTS
```

---

## Related

- [[platform/Overview]] — system-wide context
- [[services/alphaFrame/alphaFrame|alphaFrame]] — provides Postgres, Redis, MinIO, MLflow
- [[services/alphaGen/alphaGen|alphaGen]] — produces model artifacts consumed here
- [[services/alphaLink/alphaLink|alphaLink]] — primary REST caller + SSE consumer
- [[services/alphaKey/alphaKey|alphaKey]] — JWT verification
- [[reference/Glossary]] — OCO, consensus, gate, manifest, override
- [[reference/Event-Channels]] — model.ready, SSE /stream
