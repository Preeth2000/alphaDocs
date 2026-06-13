---
title: projectAlpha — Documentation Vault
description: Map of Content for the entire alphaPlatform trading system
tags:
  - moc
  - home
last-reviewed: 2026-06-06
---

# projectAlpha Documentation Vault

> Algorithmic trading platform — canonical reference documentation.  
> **Open as an Obsidian vault**: `File → Open Folder as Vault → alphaDocs/`

---

## Platform Overview

| | |
|---|---|
| **Goal** | Automated ML-driven trading across multiple instruments |
| **Architecture** | Microservices — each service is an independent repo |
| **Communication** | REST · SSE · Redis pub/sub · Celery task queue |
| **Infra host** | [[services/alphaFrame/alphaFrame\|alphaFrame]] (Docker Compose) |

→ [[platform/Overview]] for the full system map + global Mermaid diagram.

---

## Services

| Service | Status | Purpose | Port |
|---|---|---|---|
| [[services/alphaFrame/alphaFrame\|alphaFrame]] | 🟢 Full | Infrastructure — MinIO, MLflow, Redis, Postgres, Nginx, OTel | multiple |
| [[services/alphaGen/alphaGen\|alphaGen]] | 🟢 Full | ML model generation — train, validate, backtest, publish | 8000 |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | 🟢 Full | Trading executor — broker, risk, scheduler, consensus | 8001 |
| [[services/alphaLink/alphaLink\|alphaLink]] | 🟢 Full | Frontend UI — Next.js + BFF proxy | 3000 |
| [[services/alphaKey/alphaKey\|alphaKey]] | 🟡 Partial | Auth & account management | 8000 |
| [[services/alphaTest/alphaTest\|alphaTest]] | ⬜ Planned | Regression testing suite | TBD |
| [[services/alphaPerf/alphaPerf\|alphaPerf]] | ⬜ Planned | Performance testing suite | TBD |

---

## Platform-Wide Docs

| Page | Contents |
|---|---|
| [[platform/Overview]] | System map, global data flow, Mermaid architecture diagram |
| [[platform/Features]] | All platform features grouped by domain |
| [[platform/Tech-Stack]] | Full tech stack per service + shared infra |
| [[platform/Key-Decisions]] | Architecture decision records (comms, security, scaling, observability) |

---

## Reference

| Page | Contents |
|---|---|
| [[reference/Glossary]] | Domain terms: run, manifest, gate, OCO, consensus, etc. |
| [[reference/Ports-and-Endpoints]] | Port map for all services and infrastructure |
| [[reference/Event-Channels]] | Redis pub/sub + SSE channels and their consumers |

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
