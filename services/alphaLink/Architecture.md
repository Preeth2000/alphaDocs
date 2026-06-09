---
service: alphaLink
page: Architecture
tags:
  - service/alphaLink
  - architecture
---

# alphaLink — Architecture

[[services/alphaLink/alphaLink|alphaLink]] · [[services/alphaLink/Interactions|Interactions]] · [[services/alphaLink/API|API]] · [[services/alphaLink/Data|Data]] · [[services/alphaLink/Config|Config]]

---

## Purpose

alphaLink is the unified frontend for the entire alphaPlatform. It provides:
1. **Pages** — React UI for training models, monitoring live trading, backtesting, account management
2. **BFF routes** (`/app/api/*`) — server-side proxies that hide backend service URLs, inject auth, and normalise responses

---

## App Pages (`src/app/`)

| Route | Purpose |
|---|---|
| `/` | Redirects to `/trade/dashboard` |
| `/login` | Email/password login → POST `/api/auth/login` |
| `/signup` | Registration → POST `/api/auth/register` |
| `/account` | User account + VaultSection (encrypted secrets UI) |
| `/trade/dashboard` | Main trading dashboard — positions, orders, equity curve |
| `/trade/models/published` | Published models from alphaGen |
| `/trade/models/registry` | MLflow model registry view |
| `/trade/models/public` | Public model library |
| `/trade/backtest` | Backtest runner + results |
| `/trade/train/configure` | Model training config builder |
| `/trade/train/runs` | Training job list |
| `/trade/train/results` | Individual run results |
| `/trade/train/compare` | Side-by-side run comparison |
| `/trade/train/templates` | Saved config templates |
| `/trade/train/jobs` | Training job management |
| `/trade/equity` | Equity curve chart |
| `/trade/orders` | Order history |
| `/trade/positions` | Open positions |
| `/trade/pnl` | P&L by day |
| `/trade/signals` | Signal history |
| `/trade/trades` | Closed trade journal |
| `/results/[jobId]` | Individual training run result |
| `/runs` | All training runs list |
| `/templates` | Global template management |
| `/configure` | Platform configuration |
| `/compare` | Run comparison (standalone) |

---

## BFF Pattern

All browser-to-backend communication goes through Next.js API routes (`/app/api/*`):

```
Browser → /api/auth/* → alphaKey
Browser → /api/jobs/* → alphaGen
Browser → /api/trade/* → alphaTrade
Browser → /api/config/* → alphaGen
Browser → /api/templates/* → ~/.alphalink/templates/ (filesystem)
Browser → /api/instruments?q= → local search
Browser → /api/polygon-key → alphaGen .env file (read/write)
Browser → /api/fs/browse → local filesystem (safe traversal)
```

**Benefits:**
- Backend service URLs never exposed to browser
- `Authorization: Bearer <token>` injected server-side
- `refresh_token` stored in httpOnly cookie — never accessible to JavaScript
- Response normalisation (e.g. alphaGen run → frontend Job type)

---

## State Management

### Auth Store (Zustand, in-memory only)

```typescript
// src/lib/auth-store.ts
interface AuthState {
  user: { name, email, role, minio_user, minio_account } | null
  role: string
  status: 'unauthenticated' | 'authenticated' | 'loading'
  accessToken: string | null   // Never persisted to localStorage
}
```

- `accessToken` only in-memory — lost on page refresh → `AuthInitializer` restores via `POST /api/auth/refresh` (reads httpOnly cookie)
- `user` persisted to localStorage (non-sensitive identity data)

### Session Config Store (Zustand, localStorage)

```typescript
// src/lib/store.ts (name: "alphagen.session")
interface SessionConfig {
  run_name, tickers, interval, startDate, endDate, provider,
  labelStrategy, features, modelArch, trainParams, backtestConfig, ...
}
```

- Preserved across page navigation
- `PRESERVED_KEYS`: tickers, startDate, endDate, provider (not reset when changing steps)
- Used by config builder pages and `config-to-yaml.ts` to generate `RunConfig` YAML

### React Query

Server data (positions, orders, runs, models) fetched and cached via React Query — auto-invalidation on mutations.

---

## Authentication Flow

```mermaid
sequenceDiagram
    participant Browser
    participant MW as Middleware
    participant BFF_AUTH as /api/auth/*
    participant AK as alphaKey

    Browser->>BFF_AUTH: POST /api/auth/login {email, password}
    BFF_AUTH->>AK: POST /auth/login
    AK-->>BFF_AUTH: {access_token, refresh_token, ...}
    BFF_AUTH->>AK: GET /auth/me (with access_token)
    AK-->>BFF_AUTH: {id, email, role, minio_user, ...}
    BFF_AUTH-->>Browser: Set-Cookie: alphakey_session=refresh_token (httpOnly)
    BFF_AUTH-->>Browser: JSON {access_token, user}
    Browser->>Browser: Zustand.setAuth(user, access_token)

    Note over Browser: Page refresh — token lost
    Browser->>BFF_AUTH: POST /api/auth/refresh
    BFF_AUTH->>BFF_AUTH: Read httpOnly cookie
    BFF_AUTH->>AK: POST /auth/refresh {refresh_token}
    AK-->>BFF_AUTH: {access_token, (new refresh_token)}
    BFF_AUTH-->>Browser: Set-Cookie: update cookie
    BFF_AUTH-->>Browser: JSON {access_token}

    Note over MW: All protected routes
    MW->>MW: Check alphakey_session cookie
    alt Missing cookie
        MW-->>Browser: Redirect to /login
    end
```

---

## Key Components

| Category | Key Components |
|---|---|
| **Layout** | `Navigation`, `ThemeProvider`, root layout |
| **Auth** | `AuthInitializer` (restores token on mount), login/signup forms |
| **Config builder** | Multi-step form (Data → Model → Train) using SessionConfig store |
| **Trade UI** | `SSEProvider` (manages live events from alphaTrade `/stream`), position/order/signal tables |
| **Charts** | Backtest equity curve (Recharts), candlestick (lightweight-charts) |
| **Account** | `VaultSection` — encrypted credential storage UI (T212, Polygon, etc.) |
| **Models** | Registry browser, model detail, promote/demote actions |
| **UI primitives** | shadcn/ui — Card, Input, Button, Dialog, Tabs, Select, Toast (Sonner) |

---

## Key Design Decisions

- **BFF proxies everything**: No direct browser-to-backend calls. Allows service URL changes without frontend deploys.
- **Access token in-memory only**: Prevents XSS token theft. Refresh token in httpOnly cookie prevents JS access.
- **Middleware cookie guard**: All routes except `/login`, `/signup`, `/api/auth/*` require `alphakey_session` cookie.
- **`att` CLI binary**: `ATT_BIN` env var points to local `att` binary for `att validate --json` (config validation before submit).
- **Standalone Next.js output**: `output: "standalone"` in `next.config.mjs` for Docker image minimisation.
