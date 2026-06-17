---
service: alphaGen
status: full
stack: Python 3.11, FastAPI, Celery, PyTorch, ONNX, PostgreSQL, Redis, MinIO, MLflow, OpenTelemetry
owner: ""
last-reviewed: 2026-06-14
tags:
  - service/alphaGen
  - status/full
  - ml
---

# alphaGen

> ML model generation service — trains, validates, backtests, and publishes ONNX models for live trading consumption.

**Status:** 🟢 Full  
**Port:** `8000` (API), `5555` (Flower monitoring)  
**Repo path:** `projectAlpha/alphaGen/`

---

## Contents

| Page | Description |
|---|---|
| [[alphaDocs/services/alphaGen/Architecture\|Architecture]] | att library modules, training pipeline, Celery job lifecycle |
| [[alphaDocs/services/alphaGen/Interactions\|Interactions]] | All inputs/outputs, upstream/downstream services |
| [[alphaDocs/services/alphaGen/API\|API]] | 14 FastAPI endpoints + SSE streams + outbound calls |
| [[alphaDocs/services/alphaGen/Data\|Data]] | PostgreSQL tables, MinIO paths, Redis channels |
| [[alphaDocs/services/alphaGen/Config\|Config]] | Env vars, RunConfig YAML schema, validation gate config |

---

## Mermaid Flow

```mermaid
flowchart TD
    subgraph AlphaLink["alphaLink BFF"]
        BFF["POST /api/jobs → POST /runs"]
    end

    subgraph AlphaGen["alphaGen :8000"]
        API[FastAPI]
        CELERY[Celery Worker]
        ATT["att library (train/validate/backtest/export/publish)"]
        SSE_LOG["SSE /runs/{id}/log"]
        SSE_EVENTS["SSE /runs/events (model.ready)"]
    end

    subgraph Stores["External Stores"]
        PG[(PostgreSQL\nruns / validation_settings)]
        RD[(Redis\ndb0: broker+pub/sub\ndb1: results)]
        MN[(MinIO\nmodels bucket)]
        ML[MLflow :5000]
    end

    subgraph DataSources
        YF[yfinance API]
        PO[Polygon.io API]
    end

    BFF -->|POST /runs| API
    API -->|enqueue task| RD
    RD -->|task dispatch| CELERY
    CELERY --> ATT
    ATT -->|fetch OHLCV| YF & PO
    ATT -->|log params + metrics| ML
    ATT -->|publish model.onnx\nmanifest.json| MN
    ATT -->|stream log lines| RD
    CELERY -->|publish model.ready| RD

    API -->|read/write run status| PG
    API -->|read run log| RD
    RD -->|pub/sub stream| SSE_LOG
    RD -->|pub/sub model.ready| SSE_EVENTS

    BFF -->|GET /api/jobs/{id}/logs SSE| SSE_LOG
    AT[alphaTrade] -->|GET /runs/events SSE| SSE_EVENTS
    AT -->|download artifacts| MN
```

---

## Related

- [[platform/Overview]] — system-wide context
- [[alphaFrame|alphaFrame]] — provides all infra (Postgres, Redis, MinIO, MLflow)
- [[alphaTrade|alphaTrade]] — consumes published models via model.ready SSE + MinIO
- [[alphaLink|alphaLink]] — primary user-facing caller
- [[reference/Event-Channels]] — Redis pub/sub channels
- [[reference/Glossary]] — run, gate, manifest, att
