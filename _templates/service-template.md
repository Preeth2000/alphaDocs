---
service: "{{SERVICE_NAME}}"
status: "full | partial | stub"
stack: "{{STACK}}"
owner: ""
last-reviewed: "{{YYYY-MM-DD}}"
tags:
  - service/{{service_slug}}
  - status/{{status}}
---

# {{SERVICE_NAME}}

> One-line purpose of this service.

**Status:** `full | partial | stub`  
**Port:** `{{PORT}}`  
**Repo path:** `projectAlpha/{{SERVICE_NAME}}/`

---

## Contents

| Page | Description |
|---|---|
| [[alphaDocs/services/alphaKey/Architecture]] | Internal modules, components, primary flow sequence |
| [[alphaDocs/services/alphaKey/Interactions]] | All inputs and outputs, upstream/downstream |
| [[alphaDocs/services/alphaKey/API]] | Endpoints exposed + outbound calls made |
| [[alphaDocs/services/alphaKey/Data]] | Datastores, DB read/write counts per operation |
| [[alphaDocs/services/alphaKey/Config]] | Env vars, config files, override priority chain |

---

## Mermaid Flow

```mermaid
flowchart TD
    %% Replace with service-specific diagram
    A[External Caller] --> B[{{SERVICE_NAME}} API]
    B --> C[Core Logic]
    C --> D[(Datastore)]
    C --> E[External Service]
```

---

## Related

- [[platform/Overview]] — system-wide context
- [[services/{{upstream}}]] — upstream service
- [[services/{{downstream}}]] — downstream service
- [[reference/Ports-and-Endpoints]] — port map
- [[reference/Event-Channels]] — pub/sub channels

---
*Update this file when: endpoints added/removed, stack changes, new upstream/downstream wired.*
