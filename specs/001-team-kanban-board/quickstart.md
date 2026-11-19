# Quickstart Validation Guide: Team Kanban Board

**Branch**: `001-team-kanban-board` | **Date**: 2026-11-17

This guide describes how to start the application and validate that each feature
requirement works end-to-end. References to contracts and data model use relative
links; no implementation code is included here.

---

## Prerequisites

- Docker and Docker Compose installed (`docker compose version` ≥ 2.x)
- Node.js 20+ (for client generation only; not required for runtime)
- Repository cloned to local disk

---

## Start Everything

```bash
docker compose up
```

This starts three services in dependency order:

| Service | URL | Hot Reload |
|---------|-----|------------|
| MySQL 8 | `localhost:3306` | n/a |
| Backend (FastAPI) | `http://localhost:8000` | Yes (uv / uvicorn --reload) |
| Frontend (Vite) | `http://localhost:5173` | Yes (HMR) |

On first start the backend:
1. Runs Alembic migrations (`alembic upgrade head`)
2. Seeds the database (5 users + 3 sample projects + representative tasks)

Subsequent starts are idempotent — the seed function checks before inserting.

---

## Milestone 1: Stack Baseline

Validates the full chain: FastAPI → SQLAlchemy → MySQL → OpenAPI → generated client.

### 1a. Health Check

```bash
curl http://localhost:8000/health
```

Expected:
```json
{ "status": "ok", "db": "ok" }
```

### 1b. Users List

```bash
curl http://localhost:8000/api/v1/users
```

Expected: Array of exactly 5 users. Confirm names and roles match seed data
(see [data-model.md — Seed Data](data-model.md#seed-data)).

### 1c. Generated Client is Typed

```bash
cd frontend
npm run generate-client
```

Expected: `src/client/` regenerated with no TypeScript errors.
Run `npx tsc --noEmit` to confirm zero type errors.

---

## Milestone 2: Identity Selection (FR-001, FR-014)

1. Open `http://localhost:5173` in a desktop browser (1280px+ wide)
2. **Expected**: An identity-selection prompt appears listing all five team members
   with their names and roles
3. Select **Bob Kumar** (engineer)
4. **Expected**: Bob's name appears persistently in the header or sidebar
5. Navigate to any project, then switch identity to **Alice Chen**
6. **Expected**: The view remains on the current project; Alice's name replaces Bob's
   in the identity display (FR-014 — switching does not lose current view)

---

## Milestone 3: Sample Projects on First Launch (FR-010, SC-004)

1. After selecting any user identity, **without creating anything**
2. **Expected within 3 seconds**: Three sample projects are listed on the main screen
3. Click into each sample project and confirm:
   - Four Kanban columns are displayed (To Do, In Progress, In Review, Done)
   - Tasks are distributed across multiple columns (at least one column has ≥ 2 tasks)
   - Each task card shows: title, assignee name, creator name (FR-007a)

---

## Milestone 4: Create Project (FR-003)

1. Click "Create Project"
2. Submit with:
   - Name: `Validation Test Project` (28 chars)
   - Description: (optional — leave blank)
3. **Expected**: New project appears in the project list immediately
4. Validation test — submit with:
   - Name: (empty)
5. **Expected**: Inline validation error on the name field; no project created (FR-011)
6. Validation test — submit with name of 101 characters
7. **Expected**: Inline validation error; no project created

---

## Milestone 5: Create and Assign Task (FR-005, SC-005)

Start timer when clicking "Create Task":

1. Navigate to `Validation Test Project`
2. Click "Create Task"
3. Fill in:
   - Title: `First task`
   - Description: `Testing task creation`
   - Assignee: **Carol Santos**
4. Submit
5. **Expected within 60 seconds**: Task appears in the **To Do** column showing
   title, Carol's name as assignee, and current user as creator

**Validation edge cases**:
- Submit with empty title → inline error, task not created
- Submit with 101-character title → inline error, task not created
- Submit without selecting assignee → inline error, task not created

---

## Milestone 6: Move Task Across Columns (FR-006, SC-001)

Start timer when the board is open:

1. Find `First task` in the **To Do** column
2. Move it to **In Progress** (via drag-and-drop or column-change action)
3. **Expected within 5 seconds**: Task disappears from To Do and appears in In Progress
4. Move it directly from **In Progress** to **Done** (backward-skip test per clarification Q1)
5. **Expected**: Task appears in Done — no error, no restriction message
6. Move it from **Done** back to **To Do** (backward movement test)
7. **Expected**: Task returns to To Do — backward movement is unrestricted

---

## Milestone 7: Edit Task (FR-012, FR-013)

1. Click on `First task` to open the detail view
2. Edit title to `First task (edited)`; save
3. **Expected**: Title updates on the board card and in the detail view
4. Reassign from **Carol Santos** to **David Park**
5. **Expected**: Assignee name updates on the board card immediately

---

## Milestone 8: Comment Lifecycle (FR-008, FR-008a, FR-008b)

1. With `First task` detail open, as **Alice Chen**:
2. Add comment: `Looking good, moving forward.`
3. **Expected**: Comment appears attributed to Alice with a timestamp; no "(edited)" indicator

**Edit comment** (FR-008a):
4. Edit the comment body to: `Looking good, moving forward. ✓`
5. **Expected**: Comment body updates; "(edited)" indicator appears; Alice's name and
   original timestamp are unchanged

**Delete confirmation** (FR-008b edge case):
6. Click delete on the comment
7. **Expected**: A confirmation dialog/prompt appears before deletion
8. Confirm deletion
9. **Expected**: Comment permanently removed; empty-state message shown if no comments remain

**Validation edge cases**:
- Submit empty comment → inline validation error; comment not created
- Submit whitespace-only comment → inline validation error
- Edit comment to whitespace-only → inline validation error; original body preserved
- Submit comment of 2001 characters → inline validation error (max 2000)

---

## Milestone 9: Empty State (Edge Case)

1. Create a new project with no tasks
2. **Expected**: All four Kanban columns are displayed; each shows an empty-state message
   (not a blank area)

---

## Milestone 10: End-to-End Playwright Validation

```bash
cd frontend
npx playwright test tests/e2e/board.spec.ts
```

The E2E test covers the golden path: user identity selection → project navigation →
task creation → column move → verify updated column.

Expected: All assertions pass; test completes without timeout.

---

## Backend Tests

```bash
docker compose exec backend pytest -v
```

Expected: All tests pass. Key coverage areas:
- `test_health.py` — health endpoint returns `{ status: ok, db: ok }`
- `test_users.py` — GET /users returns exactly 5 seeded users
- `test_projects.py` — CRUD operations and validation rejection cases
- `test_tasks.py` — creation, column moves (including backward), reassignment, validation
- `test_comments.py` — create, edit (is_edited flag), delete, validation

---

## Success Criteria Verification Map

| Criterion | Validated by milestone |
|-----------|----------------------|
| SC-001: Column move in under 5 s | Milestone 6 (timed) |
| SC-002: Board fits 1280px viewport without horizontal scroll | Milestone 3 (visual) |
| SC-003: All 5 users selectable, identity reflected | Milestone 2 |
| SC-004: Sample projects visible in under 3 s | Milestone 3 (timed) |
| SC-005: Task created in under 60 s | Milestone 5 (timed) |
| SC-006: Content persists for session duration | Milestones 5–9 (reload browser, check persistence) |
| SC-007: Validation errors within 1 s | Milestones 4, 5, 8 (validation edge cases) |
| SC-008: 100% inputs validated | Milestones 4, 5, 8 (all rejection cases tested) |
