# alphaDocs

> Documentation vault for the projectAlpha trading platform. Single source of truth for all service architecture, APIs, configuration, and data models.

**Format:** Obsidian markdown vault · **Repo:** `projectAlpha/alphaDocs/`

> **Note:** `alphaDocs/` is a symlink to an external Obsidian vault. The git-tracked content lives inside `alphaDocs/alphaDocs/`. Open that folder in [Obsidian](https://obsidian.md) for full graph/link navigation, or browse the markdown files directly on GitHub.

---

## What it is

alphaDocs is the living technical documentation for projectAlpha — a self-hosted ML trading platform. It documents every service's architecture, API endpoints, configuration, data models, and cross-service interactions. The root READMEs in each service directory (`alphaGen/README.md`, etc.) are derived from this vault.

---

## Vault structure

```
alphaDocs/alphaDocs/
├── README.md                        # This file / vault map-of-content
├── platform/
│   ├── Overview.md                  # System map, data flow diagrams (Mermaid)
│   ├── Features.md
│   ├── Tech-Stack.md                # Versioned dependency inventory per service
│   └── Key-Decisions.md             # Architecture decision records
├── reference/
│   ├── Ports-and-Endpoints.md       # Complete port map, Nginx routes, DBs, buckets
│   ├── Event-Channels.md            # Redis pub/sub + SSE channel catalogue
│   └── Glossary.md                  # Domain terms (run, manifest, gate, consensus...)
├── services/
│   ├── alphaFrame/                  # 6-page set: hub + Architecture/Interactions/API/Data/Config
│   ├── alphaGen/
│   ├── alphaKey/
│   ├── alphaLink/
│   ├── alphaTrade/
│   ├── alphaTest/
│   └── alphaPerf/
├── _templates/
│   ├── README-template.md           # GitHub root README template (fill + drop N/A sections)
│   ├── service-template.md          # Obsidian hub note template
│   ├── API-template.md
│   ├── Architecture-template.md
│   ├── Config-template.md
│   ├── Data-template.md
│   └── Interactions-template.md
└── ToDo/
    ├── Cross-Service-Backlog.md
    └── Compliance.md
```

Each service folder under `services/` follows the same 6-page split:

| Page | Contents |
|---|---|
| `<Service>.md` | Hub — one-line purpose, Mermaid flow, links to sub-pages |
| `Architecture.md` | Modules, primary flow sequences, design decisions |
| `Interactions.md` | Upstream inputs, downstream outputs, cross-service auth |
| `API.md` | All endpoints exposed + outbound calls made |
| `Data.md` | Datastores, table schemas, read/write counts per operation |
| `Config.md` | Env vars, config files, override priority chain |

---

## How to use

**Obsidian (recommended):** Open `alphaDocs/alphaDocs/` as a vault. Wikilinks, graph view, and Mermaid diagrams render natively.

**GitHub / plain markdown:** All files are standard markdown. Mermaid diagrams render in GitHub's markdown preview. Wikilinks (`[[...]]`) appear as-is but the linked files are reachable by path.

**Starting points:**
- [`platform/Overview.md`](alphaDocs/platform/Overview.md) — full system diagram and primary data flows
- [`reference/Ports-and-Endpoints.md`](alphaDocs/reference/Ports-and-Endpoints.md) — all ports, Nginx routes, databases, buckets
- [`reference/Event-Channels.md`](alphaDocs/reference/Event-Channels.md) — Redis pub/sub and SSE channels

---

## Related services

| Service | README | Purpose |
|---|---|---|
| [alphaFrame](../alphaFrame) | [README](../alphaFrame/README.md) | Shared infra (Postgres, Redis, MinIO, Nginx, observability) |
| [alphaGen](../alphaGen) | [README](../alphaGen/README.md) | ML model trainer — outputs ONNX + manifest |
| [alphaKey](../alphaKey) | [README](../alphaKey/README.md) | Auth, JWT, encrypted credential vault |
| [alphaLink](../alphaLink) | [README](../alphaLink/README.md) | Web UI and BFF proxy |
| [alphaTrade](../alphaTrade) | [README](../alphaTrade/README.md) | Live trading bot |
| [alphaTest](../alphaTest) | [README](../alphaTest/README.md) | Cross-service regression + contract tests |
