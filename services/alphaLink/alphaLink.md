---
service: alphaLink
status: full
stack: Next.js 16 (App Router), React 18, TypeScript 5, Tailwind CSS 3, Zustand 5, React Query 5, shadcn/ui
owner: ""
last-reviewed: 2026-06-06
tags:
  - service/alphaLink
  - status/full
  - frontend
---

# alphaLink

> Frontend UI and BFF (Backend for Frontend) — Next.js app that provides the trading platform dashboard and proxies all browser requests to backend services.

**Status:** 🟢 Full  
**Port:** `3000`  
**Repo path:** `projectAlpha/alphaLink/`

---

## Contents

| Page | Description |
|---|---|
| [[services/alphaLink/Architecture\|Architecture]] | App structure, BFF pattern, state management |
| [[services/alphaLink/Interactions\|Interactions]] | All inputs/outputs, proxied services |
| [[services/alphaLink/API\|API]] | All BFF route handlers (`/app/api/*`) + outbound calls |
| [[services/alphaLink/Data\|Data]] | No DB — local template/artifact filesystem storage |
| [[services/alphaLink/Config\|Config]] | Env vars, auth flow, session config |

---

## Mermaid Flow

```mermaid
flowchart TD
    Browser -->|HTTPS| MW[Middleware\ncookie auth guard]
    MW -->|protected| Pages

    subgraph Pages["App Pages (Server + Client Components)"]
        LOGIN["/login /signup"]
        TRADE["/trade/dashboard + /trade/*"]
        TRAIN["/trade/train/*"]
        MODELS["/trade/models/*"]
        ACCOUNT["/account"]
    end

    subgraph BFF["BFF API Routes (/app/api/*)"]
        AUTH_RT["/api/auth/*"]
        JOBS_RT["/api/jobs/*"]
        TRADE_RT["/api/trade/*"]
        CFG_RT["/api/config/*"]
        TMPL_RT["/api/templates/*"]
    end

    subgraph State["Client State"]
        ZUSTAND_AUTH[Zustand: auth-store\nuser + access_token]
        ZUSTAND_CFG[Zustand: session-store\nRunConfig]
        RQ[React Query\nserver data cache]
    end

    Pages --> BFF
    Pages --> State

    AUTH_RT -->|proxy| AK[alphaKey :8000]
    JOBS_RT -->|proxy| AG[alphaGen :8000]
    TRADE_RT -->|proxy| AT[alphaTrade :8081]
    TMPL_RT -->|filesystem| FS["~/.alphalink/templates/"]
    CFG_RT --> AG

    AK -->|JWT tokens| AUTH_RT
    AUTH_RT -->|httpOnly cookie: refresh_token| Browser
    AUTH_RT -->|access_token in JSON| ZUSTAND_AUTH
```

---

## Related

- [[platform/Overview]] — system-wide context
- [[services/alphaGen/alphaGen|alphaGen]] — training API backend
- [[services/alphaTrade/alphaTrade|alphaTrade]] — trading API backend
- [[services/alphaKey/alphaKey|alphaKey]] — auth backend
- [[reference/Glossary]] — BFF, SSE
