# ADR-0002: Modular Monolith for Phase 1 (Microservice Extraction Deferred)

**Status**: Accepted
**Date**: 2026-11-19
**Deciders**: Taskify project lead
**Constitution reference**: Principle III — Microservices Architecture (justified deviation)

---

## Context

The project constitution (Principle III) requires independently deployable, loosely coupled
services, each owning its data store. Phase 1 delivers a team Kanban board with four domain
areas: users, projects, tasks, and comments.

Splitting these into four separate microservices before domain boundaries are validated by
real usage introduces:
- Service-mesh overhead (service discovery, inter-service auth, network retries)
- Distributed tracing complexity on a greenfield codebase
- Coordination cost for a two-container stack (backend + frontend) that could be one
  `docker compose up`

The domain boundaries have not yet been proven — task and comment domains may remain
co-located in production, or projects may become a separate write-model service. Building
four services before this is known risks incorrect seam placement that is expensive to undo.

---

## Decision

Phase 1 is delivered as a single FastAPI application (`backend/app/`) with a layered
internal structure that mirrors the future microservice split:

```
routes/     → HTTP boundary (one file per domain)
services/   → Business logic (no direct DB access)
repositories/ → DB access only (no business logic)
models/     → SQLAlchemy ORM models
schemas/    → Pydantic v2 request/response contracts
```

Each domain module (users, projects, tasks, comments) has zero cross-module coupling at the
service and repository layers. The OpenAPI contract (`/openapi.json`) is the versioned
interface; the generated TypeScript client enforces it on the frontend.

This is the standard "modular monolith as microservices precursor" pattern.

---

## Consequences

**Accepted risks**:
- The entire backend is a single deployable unit; a fault in one domain can affect all
  domains.
- Scaling the task-write path independently of the read path is not possible without
  extracting the service first.

**Mitigations in place**:
- Layered structure enforces zero cross-domain coupling at the code level; services and
  repositories never import from sibling domains.
- The OpenAPI contract is the extraction boundary: each domain can be lifted into its own
  FastAPI app by moving its `routes/`, `services/`, `repositories/`, `models/`, and
  `schemas/` files with no cross-domain import changes required.
- Deviation documented in `plan.md` Constitution Check and `docs/hld.md` Section 8.

**Follow-up actions**:
- Phase 2 trigger: extract if any domain requires independent scaling, separate deployment
  cadence, or a team boundary forms around it.
- When extracting: introduce an API gateway, add mTLS or signed JWT inter-service auth, and
  migrate each domain's data to its own store — all required by Principle III.
