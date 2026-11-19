# Implementation Plan: Team Kanban Board

**Branch**: `001-team-kanban-board` | **Date**: 2026-11-17 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-team-kanban-board/spec.md`

## Summary

Build Taskify phase 1: a team Kanban board where five predefined users (no login) manage
projects and tasks across four columns (To Do / In Progress / In Review / Done). The
implementation uses a FastAPI + MySQL backend with a generated TypeScript client consumed
by a React 18 frontend, all orchestrated by Docker Compose.

The first deliverable milestone is a working full-stack skeleton — Docker Compose up,
health check, and a fully typed Users list endpoint — proving the entire
FastAPI → SQLAlchemy → MySQL → OpenAPI → generated client → React chain before the
Kanban domain is built on top.

## Technical Context

**Language/Version**: Python 3.12 (backend), TypeScript 5 strict mode (frontend)

**Primary Dependencies**:
- Backend: FastAPI, Pydantic v2, SQLAlchemy 2.0 (sync), PyMySQL, Alembic,
  pydantic-settings, uv (package manager), ruff, mypy
- Frontend: React 18, Vite, React Router, Tailwind CSS,
  @hey-api/openapi-ts (client generator), Vitest, React Testing Library, Playwright,
  ESLint, Prettier

**Storage**: MySQL 8 (application DB) + separate MySQL database for tests

**Testing**:
- Backend: pytest + httpx AsyncClient / TestClient against a real test MySQL DB
- Frontend: Vitest + React Testing Library (components), Playwright (one E2E flow)

**Target Platform**: Desktop browser (1280px+ viewport); Docker Compose for local dev

**Project Type**: Full-stack web application (backend API + SPA frontend)

**Performance Goals**: Board loads within 3 s on first launch (SC-004);
task column move completes in under 5 s (SC-001); validation errors visible
within 1 s (SC-007)

**Constraints**:
- No login/authentication in phase 1; current user passed via `X-User-Id` request header
- Every API endpoint MUST have Pydantic v2 request/response schemas
  (required for fully typed OpenAPI → generated client)
- Starter baseline first: health check + Users list → then full Kanban domain
- All secrets and configuration via pydantic-settings / `.env`; `.env.example` committed
- Generated frontend client (`src/client/`) is never hand-edited

**Scale/Scope**: 5 predefined users, ≤ unlimited projects/tasks (phase 1 has no scale
requirement beyond desktop browser session)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

### Principle I — Security-First

| Check | Status | Notes |
|---|---|---|
| Secrets in env / secret manager | PASS | pydantic-settings + .env; .env committed only as .env.example |
| Input validation server-side | PASS | Pydantic v2 schemas on every request body; centralized per router |
| Generic errors to client | PASS | FastAPI exception handlers return structured errors; stack traces suppressed |
| Auth/authZ enforced server-side | ⚠️ JUSTIFIED DEVIATION | See below |
| TLS in production | DEFERRED | Out of scope for local Docker dev; enforced at deployment layer |

**Deviation — No authentication (Principle I):**
The spec explicitly scopes "no login for this first phase." All five users are
predefined internal team members operating in a trusted local environment. User identity
is conveyed via `X-User-Id` header; the backend validates it is one of the five seeded
IDs. This is a time-bounded risk acceptance: authentication (OAuth2/JWT) MUST be
implemented before any network-exposed or multi-tenant deployment. This deviation
MUST be tracked as a known gap and revisited in phase 2.

### Principle II — Input Validation


| Check | Status | Notes |
|---|---|---|
| All inputs validated server-side | PASS | Pydantic v2 schemas with field constraints |
| Allow-list schema (type, range, length, format) | PASS | Max lengths in schema; enums for column values |
| Validation errors inline | PASS | 422 response body maps field → error message; frontend displays inline |
| Centralized validation per service | PASS | One schema file per domain; no scattered ad-hoc checks |

### Principle III — Microservices Architecture

| Check | Status | Notes |
|---|---|---|
| Independently deployable services | ⚠️ JUSTIFIED DEVIATION | See below |
| Services own their data store | PASS | Single DB in phase 1; no cross-service DB access |
| Versioned contracts between services | PASS | OpenAPI spec is the contract; generated client enforces it |
| Health-check / readiness endpoint | PASS | GET /health required |

**Deviation — Monolith (Principle III):**
Phase 1 is a single FastAPI app + single React SPA. The microservices split is
deferred until domain boundaries are validated. The layered structure
(routes → services → repositories → models) maps 1:1 to future service extractions:
each domain module (users, projects, tasks, comments) can be extracted into an
independent service without cross-module coupling. This is the standard "modular
monolith as a microservices precursor" pattern.

### Principle IV — Full Documentation

| Check | Status | Notes |
|---|---|---|
| Docstrings on all public functions/endpoints | REQUIRED | Enforced by mypy + ruff in CI |
| Service README with setup + API reference | REQUIRED | backend/README.md, frontend/README.md |
| Inline comments for non-obvious logic | REQUIRED | Reviewers must flag missing explanations |

**Post-Phase-1 re-evaluation**: Both justified deviations are time-bounded and
explicitly documented here. The plan is approved to proceed.

## Project Structure

### Documentation (this feature)

```text
specs/001-team-kanban-board/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── api.md           # REST endpoint contracts
└── tasks.md             # Phase 2 output (/speckit-tasks — not created here)
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                   # FastAPI app factory + lifespan (seed data)
│   ├── settings.py               # pydantic-settings (DATABASE_URL, TEST_DATABASE_URL)
│   ├── database.py               # SQLAlchemy engine, SessionLocal, Base
│   ├── models/                   # SQLAlchemy ORM models (table definitions)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── comment.py
│   ├── schemas/                  # Pydantic v2 request/response schemas
│   │   ├── __init__.py
│   │   ├── common.py             # Shared types (e.g., KanbanColumn enum)
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── comment.py
│   ├── repositories/             # DB access only — no business logic
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── comment.py
│   ├── services/                 # Business logic — no direct DB calls
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── comment.py
│   └── api/
│       ├── __init__.py
│       ├── deps.py               # Shared deps: DB session, resolve X-User-Id
│       └── routes/
│           ├── __init__.py
│           ├── health.py         # GET /health
│           ├── users.py          # GET /api/v1/users
│           ├── projects.py       # /api/v1/projects
│           ├── tasks.py          # /api/v1/projects/{id}/tasks, /api/v1/tasks/{id}
│           └── comments.py       # /api/v1/tasks/{id}/comments, /api/v1/comments/{id}
├── alembic/
│   ├── env.py                    # imports app.models for autogenerate
│   ├── script.py.mako
│   └── versions/
│       └── 0001_initial_schema.py
├── seed/
│   └── seed_data.py              # 5 users + 3 sample projects + tasks
├── tests/
│   ├── conftest.py               # DB fixtures, TestClient, seed helpers
│   ├── test_health.py
│   ├── test_users.py
│   ├── test_projects.py
│   ├── test_tasks.py
│   └── test_comments.py
├── alembic.ini
├── pyproject.toml                # uv project: deps, ruff, mypy config
├── .env.example                  # committed; .env is gitignored
└── Dockerfile

frontend/
├── src/
│   ├── client/                   # Generated by @hey-api/openapi-ts (never hand-edited)
│   ├── components/
│   │   ├── board/
│   │   │   ├── KanbanBoard.tsx   # Four-column layout
│   │   │   ├── KanbanColumn.tsx  # Single column + task cards
│   │   │   └── TaskCard.tsx      # Title + assignee + creator (FR-007a)
│   │   ├── tasks/
│   │   │   ├── TaskDetail.tsx    # Full task view + comments
│   │   │   ├── TaskForm.tsx      # Create / edit task form
│   │   │   └── CommentList.tsx   # Comments + add/edit/delete
│   │   ├── projects/
│   │   │   ├── ProjectList.tsx
│   │   │   └── ProjectForm.tsx
│   │   └── ui/
│   │       ├── UserSelector.tsx  # Identity picker (FR-001, FR-014)
│   │       ├── EmptyState.tsx
│   │       └── ConfirmDialog.tsx # Comment delete confirmation (FR-008b)
│   ├── pages/
│   │   ├── ProjectsPage.tsx      # Route: /
│   │   └── BoardPage.tsx         # Route: /projects/:projectId
│   ├── hooks/
│   │   ├── useCurrentUser.ts     # Read/set current user from store
│   │   ├── useProjects.ts
│   │   ├── useTasks.ts
│   │   └── useComments.ts
│   ├── store/
│   │   └── userStore.ts          # Current user identity (sessionStorage)
│   ├── lib/
│   │   └── apiClient.ts          # Configures generated client base URL + X-User-Id header
│   ├── App.tsx
│   ├── main.tsx
│   └── vite-env.d.ts
├── tests/
│   ├── components/
│   │   ├── TaskCard.test.tsx
│   │   ├── TaskForm.test.tsx
│   │   └── CommentList.test.tsx
│   └── e2e/
│       └── board.spec.ts         # Playwright: create task → move column → verify
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.ts
├── playwright.config.ts
└── Dockerfile

docs/
docker-compose.yml                # services: mysql, backend, frontend
README.md                         # docker compose up quickstart
```

**Structure Decision**: Web application (Option 2). Backend and frontend are separate
top-level directories sharing a single Docker Compose file. Each is independently
containerisable. The layered backend module structure
(routes → services → repositories → models) is designed so each domain (users,
projects, tasks, comments) can be extracted into an independent microservice in a
future phase without cross-domain coupling.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| No authentication (Principle I) | Spec explicitly scopes "no login for phase 1"; 5 internal team members in trusted local environment | Adding OAuth2/JWT for a team-only prototype delays core Kanban delivery; risk is bounded by explicit phase scope and must be resolved before phase 2 |
| Monolith (Principle III) | Phase 1 starter must run with one `docker compose up` and validate the full stack before domain boundaries are proven | 4 microservices for a starter kit creates service-mesh overhead, inter-service auth, and distributed tracing complexity that blocks MVP delivery |
