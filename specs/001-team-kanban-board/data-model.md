# Data Model: Team Kanban Board

**Branch**: `001-team-kanban-board` | **Date**: 2026-11-17

---

## Entity Overview

```
User (seed-only)
 ├── creates → Project (many)
 ├── creates → Task (many, as creator)
 ├── assigned → Task (many, as assignee)
 └── authors → Comment (many)

Project
 └── contains → Task (many, cascade delete)

Task
 └── has → Comment (many, cascade delete)
```

---

## Entity: User

**Table**: `users`
**Lifecycle**: Seeded at startup. No create/update/delete via API.

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `INT` | PK, AUTO_INCREMENT | |
| `name` | `VARCHAR(100)` | NOT NULL | Full display name |
| `role` | `ENUM('product_manager','engineer')` | NOT NULL | Phase 1: display only, no behavioral distinction |

**Seed data** (exactly 5 rows, inserted idempotently on startup):

| id | name | role |
|----|------|------|
| 1 | Alice Chen | product_manager |
| 2 | Bob Kumar | engineer |
| 3 | Carol Santos | engineer |
| 4 | David Park | engineer |
| 5 | Eva Müller | engineer |

**Validation rules**:
- `name`: non-empty, max 100 chars (read-only after seeding)
- `role`: one of the two enum values

---

## Entity: Project

**Table**: `projects`
**Lifecycle**: Created by any user; cannot be deleted in phase 1.

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `INT` | PK, AUTO_INCREMENT | |
| `name` | `VARCHAR(100)` | NOT NULL | Required; max 100 chars |
| `description` | `TEXT` | NULL | Optional; max 2000 chars enforced in schema |
| `creator_id` | `INT` | NOT NULL, FK → users.id | Set from X-User-Id header |
| `created_at` | `DATETIME` | NOT NULL, DEFAULT CURRENT_TIMESTAMP | UTC |

**Validation rules**:
- `name`: required, non-whitespace, max 100 chars
- `description`: optional, max 2000 chars if present
- `creator_id`: must be a valid seeded user ID (resolved from X-User-Id header)

**Relationships**:
- `creator_id` → `users.id` (RESTRICT on delete; users cannot be deleted)
- One project has many tasks (CASCADE DELETE on tasks when project is deleted —
  not exposed in phase 1 but designed for future)

---

## Entity: Task

**Table**: `tasks`
**Lifecycle**: Created by any user; can be edited, reassigned, and moved by any user.

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `INT` | PK, AUTO_INCREMENT | |
| `project_id` | `INT` | NOT NULL, FK → projects.id | Immutable after creation |
| `title` | `VARCHAR(100)` | NOT NULL | Required; max 100 chars |
| `description` | `TEXT` | NULL | Optional; max 2000 chars in schema |
| `column` | `ENUM('todo','in_progress','in_review','done')` | NOT NULL, DEFAULT 'todo' | Kanban status |
| `assignee_id` | `INT` | NOT NULL, FK → users.id | Required at creation; updateable |
| `creator_id` | `INT` | NOT NULL, FK → users.id | Set from X-User-Id; immutable |
| `created_at` | `DATETIME` | NOT NULL, DEFAULT CURRENT_TIMESTAMP | UTC |

**Validation rules**:
- `title`: required, non-whitespace, max 100 chars
- `description`: optional, max 2000 chars
- `column`: must be one of the four enum values; any direction of movement is valid
  (FR-006 — no transition restrictions)
- `assignee_id`: required; must be a valid seeded user ID (1–5)
- `creator_id`: set from `X-User-Id` header; not accepted from request body

**State transitions** (column field):
- All transitions are valid in any direction: `todo ↔ in_progress ↔ in_review ↔ done`
- No business rules gate transitions; the column field is updated directly

**Relationships**:
- `project_id` → `projects.id` (CASCADE DELETE)
- `assignee_id` → `users.id` (RESTRICT)
- `creator_id` → `users.id` (RESTRICT)
- One task has many comments (CASCADE DELETE)

**Updatable fields** (via PATCH):
- `title` (FR-013)
- `description` (FR-013)
- `column` (FR-006)
- `assignee_id` (FR-012)

---

## Entity: Comment

**Table**: `comments`
**Lifecycle**: Created, edited, and deleted by any user (FR-008, FR-008a, FR-008b).

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `INT` | PK, AUTO_INCREMENT | |
| `task_id` | `INT` | NOT NULL, FK → tasks.id | Immutable after creation |
| `body` | `TEXT` | NOT NULL | Required; max 2000 chars |
| `author_id` | `INT` | NOT NULL, FK → users.id | Set from X-User-Id; immutable |
| `is_edited` | `BOOLEAN` | NOT NULL, DEFAULT FALSE | Set to TRUE on first edit |
| `created_at` | `DATETIME` | NOT NULL, DEFAULT CURRENT_TIMESTAMP | UTC |

**Validation rules**:
- `body`: required, non-whitespace, max 2000 chars; applies to both create and edit
- `author_id`: set from `X-User-Id` header on creation; not updatable
- Editing a comment to empty or whitespace-only body MUST be rejected (validation error)

**Relationships**:
- `task_id` → `tasks.id` (CASCADE DELETE)
- `author_id` → `users.id` (RESTRICT)

**Updatable fields** (via PATCH):
- `body` — sets `is_edited = True`

**Deletion**: permanent, no soft-delete; cascades if parent task is deleted.

---

## Pydantic v2 Schema Summary

### Request Schemas

| Schema | Used by | Key fields |
|--------|---------|------------|
| `ProjectCreate` | POST /projects | `name` (str, max 100), `description` (str \| None, max 2000) |
| `TaskCreate` | POST /projects/{id}/tasks | `title` (str, max 100), `description` (str \| None, max 2000), `assignee_id` (int) |
| `TaskUpdate` | PATCH /tasks/{id} | `title` (str \| None), `description` (str \| None), `column` (KanbanColumn \| None), `assignee_id` (int \| None) — all optional, min one required |
| `CommentCreate` | POST /tasks/{id}/comments | `body` (str, min 1 non-whitespace, max 2000) |
| `CommentUpdate` | PATCH /comments/{id} | `body` (str, min 1 non-whitespace, max 2000) |

### Response Schemas

| Schema | Fields |
|--------|--------|
| `UserResponse` | `id`, `name`, `role` |
| `ProjectResponse` | `id`, `name`, `description`, `creator` (UserResponse), `created_at` |
| `TaskResponse` | `id`, `project_id`, `title`, `description`, `column`, `assignee` (UserResponse), `creator` (UserResponse), `created_at` |
| `TaskDetailResponse` | extends TaskResponse + `comments` (list[CommentResponse]) |
| `CommentResponse` | `id`, `task_id`, `body`, `author` (UserResponse), `is_edited`, `created_at` |
| `HealthResponse` | `status` (str: "ok"), `db` (str: "ok" \| "error") |

### Shared Types

```python
# schemas/common.py
class KanbanColumn(str, Enum):
    TODO = "todo"
    IN_PROGRESS = "in_progress"
    IN_REVIEW = "in_review"
    DONE = "done"

COLUMN_DISPLAY_LABELS = {
    KanbanColumn.TODO: "To Do",
    KanbanColumn.IN_PROGRESS: "In Progress",
    KanbanColumn.IN_REVIEW: "In Review",
    KanbanColumn.DONE: "Done",
}
```

---

## Database Indexes

| Table | Index | Reason |
|-------|-------|--------|
| `tasks` | `(project_id, column)` | Board query: fetch all tasks for a project, group by column |
| `tasks` | `(assignee_id)` | Future: filter by assignee |
| `comments` | `(task_id, created_at)` | Task detail: fetch comments in chronological order |

---

## Alembic Migration Strategy

- **Initial migration** (`0001_initial_schema.py`): creates all four tables and indexes
- Autogenerate from `Base.metadata` with all models imported in `alembic/env.py`
- Future schema changes: each change gets a new numbered migration
- Test database: same migration applied via `alembic upgrade head` in `conftest.py`
  before each test session
