# Research: Team Kanban Board

**Branch**: `001-team-kanban-board` | **Date**: 2026-11-17

All technology choices were explicitly provided; this document records integration
pattern decisions that resolve ambiguities in how those choices wire together.

---

## Decision 1: SQLAlchemy Sync vs Async with PyMySQL

**Decision**: Use SQLAlchemy 2.0 in **synchronous** mode with the `pymysql` driver.

**Rationale**: The user specified PyMySQL as the driver. PyMySQL is a synchronous
driver and cannot be used with SQLAlchemy's async engine (which requires `aiomysql`
or `asyncmy`). Using sync SQLAlchemy with PyMySQL is simpler, well-documented, and
sufficient for phase 1 load characteristics (5 users, desktop browser).

**How it surfaces in code**:
- Engine: `create_engine("mysql+pymysql://...")` (not `create_async_engine`)
- Sessions: `SessionLocal = sessionmaker(...)` with `Session` type
- Route dependencies inject `db: Session = Depends(get_db)` via a generator
- Tests use `TestClient` (sync) from httpx; no `AsyncClient` needed

**Alternatives considered**:
- `aiomysql` + async SQLAlchemy: rejected — requires driver swap and async boilerplate
  not justified by phase 1 requirements
- Raw SQL via `pymysql`: rejected — ORM provides migration-safe schema management via
  Alembic autogenerate

---

## Decision 2: User Identity Without Authentication

**Decision**: Current user is identified by an `X-User-Id` integer header on every
write request. The backend validates it is one of the five seeded user IDs; rejects
with HTTP 401 if missing or invalid on write endpoints.

**Rationale**: The spec requires "no login for this phase" but comments and tasks
must be attributed to a specific user. A header-based identity is the simplest
server-side mechanism that (a) keeps attribution correct, (b) is enforced
server-side (Constitution Principle I), and (c) can be replaced by proper JWT
auth in phase 2 without changing API signatures.

**How it surfaces in code**:
- Backend `deps.py`: `get_current_user(x_user_id: int = Header(...), db: Session)`
  — looks up User by ID, raises HTTP 401 if not found
- Frontend `apiClient.ts`: reads `userStore` and injects `X-User-Id` into every
  request via the generated client's interceptor/middleware
- Read-only endpoints (GET) do not require the header

**Alternatives considered**:
- Session cookie: rejected — requires session storage infrastructure not in scope
- JWT token from a login endpoint: rejected — spec explicitly excludes login for
  phase 1; would contradict FR-001

---

## Decision 3: OpenAPI Client Generation Workflow

**Decision**: Use `@hey-api/openapi-ts` to generate the typed client from the
backend's live `/openapi.json`. An npm script regenerates `src/client/` on demand.

**How it surfaces in code**:

`package.json` script:
```json
"generate-client": "openapi-ts --input http://localhost:8000/openapi.json --output src/client --client @hey-api/client-fetch"
```

Workflow:
1. Backend must be running (`docker compose up backend`) before generation
2. Developer runs `npm run generate-client` after any backend schema change
3. `src/client/` is committed to the repo (generated code is stable and reviewable)
4. `src/client/` files are marked with a header comment and listed in `.eslintignore`
   to prevent linting of generated code

**Implications**:
- Every FastAPI endpoint MUST have explicit `response_model=` and typed request body
  for the generated client to be fully typed (stated constraint)
- Pydantic v2 `model_config = ConfigDict(from_attributes=True)` required on response
  schemas so SQLAlchemy model instances can be passed directly to response serialization

**Alternatives considered**:
- Manual API client: rejected — loses type safety and defeats the stated constraint
- GraphQL: rejected — not in the specified stack

---

## Decision 4: Seed Data Strategy

**Decision**: Seed data (5 users + 3 sample projects + representative tasks across
columns) is inserted via a startup lifespan function in `main.py` using an
idempotent upsert pattern.

**Rationale**: FR-010 requires the three sample projects to be visible immediately
on first launch. Using Alembic data migrations would couple seed data to schema
migrations; a lifespan hook is simpler and idempotent (safe to re-run).

**How it surfaces in code**:
- `main.py` lifespan: on startup, calls `seed/seed_data.py::run_seed(db)` which
  checks if users exist before inserting (idempotent)
- Test `conftest.py` calls the same seed function after creating tables in the
  test database

**Alternatives considered**:
- Alembic data migration: rejected — ties seed data to schema versioning;
  harder to update independently
- Fixture file (JSON/YAML): rejected — adds parsing layer; direct Python is simpler

---

## Decision 5: Alembic Configuration with SQLAlchemy 2.0

**Decision**: Use Alembic autogenerate with all models imported in `alembic/env.py`.

**How it surfaces in code**:
```python
# alembic/env.py
from app.models import Base  # triggers all model imports via __init__.py
target_metadata = Base.metadata
```

The `alembic.ini` `sqlalchemy.url` is overridden at runtime from `app/settings.py`
so the URL stays in `.env` and is never duplicated.

---

## Decision 6: Kanban Column Representation

**Decision**: Column values are stored as a string enum in MySQL with these values:
`todo`, `in_progress`, `in_review`, `done`.

**How it surfaces in code**:
- Python: `KanbanColumn(str, Enum)` in `schemas/common.py`; SQLAlchemy column
  uses `Enum(KanbanColumn)` mapped to `VARCHAR(20)`
- API: column field serializes/deserializes as these exact string values
- Frontend: display labels mapped in a constant:
  `{ todo: "To Do", in_progress: "In Progress", in_review: "In Review", done: "Done" }`

---

## Decision 7: Comment Edit Tracking

**Decision**: The Comment entity stores `is_edited: bool` (default `False`).
When any user edits a comment body, `is_edited` is set to `True`. The UI displays
an "(edited)" indicator. The original `created_at` timestamp and author are unchanged.

**Rationale**: Clarification session Q2 confirmed any user can edit any comment.
Keeping the edit indicator gives readers context without a full edit history.

**Alternatives considered**:
- Full edit history table: rejected — over-engineered for phase 1; no requirement
  for revision history
- No edit indicator: rejected — misleading; readers would see edited content as
  original

---

## Decision 8: Docker Compose Service Configuration

**Decision**: Three services — `mysql`, `backend`, `frontend` — per stated requirement.

**Key details**:
- `mysql`: `mysql:8`, named volume for data, healthcheck so backend waits for DB ready
- `backend`: mounts `./backend` for hot reload via `--reload`; depends on mysql healthcheck
- `frontend`: Vite dev server (`npm run dev -- --host`) with hot module replacement;
  proxies `/api` and `/openapi.json` to backend via `vite.config.ts` proxy config
  (avoids CORS in dev without changing backend CORS settings)

**Vite proxy config**:
```typescript
// vite.config.ts
server: {
  proxy: {
    '/api': 'http://backend:8000',
    '/openapi.json': 'http://backend:8000',
    '/health': 'http://backend:8000',
  }
}
```
