# Tasks: Team Kanban Board

**Input**: Design documents from `specs/001-team-kanban-board/`

**Branch**: `001-team-kanban-board` | **Generated**: 2026-11-19
**Architecture**: [High-Level Design](../../../docs/hld.md)

**Prerequisites**: plan.md ✓ | spec.md ✓ | research.md ✓ | data-model.md ✓ | contracts/api.md ✓ | quickstart.md ✓

**Organization**: Tasks are grouped by user story to enable independent implementation and
testing of each story. Tests are included as part of each story's deliverable per plan.md.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks in this phase)
- **[Story]**: Which user story this task belongs to (US1–US4)
- Exact file paths derive from `plan.md` Project Structure section

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create all files and directory scaffolding needed before writing application code.
Everything here can be done before the database or any service is running.

- [ ] T001 Create root-level `docker-compose.yml` with three services: `mysql` (MySQL 8, port 3306, volume for data), `backend` (Dockerfile build, port 8000, hot reload, depends_on mysql), `frontend` (Dockerfile build, port 5173, Vite dev server, depends_on backend); include env_file references for backend secrets
- [ ] T002 Create `README.md` at repo root with quickstart section: `docker compose up`, service URLs, and links to `specs/001-team-kanban-board/quickstart.md` for full validation guide
- [ ] T003 [P] Create `backend/` directory scaffold per plan.md: `app/__init__.py`, `app/models/__init__.py`, `app/schemas/__init__.py`, `app/repositories/__init__.py`, `app/services/__init__.py`, `app/api/__init__.py`, `app/api/routes/__init__.py`, `seed/`, `tests/`, `alembic/versions/`
- [ ] T004 [P] Create `backend/pyproject.toml` with uv project config: runtime deps (fastapi, pydantic>=2, pydantic-settings, sqlalchemy>=2, pymysql, alembic, uvicorn), dev deps (pytest, httpx, ruff, mypy); configure ruff and mypy in `[tool.ruff]` and `[tool.mypy]` sections
- [ ] T005 [P] Create `backend/.env.example` with all required env vars: `DATABASE_URL=mysql+pymysql://taskify:taskify@mysql:3306/taskify`, `TEST_DATABASE_URL=mysql+pymysql://taskify:taskify@mysql:3306/taskify_test`
- [ ] T006 [P] Create `backend/Dockerfile`: Python 3.12 slim base, install uv, copy pyproject.toml, run `uv sync`, copy app source, `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]`
- [ ] T007 [P] Create `frontend/` directory scaffold per plan.md: `src/client/`, `src/components/board/`, `src/components/tasks/`, `src/components/projects/`, `src/components/ui/`, `src/pages/`, `src/hooks/`, `src/store/`, `src/lib/`, `tests/components/`, `tests/e2e/`
- [ ] T008 [P] Create `frontend/package.json` with deps (react@18, react-dom, react-router-dom, @hey-api/openapi-ts, @hey-api/client-fetch) and devDeps (typescript, vite, @vitejs/plugin-react, tailwindcss, vitest, @testing-library/react, playwright, eslint, prettier); add scripts: `dev`, `build`, `generate-client` (`openapi-ts --input http://localhost:8000/openapi.json --output src/client --client @hey-api/client-fetch`), `test`, `lint`
- [ ] T009 [P] Create `frontend/tsconfig.json` (strict mode: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"exactOptionalPropertyTypes": true`), `frontend/vite.config.ts` (React plugin + server proxy: `/api`, `/health`, `/openapi.json` → `http://backend:8000`), `frontend/tailwind.config.ts`, `frontend/playwright.config.ts`
- [ ] T010 [P] Create `frontend/Dockerfile`: Node 20 slim base, copy package.json, run `npm ci`, copy source, `CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]`

**Checkpoint**: All scaffolding created. `docker compose build` should succeed (no application code needed yet).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core backend infrastructure that MUST be complete before ANY user story begins.
Includes all ORM models (required together for the single initial Alembic migration), the
database layer, seed data, app factory, identity resolver, health endpoint, and the
frontend identity chain (userStore → UserSelector → generated client). Validates the full
FastAPI → SQLAlchemy → MySQL → OpenAPI → generated client → React chain (Milestone 1 of quickstart.md).

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T011 Implement `backend/app/settings.py` using pydantic-settings: `class Settings(BaseSettings)` with `DATABASE_URL: str`, `TEST_DATABASE_URL: str`; `model_config = SettingsConfigDict(env_file=".env")`; module-level `settings = Settings()`
- [ ] T012 Implement `backend/app/database.py`: create sync SQLAlchemy engine from `settings.DATABASE_URL` using `create_engine`; `SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)`; `Base = declarative_base()`
- [ ] T013 [P] Create `backend/app/models/user.py`: SQLAlchemy `User` model, table `users`; columns: `id` (Integer PK AUTO_INCREMENT), `name` (String(100) NOT NULL), `role` (Enum('product_manager','engineer') NOT NULL); no `created_at` (seed-only entity)
- [ ] T014 [P] Create `backend/app/models/project.py`: SQLAlchemy `Project` model, table `projects`; columns: `id` (Integer PK), `name` (String(100) NOT NULL), `description` (Text nullable), `creator_id` (Integer FK→users.id NOT NULL, RESTRICT on delete), `created_at` (DateTime NOT NULL server_default=func.now()); relationship: `creator` → User, `tasks` → Task (cascade='all, delete-orphan')
- [ ] T015 [P] Create `backend/app/models/task.py`: SQLAlchemy `Task` model, table `tasks`; columns: `id` (Integer PK), `project_id` (Integer FK→projects.id NOT NULL, CASCADE), `title` (String(100) NOT NULL), `description` (Text nullable), `column` (Enum('todo','in_progress','in_review','done') NOT NULL default='todo'), `assignee_id` (Integer FK→users.id NOT NULL, RESTRICT), `creator_id` (Integer FK→users.id NOT NULL, RESTRICT), `created_at` (DateTime NOT NULL server_default=func.now()); add `__table_args__` with composite index on `(project_id, column)` and index on `(assignee_id)`; relationships: `project`, `assignee`, `creator`, `comments` (cascade delete)
- [ ] T016 [P] Create `backend/app/models/comment.py`: SQLAlchemy `Comment` model, table `comments`; columns: `id` (Integer PK), `task_id` (Integer FK→tasks.id NOT NULL, CASCADE), `body` (Text NOT NULL), `author_id` (Integer FK→users.id NOT NULL, RESTRICT), `is_edited` (Boolean NOT NULL default=False), `created_at` (DateTime NOT NULL server_default=func.now()); add `__table_args__` with composite index on `(task_id, created_at)`; relationships: `task`, `author`
- [ ] T017 Create `backend/app/schemas/common.py`: `class KanbanColumn(str, Enum)` with values `TODO='todo'`, `IN_PROGRESS='in_progress'`, `IN_REVIEW='in_review'`, `DONE='done'`; `COLUMN_DISPLAY_LABELS` dict mapping each value to its display string
- [ ] T018 [P] Create `backend/app/schemas/user.py`: `class UserResponse(BaseModel)` with fields `id: int`, `name: str`, `role: str`; `model_config = ConfigDict(from_attributes=True)`
- [ ] T018a [P] Create `backend/app/schemas/comment.py` (Phase 2 stub — response schema only): `class CommentResponse(BaseModel)` with `id: int`, `task_id: int`, `body: str`, `author: UserResponse`, `is_edited: bool`, `created_at: datetime`; `model_config = ConfigDict(from_attributes=True)`; `CommentCreate` and `CommentUpdate` are added in Phase 6 (T072) — this stub allows Phase 3 (T037) to import `CommentResponse` for `TaskDetailResponse` without a forward-reference or circular-import error
- [ ] T019 Configure `backend/alembic.ini` pointing `sqlalchemy.url` to `%(DATABASE_URL)s`; create `backend/alembic/env.py` that imports `Base` and all four models (`user`, `project`, `task`, `comment`) for autogenerate and reads `DATABASE_URL` from environment
- [ ] T020 Generate and write `backend/alembic/versions/0001_initial_schema.py` (initial migration): creates tables `users`, `projects`, `tasks`, `comments` with all columns, FK constraints, and indexes as defined in models T013–T016; this is the single migration that creates the entire schema
- [ ] T021 Implement `backend/seed/seed_data.py`: idempotent function `seed_database(db: Session) → None` that checks before inserting; seeds exactly 5 users: id=1 Alice Chen/product_manager, id=2 Bob Kumar/engineer, id=3 Carol Santos/engineer, id=4 David Park/engineer, id=5 Eva Müller/engineer; seeds 3 sample projects (creator_id=1) with representative tasks distributed across all 4 columns
- [ ] T022 Implement `backend/app/main.py`: `create_app()` factory returning FastAPI instance; `@asynccontextmanager` lifespan that runs `alembic upgrade head` then calls `seed_database`; register routers for health, users, projects, tasks, comments; include `app = create_app()` module-level for uvicorn
- [ ] T023 Implement `backend/app/api/deps.py`: `get_db()` generator yielding `SessionLocal` with `finally: db.close()`; `get_current_user(x_user_id: int = Header(None), db: Session = Depends(get_db)) → User` that queries `users` table for the given ID and raises `HTTPException(status_code=401, detail="Valid X-User-Id header required")` if missing or not in seeded set (1–5)
- [ ] T024 [P] Create `backend/app/schemas/health.py`: `class HealthResponse(BaseModel)` with `status: str` and `db: str`
- [ ] T025 [P] Implement `backend/app/api/routes/health.py`: `GET /health` endpoint using `HealthResponse`; performs a `SELECT 1` to check DB; returns `{"status":"ok","db":"ok"}` on success or `{"status":"ok","db":"error"}` if DB unreachable; no authentication required
- [ ] T026 Implement `backend/app/repositories/user.py`: `UserRepository` class with `get_all(db) → list[User]` and `get_by_id(db, user_id: int) → User | None`
- [ ] T027 Implement `backend/app/services/user.py`: `UserService` with `get_all(db) → list[User]`; depends on `UserRepository`
- [ ] T028 [P] Implement `backend/app/api/routes/users.py`: `GET /api/v1/users` returning `list[UserResponse]`; calls `UserService.get_all`; no authentication required
- [ ] T029 Create `backend/tests/conftest.py`: pytest fixtures using `TEST_DATABASE_URL`; `engine` fixture running `alembic upgrade head` before session; `db` fixture providing `TestClient` with overridden `get_db` dependency; `seed_db` fixture calling `seed_database`
- [ ] T030 Write `backend/tests/test_health.py`: test `GET /health` returns HTTP 200 with `{"status":"ok","db":"ok"}`
- [ ] T031 Write `backend/tests/test_users.py`: test `GET /api/v1/users` returns HTTP 200 with exactly 5 users; verify names and roles match seed data (Alice Chen/product_manager, Bob Kumar/engineer, Carol Santos/engineer, David Park/engineer, Eva Müller/engineer)
- [ ] T032 Implement `frontend/src/store/userStore.ts`: Zustand or plain sessionStorage store with `currentUserId: number | null` and `setCurrentUser(id: number): void`; persists to `sessionStorage` so identity survives page refresh within the same tab
- [ ] T033 Implement `frontend/src/lib/apiClient.ts`: configure `@hey-api/client-fetch` base URL (`http://localhost:8000` in dev, resolved via Vite proxy); intercept all requests to inject `X-User-Id` header from `userStore` for write operations
- [ ] T034 Create `frontend/src/components/ui/UserSelector.tsx`: renders a list/dropdown of all 5 users fetched from `GET /api/v1/users`; calls `userStore.setCurrentUser` on selection; FR-001; shown when no user is selected in `sessionStorage`
- [ ] T035 Create `frontend/src/hooks/useCurrentUser.ts`: reads `currentUserId` from `userStore`; fetches user detail from cached users list; exposes `currentUser`, `setCurrentUser`
- [ ] T035a Create `frontend/src/components/ui/IdentityBadge.tsx`: persistent header component displaying `currentUser.name` and `currentUser.role`; includes a "Switch User" button that opens `UserSelector` as an overlay without navigating away (FR-014 — identity must remain persistently visible and switchable at any time); wire into `App.tsx` layout above the router outlet so it appears on every page

**Checkpoint**: `docker compose up` → health check passes, `GET /api/v1/users` returns 5 users, `npm run generate-client` succeeds, `src/client/` generated with zero TypeScript errors (Milestone 1 of quickstart.md). Frontend loads with UserSelector visible.

---

## Phase 3: User Story 1 — View Projects and Kanban Board (Priority: P1) 🎯 MVP

**Goal**: Any user can select their identity, see all projects, navigate into a project, and view the
Kanban board with tasks distributed across the four columns. Three sample projects are pre-seeded.

**Independent Test**: Select any identity → see three sample projects on main screen (< 3 s, SC-004) →
click into a project → four columns displayed with tasks in correct positions (SC-002).

### Implementation for User Story 1

- [ ] T036 [P] [US1] Create `backend/app/schemas/project.py`: `class ProjectResponse(BaseModel)` with `id: int`, `name: str`, `description: str | None`, `creator: UserResponse`, `created_at: datetime`; `model_config = ConfigDict(from_attributes=True)`
- [ ] T037 [P] [US1] Create `backend/app/schemas/task.py`: `class TaskResponse(BaseModel)` with `id: int`, `project_id: int`, `title: str`, `description: str | None`, `column: KanbanColumn`, `assignee: UserResponse`, `creator: UserResponse`, `created_at: datetime`; `class TaskDetailResponse(TaskResponse)` extending with `comments: list[CommentResponse]`; import `CommentResponse` from `schemas/comment.py` (created in T018a — do NOT redefine it here); `model_config = ConfigDict(from_attributes=True)`
- [ ] T038 [P] [US1] Implement `backend/app/repositories/project.py`: `ProjectRepository` with `get_all(db) → list[Project]` (order by `created_at` desc) and `get_by_id(db, project_id: int) → Project | None`
- [ ] T039 [P] [US1] Implement `backend/app/services/project.py`: `ProjectService` with `get_all(db) → list[Project]` and `get_by_id(db, project_id: int) → Project` (raises 404 if not found)
- [ ] T040 [US1] Implement `backend/app/api/routes/projects.py` (read endpoints only): `GET /api/v1/projects` returning `list[ProjectResponse]`; `GET /api/v1/projects/{project_id}` returning `ProjectResponse` (404 if not found); no auth required
- [ ] T041 [P] [US1] Implement `backend/app/repositories/task.py`: `TaskRepository` with `get_by_project_id(db, project_id: int) → list[Task]` (order by `created_at` asc) and `get_by_id(db, task_id: int) → Task | None`
- [ ] T042 [P] [US1] Implement `backend/app/services/task.py`: `TaskService` with `get_by_project_id(db, project_id: int) → list[Task]` (raises 404 if project not found) and `get_by_id(db, task_id: int) → Task` (raises 404 if not found)
- [ ] T043 [US1] Implement read endpoints in `backend/app/api/routes/tasks.py`: `GET /api/v1/projects/{project_id}/tasks` returning `list[TaskResponse]` (404 if project not found); `GET /api/v1/tasks/{task_id}` returning `TaskDetailResponse` with `comments` populated from the SQLAlchemy `task.comments` relationship (empty list `[]` when no comments exist — `CommentRepository` is not yet wired; the ORM relationship on `Task` model already eager-loads or lazy-loads `comments`); 404 if not found; no auth required
- [ ] T044 [P] [US1] Create `frontend/src/hooks/useProjects.ts`: fetches `GET /api/v1/projects` using generated client; returns `{ projects, isLoading, error }`
- [ ] T045 [P] [US1] Create `frontend/src/hooks/useTasks.ts`: fetches `GET /api/v1/projects/{projectId}/tasks` using generated client; returns `{ tasks, isLoading, error }` keyed by `projectId`
- [ ] T046 [P] [US1] Create `frontend/src/components/ui/EmptyState.tsx`: accepts `message: string` prop; renders a styled empty-state block (no tasks in this column, no projects yet, etc.)
- [ ] T047 [P] [US1] Create `frontend/src/components/board/TaskCard.tsx`: receives `task: TaskResponse`; displays title, `task.assignee.name`, `task.creator.name` (FR-007a); no interaction beyond click-to-open detail (wired in Phase 6)
- [ ] T048 [P] [US1] Create `frontend/src/components/board/KanbanColumn.tsx`: receives `column: KanbanColumn`, `tasks: TaskResponse[]`, `columnLabel: string`; renders column header + list of `TaskCard`; shows `EmptyState` when `tasks.length === 0` (FR-004, edge case)
- [ ] T049 [US1] Create `frontend/src/components/board/KanbanBoard.tsx`: receives `projectId: number`; calls `useTasks`; groups tasks by `column` value into the four fixed columns (todo→To Do, in_progress→In Progress, in_review→In Review, done→Done) in fixed left-to-right order; renders four `KanbanColumn` components; must fit 1280px viewport without horizontal scroll (SC-002)
- [ ] T050 [P] [US1] Create `frontend/src/components/projects/ProjectList.tsx`: receives `projects: ProjectResponse[]`; renders project name cards; each card navigates to `/projects/:id`; shows `EmptyState` when no projects
- [ ] T051 [P] [US1] Create `frontend/src/pages/ProjectsPage.tsx`: route `/`; calls `useProjects`; wraps `ProjectList`; sample projects visible within 3 s (SC-004)
- [ ] T052 [P] [US1] Create `frontend/src/pages/BoardPage.tsx`: route `/projects/:projectId`; renders `KanbanBoard` for the given project ID; shows project name in heading
- [ ] T053 [US1] Implement `frontend/src/App.tsx` with React Router: `<Route path="/" element={<ProjectsPage />}>`; `<Route path="/projects/:projectId" element={<BoardPage />}>`; wrap with `UserSelector` gate (redirect or overlay if no identity selected); `frontend/src/main.tsx` mounts `<App />`
- [ ] T054 [P] [US1] Write `backend/tests/test_projects.py` (read coverage): `GET /api/v1/projects` returns 3 seeded projects; `GET /api/v1/projects/{id}` returns correct project; `GET /api/v1/projects/999` returns 404; `GET /api/v1/projects/{id}/tasks` returns tasks for seeded project

**Checkpoint**: `docker compose up` → identity selection → three sample projects visible → board shows 4 columns with tasks in correct positions. Validates US1 independently (Milestones 2 and 3 of quickstart.md).

---

## Phase 4: User Story 2 — Create and Assign Tasks (Priority: P2)

**Goal**: Any authenticated user can create a new project and create tasks within a project, assigning
each task to one of the five predefined team members. New tasks appear immediately in the To Do column.
Input validation rejects empty titles, invalid assignees, and over-length fields (FR-011).

**Independent Test**: Select identity → open Create Task form → fill valid title/description/assignee →
submit → task appears in To Do column with correct assignee name and creator name (SC-005: < 60 s).

### Implementation for User Story 2

- [ ] T055 [P] [US2] Add `ProjectCreate` schema to `backend/app/schemas/project.py`: `name: Annotated[str, Field(min_length=1, max_length=100)]` (required, max 100 chars per FR-011), `description: Annotated[str | None, Field(max_length=2000)] = None` (optional, max 2000 chars per FR-011); add `@field_validator('name', mode='before')` that strips leading/trailing whitespace and raises `ValueError` if the stripped result is empty (prevents whitespace-only names consistent with CommentCreate validator)
- [ ] T056 [P] [US2] Add `TaskCreate` schema to `backend/app/schemas/task.py`: `title: Annotated[str, Field(min_length=1, max_length=100)]` (required, max 100 chars per FR-011), `description: Annotated[str | None, Field(max_length=2000)] = None` (optional per FR-011), `assignee_id: Annotated[int, Field(ge=1, le=5)]` (required; must be one of the 5 seeded IDs per data-model.md); add `@field_validator('title', mode='before')` that strips leading/trailing whitespace and raises `ValueError` if the stripped result is empty (prevents whitespace-only titles consistent with CommentCreate validator)
- [ ] T057 [US2] Add `create(db, data: ProjectCreate, creator_id: int) → Project` to `backend/app/repositories/project.py`; add `create_project(db, data, creator_id) → Project` to `backend/app/services/project.py`
- [ ] T058 [US2] Add `POST /api/v1/projects` to `backend/app/api/routes/projects.py`: body `ProjectCreate`, requires `X-User-Id` via `get_current_user` dep; sets `creator_id` from resolved user; returns HTTP 201 `ProjectResponse`; rejects invalid body with 422
- [ ] T059 [US2] Add `create(db, data: TaskCreate, project_id: int, creator_id: int) → Task` to `backend/app/repositories/task.py`; add `create_task(db, data, project_id, creator_id) → Task` to `backend/app/services/task.py` (raises 404 if project not found; validates `assignee_id` exists in seeded users)
- [ ] T060 [US2] Add `POST /api/v1/projects/{project_id}/tasks` to `backend/app/api/routes/tasks.py`: body `TaskCreate`, requires `X-User-Id`; new task always starts in `column='todo'`; returns HTTP 201 `TaskResponse`; rejects invalid body with 422 (empty title, assignee_id outside 1–5, title > 100 chars per FR-011)
- [ ] T061 [P] [US2] Create `frontend/src/components/projects/ProjectForm.tsx`: modal or inline form with `name` field (required, max 100 chars, FR-003); `description` field (optional, max 2000 chars); client-side validation shows inline error within 1 s (SC-007); on success calls `POST /api/v1/projects` via generated client and refreshes project list
- [ ] T062 [P] [US2] Create `frontend/src/components/tasks/TaskForm.tsx`: form with `title` field (required, max 100 chars), `description` field (optional, max 2000 chars), `assignee` dropdown populated from `GET /api/v1/users` (all 5 users, required per FR-005); client-side validation inline within 1 s (SC-007); on success calls `POST /api/v1/projects/{id}/tasks` and refreshes board
- [ ] T063 [US2] Wire `ProjectForm` into `ProjectsPage.tsx` ("Create Project" button) and `TaskForm` into `BoardPage.tsx` ("Create Task" button); update `useProjects` and `useTasks` hooks to refetch after mutations
- [ ] T064 [P] [US2] Write `backend/tests/test_projects.py` (write coverage): `POST /api/v1/projects` with valid body + X-User-Id returns 201; missing name → 422; name > 100 chars → 422; missing X-User-Id → 401; invalid X-User-Id → 401
- [ ] T065 [P] [US2] Write `backend/tests/test_tasks.py` (creation coverage): `POST /api/v1/projects/{id}/tasks` with valid body returns 201 with `column='todo'`; empty title → 422; title > 100 chars → 422; missing assignee_id → 422; assignee_id=6 → 422; missing X-User-Id → 401; project not found → 404

**Checkpoint**: Create a project → create a task within it → task appears in To Do column with correct assignee and creator. Validation rejects empty title, long title, missing assignee (Milestones 4 and 5 of quickstart.md).

---

## Phase 5: User Story 3 — Move Tasks Across Columns (Priority: P3)

**Goal**: Any user can move any task to any other column in any direction (FR-006). No transition
restrictions. Board updates immediately to reflect new position (SC-001: < 5 s).

**Independent Test**: Find a task in To Do → move to In Progress → task moves columns; move from
In Progress → Done (skip) → move from Done → To Do (backward). All work with no errors (SC-001).

### Implementation for User Story 3

- [ ] T066 [P] [US3] Add `TaskUpdate` schema to `backend/app/schemas/task.py`: all fields optional (`title: str | None = None` max 100 chars, `description: str | None = None` max 2000 chars, `column: KanbanColumn | None = None`, `assignee_id: int | None = None` ge=1 le=5); add `@model_validator(mode='after')` that raises `ValueError` if all four fields are None (at least one required per contracts)
- [ ] T067 [US3] Add `update(db, task: Task, data: TaskUpdate) → Task` to `backend/app/repositories/task.py`; add `update_task(db, task_id, data: TaskUpdate) → Task` to `backend/app/services/task.py` (raises 404 if task not found); apply only non-None fields (partial update semantics)
- [ ] T068 [US3] Add `PATCH /api/v1/tasks/{task_id}` to `backend/app/api/routes/tasks.py`: body `TaskUpdate`, requires `X-User-Id`; returns HTTP 200 `TaskResponse` (without comments); 404 if not found; 422 if no field provided or field validation fails
- [ ] T069 [P] [US3] Add column-move action to `frontend/src/components/board/KanbanColumn.tsx`: renders a move-to-column selector or drag-drop target that calls `PATCH /api/v1/tasks/{id}` with `{column: targetColumn}`; refreshes board on success; no transition restrictions (FR-006)
- [ ] T070 [P] [US3] Update `frontend/src/components/board/TaskCard.tsx`: add move-column affordance (dropdown or drag handle); calls PATCH via generated client; optimistically updates local task state or refetches board
- [ ] T071 [P] [US3] Write `backend/tests/test_tasks.py` (PATCH coverage): `PATCH /api/v1/tasks/{id}` with `{column:"in_progress"}` returns 200; `{column:"done"}` from any column → 200 (backward allowed, FR-006); `{column:"todo"}` from done → 200; empty body `{}` → 422; task not found → 404; missing X-User-Id → 401

**Checkpoint**: Move task To Do→In Progress; move In Progress→Done (skip); move Done→To Do (backward). All transitions succeed without error. Column change persists across page reload (SC-006, Milestone 6 of quickstart.md).

---

## Phase 6: User Story 4 — Comment on Tasks (Priority: P4)

**Goal**: Any user can open a task detail view, add comments, edit any comment (preserving original
author, adding "(edited)" indicator), and delete any comment with a confirmation prompt (FR-008,
FR-008a, FR-008b). Comments display in ascending chronological order (FR-009).

**Independent Test**: Open task detail → add comment → edit comment (see "edited" indicator) →
delete comment with confirmation → empty-state message shown. Whitespace-only comment rejected (SC-008).

### Implementation for User Story 4

- [ ] T072 [P] [US4] Extend `backend/app/schemas/comment.py` (stub created in T018a — `CommentResponse` already exists; do NOT redefine it): add `CommentCreate(BaseModel)`: `body: Annotated[str, Field(min_length=1, max_length=2000)]` with `@field_validator('body', mode='before')` that strips whitespace and raises `ValueError` if empty (non-whitespace required per FR-011); add `CommentUpdate(BaseModel)` with same `body` field and validator
- [ ] T073 [P] [US4] Implement `backend/app/repositories/comment.py`: `CommentRepository` with `get_by_task_id(db, task_id: int) → list[Comment]` (order by `created_at` asc), `create(db, data: CommentCreate, task_id: int, author_id: int) → Comment`, `get_by_id(db, comment_id: int) → Comment | None`, `update(db, comment: Comment, data: CommentUpdate) → Comment` (sets `is_edited=True`, updates `body`), `delete(db, comment: Comment) → None`
- [ ] T074 [US4] Implement `backend/app/services/comment.py`: `CommentService` with `get_by_task(db, task_id) → list[Comment]` (raises 404 if task not found), `create_comment(db, data, task_id, author_id) → Comment` (raises 404 if task not found), `update_comment(db, comment_id, data) → Comment` (raises 404 if not found), `delete_comment(db, comment_id) → None` (raises 404 if not found)
- [ ] T075 [US4] Implement `backend/app/api/routes/comments.py`: `GET /api/v1/tasks/{task_id}/comments` → `list[CommentResponse]` (no auth); `POST /api/v1/tasks/{task_id}/comments` → HTTP 201 `CommentResponse` (requires X-User-Id; 422 if body empty or whitespace-only or > 2000 chars); `PATCH /api/v1/comments/{comment_id}` → HTTP 200 `CommentResponse` with `is_edited=True` (requires X-User-Id; 422 if body invalid); `DELETE /api/v1/comments/{comment_id}` → HTTP 204 (requires X-User-Id; 404 if not found)
- [ ] T076 [US4] Wire `CommentService` into `GET /api/v1/tasks/{task_id}` in `backend/app/api/routes/tasks.py`: replace the Phase 3 ORM-relationship approach with an explicit `CommentService.get_by_task(db, task_id)` call; `TaskDetailResponse.comments` is already defined in T037 and schema is already imported — no schema changes needed; ensure comments are ordered by `created_at` asc
- [ ] T077 [P] [US4] Create `frontend/src/hooks/useComments.ts`: fetches `GET /api/v1/tasks/{taskId}/comments`; exposes `comments`, `addComment`, `editComment`, `deleteComment` backed by generated client calls; refetches after each mutation
- [ ] T078 [P] [US4] Create `frontend/src/components/ui/ConfirmDialog.tsx`: modal with "Are you sure?" message, "Cancel" and "Confirm" buttons; used for comment deletion confirmation (FR-008b)
- [ ] T079 [P] [US4] Create `frontend/src/components/tasks/CommentList.tsx`: receives `taskId: number`; calls `useComments`; renders comments ordered oldest-first (FR-009); each comment shows body, `author.name`, `created_at` timestamp, `is_edited` → "(edited)" indicator (FR-008a); "Edit" inline form replacing body on click; "Delete" button triggers `ConfirmDialog` before calling delete (FR-008b); shows `EmptyState` when no comments; validates that edit body is non-whitespace before submitting
- [ ] T080 [US4] Create `frontend/src/components/tasks/TaskDetail.tsx`: renders full task info (title, description, assignee, creator, column, created_at) per FR-007; inline edit for title and description calling `PATCH /api/v1/tasks/{id}` (FR-013); assignee dropdown for reassignment calling PATCH (FR-012); embeds `CommentList`; wire TaskCard click in KanbanBoard to open TaskDetail (modal or navigate)
- [ ] T081 [P] [US4] Write `backend/tests/test_comments.py`: `POST` creates comment; body=whitespace → 422; body > 2000 chars → 422; missing X-User-Id → 401; `PATCH` updates body and sets `is_edited=True`, preserves `author_id`; edit to whitespace → 422; `DELETE` returns 204; second `DELETE` → 404; `GET` returns comments in `created_at` asc order
- [ ] T082 [P] [US4] Write `frontend/tests/components/CommentList.test.tsx` using Vitest + RTL: renders comment with author name and timestamp; "(edited)" appears after edit; delete button triggers confirmation before removal; empty-state shown when no comments; whitespace-only body shows inline validation error
- [ ] T083 [P] [US4] Write `frontend/tests/components/TaskCard.test.tsx`: renders task title, assignee name, and creator name (FR-007a); no detail view opened by default
- [ ] T084 [P] [US4] Write `frontend/tests/components/TaskForm.test.tsx`: renders all required fields; empty title shows inline error; assignee dropdown lists all 5 users; description optional; form submits only when valid

**Checkpoint**: Open task detail → add/edit/delete comments with "(edited)" indicator and confirmation prompt. Whitespace-only body rejected inline. Task reassignment and title/description edit work from detail view (Milestones 7 and 8 of quickstart.md).

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: E2E coverage, error states, performance validation, documentation, and final integration checks.

- [ ] T085 Write `frontend/tests/e2e/board.spec.ts` Playwright test: golden path — select identity (Bob Kumar) → navigate to any project → create task "E2E Test Task" assigned to Carol Santos → verify task in To Do column with both names → move task to In Progress → verify column change → reload the page → verify task remains in In Progress column (SC-006: column move persists across page reload) → all within SC-001 timing target
- [ ] T086 [P] Add global error handling to `frontend/src/lib/apiClient.ts`: catch HTTP 422 responses and surface per-field errors to form components; catch HTTP 401 and prompt identity re-selection; catch network errors with user-friendly toast/banner message
- [ ] T087 [P] Add loading states to all data-fetching components: skeleton or spinner during `GET /api/v1/projects` and `GET /api/v1/projects/{id}/tasks`; prevents blank flash on first load
- [ ] T088 [P] Create `backend/README.md`: purpose, local setup (`uv sync`, `.env` from `.env.example`), `docker compose up`, running tests (`docker compose exec backend pytest -v`), API reference link to `contracts/api.md`
- [ ] T089 [P] Create `frontend/README.md`: purpose, local setup (`npm ci`), Vite dev server, `npm run generate-client` prerequisite (backend must be running), `npm test`, Playwright setup
- [ ] T090 Validate ruff and mypy pass with zero errors/warnings across all backend source files (`backend/app/`); fix any flagged issues
- [ ] T091 [P] Run all quickstart.md milestones end-to-end: Milestone 1 (stack baseline), Milestone 2 (identity), Milestone 3 (sample projects, SC-004 < 3 s), Milestone 4 (project creation + validation), Milestone 5 (task creation, SC-005 < 60 s), Milestone 6 (column moves + backward, SC-001 < 5 s), Milestone 7 (edit task), Milestone 8 (comment lifecycle + validation), Milestone 9 (empty state), Milestone 10 (Playwright E2E)
- [ ] T092 [P] Verify SC-002: Kanban board (4 columns with tasks) renders without horizontal scroll at exactly 1280px viewport width in Chrome and Firefox

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Requires Phase 1 complete — **blocks all user stories**
- **US1 (Phase 3)**: Requires Phase 2 complete; no dependency on US2/US3/US4
- **US2 (Phase 4)**: Requires Phase 2 complete; no dependency on US1/US3/US4 (may start in parallel with US1)
- **US3 (Phase 5)**: Requires Phase 2 complete; depends on US2 (task PATCH extends the task creation data model)
- **US4 (Phase 6)**: Requires Phase 2 complete; can start in parallel with US3
- **Polish (Phase 7)**: Requires all user story phases complete

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational. No dependency on US2/US3/US4.
- **US2 (P2)**: Can start after Foundational. No dependency on US1 (read path not required to implement write path, though both share the same DB).
- **US3 (P3)**: Depends on US2 (task creation must exist before task update can be meaningfully tested end-to-end).
- **US4 (P4)**: Can start after Foundational; builds on entities established in US1/US2 but comment routes are independent.

### Within Each User Story

- Schemas before repositories (repositories depend on model + schema types)
- Repositories before services
- Services before route handlers
- Backend endpoints before frontend hooks (frontend uses generated client)
- Frontend hooks before UI components
- Implementation before tests (pragmatic: tests verify correctness after implementation)

---

## Parallel Opportunities

### Phase 2 Parallel Block (after T011–T012 complete)

```
T013 models/user.py          T014 models/project.py       T015 models/task.py
T016 models/comment.py       T017 schemas/common.py        T018 schemas/user.py
                        → T019 alembic env.py (needs all models)
                        → T020 initial migration (needs T019)
```

### Phase 3 Parallel Block (after T035 complete)

```
T036 schemas/project.py      T037 schemas/task.py
T038 repo/project.py         T041 repo/task.py
T044 useProjects.ts          T045 useTasks.ts         T046 EmptyState.tsx
T047 TaskCard.tsx            T048 KanbanColumn.tsx
```

### Phase 6 Parallel Block (after Phase 2 complete)

```
T072 schemas/comment.py      T073 repo/comment.py
T077 useComments.ts          T078 ConfirmDialog.tsx
T082 CommentList.test.tsx    T083 TaskCard.test.tsx   T084 TaskForm.test.tsx
```

---

## Implementation Strategy

### MVP First (US1 Only — Phases 1–3)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (stack baseline proven)
3. Complete Phase 3: US1 (read-only board with sample data)
4. **STOP and VALIDATE**: Three sample projects visible, board loads, 4 columns display
5. This alone validates SC-002, SC-003, SC-004

### Incremental Delivery

1. Phase 1 + 2 → Stack proven, identity selection works
2. Phase 3 (US1) → Read-only board with sample data (MVP demo)
3. Phase 4 (US2) → Task creation; board becomes writable
4. Phase 5 (US3) → Column moves; Kanban workflow live
5. Phase 6 (US4) → Comments; team communication added
6. Phase 7 → Polish, E2E tests, all quickstart.md milestones pass

### Parallel Team Strategy

Once Phase 2 is complete:
- **Developer A**: US1 (Phase 3) — read-only board, project list, KanbanBoard components
- **Developer B**: US2 (Phase 4) — task and project creation, TaskForm, ProjectForm
- **Developer C**: US4 (Phase 6) — comment routes, CommentList (US3 unblocked once US2 task PATCH schema lands)

---

## Notes

- `[P]` tasks operate on different files with no within-phase dependencies — safe to parallelize
- `[Story]` label maps each task to its user story for traceability to spec.md
- All test tasks use the real MySQL test DB via `TEST_DATABASE_URL` (no mocks — per ADR-0003 and research.md)
- Generated `frontend/src/client/` is never hand-edited; regenerate after any backend schema change
- `X-User-Id` identity pattern is a phase 1 deviation (ADR-0001); `deps.py` isolates the resolver for phase 2 migration to JWT
- Commit after each logical task group; each checkpoint is a valid demo state
