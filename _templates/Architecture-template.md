---
service: "{{SERVICE_NAME}}"
page: Architecture
tags:
  - service/{{service_slug}}
  - architecture
---

# {{SERVICE_NAME}} — Architecture

[[{{SERVICE_NAME}}]] · [[alphaDocs/services/alphaKey/Interactions]] · [[alphaDocs/services/alphaKey/API]] · [[alphaDocs/services/alphaKey/Data]] · [[alphaDocs/services/alphaKey/Config]]

---

## Purpose

> Brief statement: what problem this service solves in the platform.

---

## Internal Modules

| Module / Package | Path | Responsibility |
|---|---|---|
| `api` | `{{src}}/api/` | FastAPI app, routers, dependency injection |
| `store` | `{{src}}/store/` | DB models, repos, migrations |
| `…` | `…` | … |

---

## Primary Flow — Sequence

```mermaid
sequenceDiagram
    participant Caller
    participant API
    participant Logic
    participant DB
    participant External

    Caller->>API: POST /endpoint
    API->>Logic: process(payload)
    Logic->>DB: read/write
    Logic->>External: call
    API-->>Caller: response
```

---

## Component Interaction

```mermaid
flowchart LR
    A[Module A] --> B[Module B]
    B --> C[(DB)]
    B --> D[External]
```

---

## Key Design Decisions

- Decision 1 — reason
- Decision 2 — reason

---

*See [[platform/Key-Decisions]] for platform-wide ADRs.*
