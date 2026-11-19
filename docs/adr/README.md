# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the Taskify project.

Per the project constitution (Architecture Standards — ADR governance rule), any change to
the system architecture documented in `docs/hld.md` MUST be preceded by an ADR that is
created and approved before the corresponding plan or implementation proceeds.

## Format

Each ADR follows this structure:

| Field | Description |
|-------|-------------|
| **Title** | Short imperative phrase naming the decision |
| **Status** | `Proposed` / `Accepted` / `Deprecated` / `Superseded` |
| **Context** | The problem, constraint, or force driving the decision |
| **Decision** | The chosen approach |
| **Consequences** | Trade-offs, risks, accepted costs, and follow-up actions |

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-phase1-no-authentication.md) | Defer Authentication to Phase 2 (X-User-Id Header for Phase 1) | Accepted |
| [0002](0002-phase1-modular-monolith.md) | Modular Monolith for Phase 1 (Microservice Extraction Deferred) | Accepted |
| [0003](0003-sync-sqlalchemy-pymysql.md) | Synchronous SQLAlchemy 2.0 with PyMySQL Driver | Accepted |
| [0004](0004-openapi-generated-typescript-client.md) | Generated TypeScript Client from OpenAPI Schema (@hey-api/openapi-ts) | Accepted |

## Creating a new ADR

1. Copy the structure above into `docs/adr/NNNN-short-title.md`
2. Set status to `Proposed`
3. Open a PR — an ADR MUST be approved before the architecture change is implemented
4. Update this index
