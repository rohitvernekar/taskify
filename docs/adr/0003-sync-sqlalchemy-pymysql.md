# ADR-0003: Synchronous SQLAlchemy 2.0 with PyMySQL Driver

**Status**: Accepted
**Date**: 2026-11-19
**Deciders**: Taskify project lead

---

## Context

The backend requires a Python ORM for MySQL 8. SQLAlchemy 2.0 supports both synchronous
and asynchronous session modes. The async mode requires an async-capable driver
(e.g., `aiomysql`, `asyncmy`); the sync mode works with PyMySQL, which is a
well-maintained pure-Python MySQL driver.

FastAPI supports both sync and async route handlers. For async handlers with sync
SQLAlchemy, calls must be wrapped in `run_in_executor` to avoid blocking the event loop —
adding complexity without benefit for a single-service, low-concurrency phase 1 workload.

---

## Decision

Use **SQLAlchemy 2.0 in synchronous mode** with the **PyMySQL** driver
(`mysql+pymysql://`). Route handlers are synchronous (`def`, not `async def`) where they
perform DB access.

Rationale:
- PyMySQL is pure Python, zero C-extension build step, works identically in Docker and
  CI without driver compilation.
- Sync SQLAlchemy + sync FastAPI handlers is the simplest correct configuration; no
  `run_in_executor` wrappers needed.
- Phase 1 concurrency target is a single developer session on localhost; async I/O
  multiplexing provides no measurable benefit.

---

## Consequences

**Trade-offs**:
- If the backend is later extracted into a high-concurrency async microservice, the ORM
  layer will need to be migrated to `asyncmy` or `aiomysql` + async SQLAlchemy sessions.
  The migration surface is contained to `database.py`, `repositories/`, and route handler
  signatures — no business logic in `services/` changes.

**Follow-up actions**:
- Revisit in phase 2 if concurrency requirements emerge (e.g., WebSocket support for
  real-time board updates).
- If async migration is needed: swap driver to `aiomysql`, update `create_engine` to
  `create_async_engine`, update `SessionLocal` to `AsyncSession`, convert route handlers
  to `async def`.
