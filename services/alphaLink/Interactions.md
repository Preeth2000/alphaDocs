---
service: alphaLink
page: Interactions
tags:
  - service/alphaLink
  - interactions
---

# alphaLink — Interactions

[[alphaLink|alphaLink]] · [[alphaDocs/services/alphaLink/Architecture|Architecture]] · [[alphaDocs/services/alphaLink/API|API]] · [[alphaDocs/services/alphaLink/Data|Data]] · [[alphaDocs/services/alphaLink/Config|Config]]

---

## Inputs

| Source | Mechanism | Format | Trigger |
|---|---|---|---|
| Browser | HTTP GET/POST/etc. | JSON, form data | User interaction |
| Browser | SSE long-poll `GET /api/jobs/[id]/logs` | — | User views training run |
| Browser | SSE long-poll `GET /api/trade/events` | — | User opens dashboard |
| [[alphaKey\|alphaKey]] | HTTP response to `/api/auth/*` | JWT tokens, user object | Login / refresh |
| [[alphaGen\|alphaGen]] | HTTP responses to `/api/jobs/*` | Run DTOs, model info | Training management |
| [[alphaGen\|alphaGen]] | SSE `GET /runs/{id}/log` | Log events | Forwarded via `/api/jobs/[id]/logs` |
| [[alphaTrade\|alphaTrade]] | HTTP responses to `/api/trade/*` | Positions, orders, models, etc. | Trading management |
| [[alphaTrade\|alphaTrade]] | SSE `/stream` | Event stream | Forwarded via `/api/trade/events` |
| Local filesystem | File read | JSON templates | `GET /api/templates` |
| `att` CLI binary | Subprocess stdout | JSON validation result | `POST /api/jobs/validate` |

---

## Outputs

| Destination | Mechanism | Format | Trigger |
|---|---|---|---|
| Browser | HTTP response | JSON, HTML (SSR) | All page loads + API calls |
| Browser | SSE stream | `{type: "log"\|"status", ...}` | Training log forwarding |
| Browser | SSE stream | alphaTrade event JSON | Trading event forwarding |
| [[alphaKey\|alphaKey]] | HTTP proxy | Auth requests | Login, logout, refresh, register |
| [[alphaGen\|alphaGen]] | HTTP proxy | Training requests | Job submission, cancel, publish |
| [[alphaTrade\|alphaTrade]] | HTTP proxy | Trading requests | All trade management actions |
| Local filesystem | File write | JSON | `POST /api/templates` (save template) |
| `~/.alphalink/artifacts/` | File write | Binary (model.onnx, manifest.json) | `GET /api/jobs/[id]/artifacts/[filename]` |
| alphaGen `.env` file | File write | env file format | `POST /api/polygon-key` (sets POLYGON_API_KEY) |

---

## Upstream Dependencies

| Service | What it provides | Required? |
|---|---|---|
| [[alphaKey\|alphaKey]] | Authentication (login, token refresh, user identity) | ✅ |
| [[alphaGen\|alphaGen]] | Training jobs, model list, validation config | ✅ |
| [[alphaTrade\|alphaTrade]] | Live trading state, SSE events, model management | ✅ |
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
