---
service: alphaFrame
status: full
stack: Docker Compose, PostgreSQL 16, Redis 7, MinIO, MLflow, Nginx, OpenTelemetry / LGTM
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaFrame
  - status/full
  - infrastructure
---

# alphaFrame

> Shared infrastructure layer — runs all stateful services and observability stack that every other platform service depends on.

**Status:** 🟢 Full  
**Repo path:** `projectAlpha/alphaFrame/`  
**No application port** — pure infrastructure; exposed via individual service ports.

---

## Contents

| Page | Description |
|---|---|
| [[services/alphaFrame/Architecture\|Architecture]] | Docker services, init flow, observability pipeline |
| [[services/alphaFrame/Interactions\|Interactions]] | Inputs, outputs, service connectivity map |
| [[services/alphaFrame/API\|API]] | No inbound API — port map and Nginx routes |
| [[services/alphaFrame/Data\|Data]] | PostgreSQL databases, MinIO buckets, Redis key spaces |
| [[services/alphaFrame/Config\|Config]] | All env vars, docker-compose wiring, Nginx config |

---

## Mermaid Flow

```mermaid
flowchart TD
    subgraph External["External Access"]
        Browser["Browser / CI"]
    end

    subgraph Nginx["Nginx :80/:443"]
        N_AUTH["/auth/ → alphaKey"]
        N_GEN["/alphagen/ → alphaGen"]
        N_SSE_GEN["/alphagen/runs/events → SSE"]
        N_SSE_TRADE["/stream → alphaTrade SSE"]
        N_TRADE["/ → alphaTrade"]
    end

    subgraph AppServices["Application Services"]
        AK[alphaKey :8000]
        AG[alphaGen :8000]
        AT[alphaTrade :8081/8080]
        AL[alphaLink :3000]
    end

    subgraph Infra["Shared Infra (platform network)"]
        PG[(PostgreSQL :5432)]
        RD[(Redis :6379)]
        MN[(MinIO :9000/9001)]
        ML[MLflow :5000]
    end

    subgraph Obs["Observability LGTM"]
        OC[OTel Collector :4317]
        PR[Prometheus :9090]
        LK[Loki :3100]
        TM[Tempo :3200]
        GR[Grafana :3001]
        AM[Alertmanager :9093]
    end

    Browser --> |HTTPS| Nginx
    Nginx --> AK & AG & AT
    AL --> |HTTP| AG & AT & AK

    AK & AG & AT --> PG
    AK & AG & AT --> RD
    AG & AT --> MN
    AG & AT --> ML

    AK & AG & AT --> |OTLP gRPC| OC
    OC --> TM
    OC --> PR
    OC --> LK
    PR --> AM
    PR & LK & TM --> GR

    PG --> PG_EXP[postgres-exporter :9187]
    RD --> RD_EXP[redis-exporter :9121]
    PG_EXP & RD_EXP --> PR
```

---

## Related

- [[platform/Overview]] — system-wide context
- [[reference/Ports-and-Endpoints]] — full port map
- [[reference/Event-Channels]] — Redis pub/sub channels
- [[platform/Key-Decisions]] — why shared infra / LGTM stack chosen
