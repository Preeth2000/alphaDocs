---
service: alphaLink
page: Interactions
tags:
  - service/alphaLink
  - interactions
---

# alphaLink — Interactions

[[services/alphaLink/alphaLink|alphaLink]] · [[services/alphaLink/Architecture|Architecture]] · [[services/alphaLink/API|API]] · [[services/alphaLink/Data|Data]] · [[services/alphaLink/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger |
|---|---|---|---|
| Browser | HTTP GET/POST/etc. | JSON, form data | User interaction |
| Browser | SSE long-poll `GET /api/jobs/[id]/logs` | — | User views training run |
| Browser | SSE long-poll `GET /api/trade/events` | — | User opens dashboard |
| [[services/alphaKey/alphaKey\|alphaKey]] | HTTP response to `/api/auth/*` | JWT tokens, user object | Login / refresh |
| [[services/alphaGen/alphaGen\|alphaGen]] | HTTP responses to `/api/jobs/*` | Run DTOs, model info | Training management |
| [[services/alphaGen/alphaGen\|alphaGen]] | SSE `GET /runs/{id}/log` | Log events | Forwarded via `/api/jobs/[id]/logs` |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | HTTP responses to `/api/trade/*` | Positions, orders, models, etc. | Trading management |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | SSE `/stream` | Event stream | Forwarded via `/api/trade/events` |
| Local filesystem | File read | JSON templates | `GET /api/templates` |
| `att` CLI binary | Subprocess stdout | JSON validation result | `POST /api/jobs/validate` |

---

## Outputs

| Destination | Mechanism | Format | Trigger |
|---|---|---|---|
| Browser | HTTP response | JSON, HTML (SSR) | All page loads + API calls |
| Browser | SSE stream | `{type: "log"\|"status", ...}` | Training log forwarding |
| Browser | SSE stream | alphaTrade event JSON | Trading event forwarding |
| [[services/alphaKey/alphaKey\|alphaKey]] | HTTP proxy | Auth requests | Login, logout, refresh, register |
| [[services/alphaGen/alphaGen\|alphaGen]] | HTTP proxy | Training requests | Job submission, cancel, publish |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | HTTP proxy | Trading requests | All trade management actions |
| Local filesystem | File write | JSON | `POST /api/templates` (save template) |
| `~/.alphalink/artifacts/` | File write | Binary (model.onnx, manifest.json) | `GET /api/jobs/[id]/artifacts/[filename]` |
| alphaGen `.env` file | File write | env file format | `POST /api/polygon-key` (sets POLYGON_API_KEY) |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[services/alphaKey/alphaKey\|alphaKey]] | Authentication (login, token refresh, user identity) | ✅ |
| [[services/alphaGen/alphaGen\|alphaGen]] | Training jobs, model list, validation config | ✅ |
| [[services/alphaTrade/alphaTrade\|alphaTrade]] | Live trading state, SSE events, model management | ✅ |
| `att` CLI binary | Config validation | ⬜ (graceful fallback if missing) |

---

## Downstream Consumers

| Consumer | What it gets |
|---|---|
| Browser / user | Full React UI, live dashboards, SSE event streams |

---

## Data alphaTrade Consumes via alphaLink

alphaLink surfaces the following alphaTrade data to the user:

| alphaTrade data | alphaLink page |
|---|---|
| Open positions | `/trade/positions`, `/trade/dashboard` |
| Order history | `/trade/orders` |
| Signal history | `/trade/signals` |
| Daily P&L | `/trade/pnl` |
| Closed trade journal | `/trade/trades` |
| Equity curve | `/trade/equity`, `/trade/dashboard` |
| Model list + performance | `/trade/models/*` |
| Backtest runs + results | `/trade/backtest` |
| Bot health + settings | Settings page |
| Kill switch state | Dashboard header |
| Live SSE events | Real-time dashboard updates |
