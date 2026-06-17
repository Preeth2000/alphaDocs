---
page: ToDo
tags:
  - todo
  - backlog
---

# ToDo

[[README]] · [[platform/Overview]]

Working backlog for two kinds of not-yet-done work:

- **Cross-service work** — an obligation created by a change in one service that another
  service hasn't implemented yet (contract changes, new env vars, new consumers, etc.).
- **Long-term tasks** — known work that has no owner/session working on it yet.

## Rule

Once a task here is done **and** documented in its proper place (the owning service's
Architecture/API/Config/Interactions doc, or `platform/`), remove it from this folder.
This folder should never hold a description of finished work — that's what the proper docs
are for.

## Current files

- [[ToDo/Cross-Service-Backlog]] — open cross-service items + confirmed unfixed races/bugs.
- [[ToDo/Compliance]] — open legal/compliance items requiring human review.
