# API Contracts: Team Kanban Board

**Branch**: `001-team-kanban-board` | **Date**: 2026-11-17
**Base URL**: `http://localhost:8000`
**OpenAPI schema**: `GET /openapi.json`

---

## Conventions

### Authentication (Phase 1)

Write operations (`POST`, `PATCH`, `DELETE`) require the `X-User-Id` header containing
the integer ID of one of the five predefined users. Read operations (`GET`) do not
require the header.

```
X-User-Id: 1
```

Responses when header is missing or invalid on a write endpoint:
```json
HTTP 401 Unauthorized
{ "detail": "Valid X-User-Id header required" }
```

### Common Error Responses

| Status | Meaning |
|--------|---------|
| 400 | Malformed request |
| 401 | Missing or invalid X-User-Id on write endpoint |
| 404 | Resource not found |
| 422 | Validation failure — body includes per-field errors |

**422 body structure**:
```json
{
  "detail": [
    { "loc": ["body", "title"], "msg": "String should have at most 100 characters", "type": "string_too_long" }
  ]
}
```

### Timestamps

All `created_at` values are ISO 8601 UTC strings: `"2026-11-17T10:30:00Z"`

---

## Endpoints

### Health

#### `GET /health`

Returns the operational status of the backend and its database connection.
No authentication required.

**Response 200**:
```json
{
  "status": "ok",
  "db": "ok"
}
```

If the DB is unreachable, `"db"` returns `"error"` and status is still `200`
(the endpoint itself is healthy; the DB state is reported, not thrown).

---

### Users

#### `GET /api/v1/users`

Returns all five predefined team members. No authentication required.
Used by the frontend to populate user-selector and assignee dropdowns.

**Response 200**:
```json
[
  { "id": 1, "name": "Alice Chen", "role": "product_manager" },
  { "id": 2, "name": "Bob Kumar", "role": "engineer" },
  { "id": 3, "name": "Carol Santos", "role": "engineer" },
  { "id": 4, "name": "David Park", "role": "engineer" },
  { "id": 5, "name": "Eva Müller", "role": "engineer" }
]
```

---

### Projects

#### `GET /api/v1/projects`

Returns all projects ordered by `created_at` descending. No authentication required.

**Response 200**:
```json
[
  {
    "id": 1,
    "name": "Mobile App Redesign",
    "description": "Redesign the core mobile flows.",
    "creator": { "id": 1, "name": "Alice Chen", "role": "product_manager" },
    "created_at": "2026-11-17T09:00:00Z"
  }
]
```

---

#### `POST /api/v1/projects`

Creates a new project. Requires `X-User-Id` header.

**Request body**:
```json
{
  "name": "New Project",           // required, max 100 chars
  "description": "Optional text."  // optional, max 2000 chars
}
```

**Response 201**:
```json
{
  "id": 4,
  "name": "New Project",
  "description": "Optional text.",
  "creator": { "id": 1, "name": "Alice Chen", "role": "product_manager" },
  "created_at": "2026-11-17T10:00:00Z"
}
```

---

#### `GET /api/v1/projects/{project_id}`

Returns a single project. No authentication required.

**Response 200**: Same shape as one element from `GET /api/v1/projects`.
**Response 404** if project not found.

---

### Tasks

#### `GET /api/v1/projects/{project_id}/tasks`

Returns all tasks for a project ordered by `created_at` ascending.
No authentication required. Frontend groups tasks by `column` client-side.

**Response 200**:
```json
[
  {
    "id": 1,
    "project_id": 1,
    "title": "Design login screen",
    "description": null,
    "column": "in_progress",
    "assignee": { "id": 2, "name": "Bob Kumar", "role": "engineer" },
    "creator": { "id": 1, "name": "Alice Chen", "role": "product_manager" },
    "created_at": "2026-11-17T09:05:00Z"
  }
]
```

**Response 404** if project not found.

---

#### `POST /api/v1/projects/{project_id}/tasks`

Creates a task in the given project. New tasks start in column `todo`. Requires `X-User-Id`.

**Request body**:
```json
{
  "title": "Implement auth flow",   // required, max 100 chars
  "description": "Details here.",   // optional, max 2000 chars
  "assignee_id": 3                  // required; must be 1–5
}
```

**Response 201**: Full `TaskResponse` with `column: "todo"` and embedded
`assignee` and `creator` objects.
**Response 404** if project not found.
**Response 422** if validation fails (title missing, assignee_id invalid, etc.).

---

#### `GET /api/v1/tasks/{task_id}`

Returns full task detail including all comments. No authentication required.

**Response 200**:
```json
{
  "id": 1,
  "project_id": 1,
  "title": "Design login screen",
  "description": "Figma link: ...",
  "column": "in_progress",
  "assignee": { "id": 2, "name": "Bob Kumar", "role": "engineer" },
  "creator": { "id": 1, "name": "Alice Chen", "role": "product_manager" },
  "created_at": "2026-11-17T09:05:00Z",
  "comments": [
    {
      "id": 1,
      "task_id": 1,
      "body": "Figma draft shared in Slack.",
      "author": { "id": 2, "name": "Bob Kumar", "role": "engineer" },
      "is_edited": false,
      "created_at": "2026-11-17T10:00:00Z"
    }
  ]
}
```

**Response 404** if task not found.

---

#### `PATCH /api/v1/tasks/{task_id}`

Partially updates a task. At least one field must be provided. Requires `X-User-Id`.
All fields are optional; omitted fields are unchanged.

**Request body** (all fields optional, min one required):
```json
{
  "title": "Updated title",        // max 100 chars
  "description": "New details.",   // max 2000 chars; send null to clear
  "column": "in_review",           // one of: todo, in_progress, in_review, done
  "assignee_id": 4                 // must be 1–5
}
```

**Response 200**: Updated `TaskResponse` (without comments; use GET for full detail).
**Response 404** if task not found.
**Response 422** if validation fails.

---

### Comments

#### `GET /api/v1/tasks/{task_id}/comments`

Returns all comments on a task ordered by `created_at` ascending. No authentication required.

**Response 200**:
```json
[
  {
    "id": 1,
    "task_id": 1,
    "body": "LGTM, moving to review.",
    "author": { "id": 3, "name": "Carol Santos", "role": "engineer" },
    "is_edited": false,
    "created_at": "2026-11-17T11:00:00Z"
  }
]
```

**Response 404** if task not found.

---

#### `POST /api/v1/tasks/{task_id}/comments`

Adds a comment to a task. Requires `X-User-Id`.

**Request body**:
```json
{
  "body": "This needs another look."  // required, non-whitespace, max 2000 chars
}
```

**Response 201**: Full `CommentResponse` with embedded `author`.
**Response 404** if task not found.
**Response 422** if `body` is empty, whitespace-only, or exceeds 2000 chars.

---

#### `PATCH /api/v1/comments/{comment_id}`

Edits the body of any comment. Any user may edit any comment (FR-008a).
Sets `is_edited = true`. Original `author` and `created_at` are unchanged.
Requires `X-User-Id`.

**Request body**:
```json
{
  "body": "Corrected comment text."  // required, non-whitespace, max 2000 chars
}
```

**Response 200**: Updated `CommentResponse` with `"is_edited": true`.
**Response 404** if comment not found.
**Response 422** if `body` is empty, whitespace-only, or exceeds 2000 chars.

---

#### `DELETE /api/v1/comments/{comment_id}`

Permanently deletes a comment. Any user may delete any comment (FR-008b).
No recovery mechanism. Requires `X-User-Id`.

**Response 204 No Content** on success.
**Response 404** if comment not found.

---

## Frontend Client Generation

The frontend generates a fully typed API client from this schema:

```bash
# Run with backend running on port 8000
npm run generate-client
```

This calls:
```
openapi-ts --input http://localhost:8000/openapi.json \
           --output src/client \
           --client @hey-api/client-fetch
```

Generated files in `src/client/` are never hand-edited. All imports go through
`src/lib/apiClient.ts` which configures the base URL and injects the `X-User-Id`
header from the user store.

---

## OpenAPI Tags (for client organization)

| Tag | Endpoints |
|-----|-----------|
| `health` | GET /health |
| `users` | GET /api/v1/users |
| `projects` | GET/POST /api/v1/projects, GET /api/v1/projects/{id} |
| `tasks` | GET/POST /api/v1/projects/{id}/tasks, GET/PATCH /api/v1/tasks/{id} |
| `comments` | GET/POST /api/v1/tasks/{id}/comments, PATCH/DELETE /api/v1/comments/{id} |
