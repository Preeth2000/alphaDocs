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
| **Infra host** | [[alphaFrame\|alphaFrame]] (Docker Compose) |

→ [[platform/Overview]] for the full system map + global Mermaid diagram.

---

## Services

| Service | Purpose | Port |
|---|---|---|
| [[alphaFrame\|alphaFrame]] | Shared infra — Postgres, Redis, MinIO, MLflow, Nginx, Observability | multiple |
| [[alphaGen\|alphaGen]] | ML model generation — train, validate, backtest, publish | 8000 |
| [[alphaTrade\|alphaTrade]] | Trading executor — scheduler, inference, risk, orders | 8081 |
| [[alphaLink\|alphaLink]] | Frontend + BFF — Next.js UI + proxy | 3000 |
| [[alphaKey\|alphaKey]] | Auth & credential vault — JWT, Argon2, Fernet | 8000 |
| [[alphaTest\|alphaTest]] | Regression testing suite | TBD |
| [[alphaPerf\|alphaPerf]] | Performance testing suite | TBD |

---

## Platform-Wide Docs

| Page | Contents |
|---|---|
| [[platform/Overview]] | System map, global data flow, Mermaid architecture diagram |
| [[platform/Features]] | All platform features grouped by domain |
| [[platform/Tech-Stack]] | Full tech stack per service + shared infra |
| [[platform/Key-Decisions]] | Architecture decision records (comms, security, scaling, observability) |

---

## Primary Data Flows

(Training/Publishing, Live trading tick — see platform/Overview and service pages for sequence diagrams and flowcharts.)

---

## ToDo

Not-yet-done cross-service work and long-term tasks. See [[ToDo]] and [[ToDo/Cross-Service-Backlog]].

## Reference

See [[reference/Glossary]], [[reference/Ports-and-Endpoints]], and [[reference/Event-Channels]] for domain terms and port maps.

## Templates

Templates for new service pages and common doc types live in `_templates/` (API, Architecture, Data, Config, Interactions).

---

## Contributing

If you'd like to contribute changes to the docs, open a PR against the `main` branch. Keep docs in Markdown and follow existing templates in `_templates/`.

## Notes about Obsidian

This repository also stores an Obsidian vault. The vault-friendly README is preserved as `README.obsidian.md` for local editing (backlinks, transclusions). `README.md` is intended for GitHub rendering.

## License

See the LICENSE file in the repository root if present.

