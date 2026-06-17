# projectAlpha Documentation Vault

> projectAlpha canonical reference documentation.  
> **For best usage, open as an Obsidian vault**: `File → Open Folder as Vault → alphaDocs/`

---

# Platform Overview

[[README]] · [[platform/Features]] · [[platform/Tech-Stack]] · [[platform/Key-Decisions]]

> projectAlpha is an automated algorithmic trading platform — ML models trained on price data are validated, published, and executed live against Trading 212.

| | |
|---|---|
| **Goal** | Automated ML-driven trading across multiple instruments |
| **Architecture** | Microservices — each service is an independent repo |
| **Communication** | REST · SSE · Redis pub/sub · Celery task queue |
| **Infra host** | [[alphaFrame\|alphaFrame]] (Docker Compose) |

→ [[platform/Overview]] for the full system map + global Mermaid diagram.

---

## Platform-Wide Docs

| Page | Contents |
|---|---|
| [[platform/Overview]] | System map, global data flow, Mermaid architecture diagram |
| [[platform/Features]] | All platform features grouped by domain |
| [[platform/Tech-Stack]] | Full tech stack per service + shared infra |
| [[platform/Key-Decisions]] | Architecture decision records (comms, security, scaling, observability) |

---

## Services

| Service  | Purpose | Port |
|---|---|---|
| alphaFrame | Infrastructure — MinIO, MLflow, Redis, Postgres, Nginx, OTel | multiple |
| alphaGen | ML model generation — train, validate, backtest, publish | 8000 |
| alphaTrade  | Trading executor — broker, risk, scheduler, consensus | 8001 |
| alphaLink| Frontend UI — Next.js + BFF proxy | 3000 |
| alphaKey | Auth & account management | 8000 |
| alphaTest| Regression testing suite | TBD |
| alphaPerf | Performance testing suite | TBD |

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

1. Download all repos to parent directory.
2. From service alphaFrame, run ```docker compose --build```

---

## ToDo

Not-yet-done cross-service work and long-term tasks. See [[ToDo]].

| Page | Contents |
|---|---|
| [[ToDo/Cross-Service-Backlog]] | Open cross-service obligations + known unfixed races/bugs |
| [[ToDo/Compliance]] | Open compliance items (T212 ToS, FCA perimeter, GDPR, production secrets) |

---

## Templates

| Template | Use |
|---|---|
| [[_templates/service-template]] | Hub note for a new service |
| [[_templates/Architecture-template]] | Architecture sub-page |
| [[_templates/Interactions-template]] | Interactions sub-page |
| [[_templates/API-template]] | API sub-page |
| [[_templates/Data-template]] | Data sub-page |
| [[_templates/Config-template]] | Config sub-page |

> [!tip] Adding a new service
> 1. Create `services/<ServiceName>/` folder
> 2. Copy each template; replace `{{placeholders}}`
> 3. Add entry to this README table
> 4. Add node to [[platform/Overview]] Mermaid diagram
> 5. Update [[reference/Ports-and-Endpoints]] and [[platform/Tech-Stack]]

---

## Update Policy

These docs are **living documentation** — update them when:
- New API endpoint added/removed
- New env var added
- DB schema changed (migration added)
- New service dependency wired
- Override chain / config logic changes
- New service spun up

*Last reviewed: 2026-06-06*



## Reference

| Page | Contents |
|---|---|
| [[reference/Glossary]] | Domain terms: run, manifest, gate, OCO, consensus, etc. |
| [[reference/Ports-and-Endpoints]] | Port map for all services and infrastructure |
| [[reference/Event-Channels]] | Redis pub/sub + SSE channels and their consumers |


## Contributing
If you'd like to contribute changes to the docs, open a PR against the `main` branch. Keep docs in Markdown and follow existing templates in `_templates/`.

## Notes about Obsidian
This repository also stores an Obsidian vault. The vault-friendly README is preserved as `README.obsidian.md` for local editing (backlinks, transclusions). `README.md` is intended for GitHub rendering.

## License
See the LICENSE file in the repository root if present. (Currently in ToDo)
