# Feature Specification: Team Kanban Board

**Feature Branch**: `001-team-kanban-board`

**Created**: 2026-11-17

**Status**: Draft

**Input**: User description: "Develop Taskify, a team productivity platform where predefined users create projects, assign tasks, comment, and move tasks across Kanban columns (To Do, In Progress, In Review, Done). Five users (one product manager, four engineers), three sample projects, no login for this first phase."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View Projects and Kanban Board (Priority: P1)

A team member opens Taskify, selects their identity from the predefined user list, and
immediately sees all available projects. They navigate into a project and view the Kanban
board showing tasks organized across the four columns: To Do, In Progress, In Review,
and Done.

**Why this priority**: This is the foundational view the entire platform is built around.
Without a working board, no other story delivers value. It also validates the predefined
user model and the three sample projects requirement.

**Independent Test**: Can be fully tested by launching the app, selecting any user, navigating
to any of the three sample projects, and confirming tasks appear in their correct columns —
delivers immediate visibility value with no other features needed.

**Acceptance Scenarios**:

1. **Given** the app is loaded for the first time, **When** a user opens it, **Then** a
   user-selection prompt displays all five predefined team members (1 product manager,
   4 engineers) with their names and roles visible.
2. **Given** a user has been selected, **When** they view the main screen, **Then** three
   sample projects are listed with their names visible.
3. **Given** a user selects a project, **When** the board opens, **Then** four columns
   (To Do, In Progress, In Review, Done) are displayed left-to-right, each containing
   the tasks assigned to that column.
4. **Given** a project has tasks in multiple columns, **When** the board loads, **Then**
   each task appears in exactly one column matching its current status.

---

### User Story 2 - Create and Assign Tasks (Priority: P2)

A product manager or engineer creates a new task within a project, sets a title and
description, and assigns it to one of the five team members. The task immediately
appears in the "To Do" column of the board.

**Why this priority**: Task creation is the primary write action on the platform. All
downstream actions (moving, commenting) depend on tasks existing.

**Independent Test**: Can be fully tested by creating a task with all required fields,
assigning it to a team member, and confirming it appears in the "To Do" column attributed
to the correct assignee — delivers core task-tracking value independently.

**Acceptance Scenarios**:

1. **Given** a user is viewing a project board, **When** they open the new-task form and
   submit a valid title, description, and assignee, **Then** the task appears in the
   "To Do" column immediately.
2. **Given** a user submits a new task with a missing title, **When** they attempt to
   save, **Then** a validation error is displayed inline and the task is not created.
3. **Given** a task has been created, **When** any user views the board, **Then** the task
   displays the assignee's name and the creating user's identity.
4. **Given** a user submits a task with a title exceeding the maximum allowed length,
   **When** they attempt to save, **Then** a validation error is shown and the task is not
   created.

---

### User Story 3 - Move Tasks Across Kanban Columns (Priority: P3)

A team member drags a task card or uses a column-change action to move a task from its
current column to another (e.g., from "In Progress" to "In Review"). The board updates
immediately to reflect the new position.

**Why this priority**: Column movement is the core Kanban workflow action. Without it, the
board is read-only and loses its primary value as a workflow management tool.

**Independent Test**: Can be fully tested by moving an existing task between columns and
confirming the board reflects the change — delivers the core Kanban workflow independently.

**Acceptance Scenarios**:

1. **Given** a task is in the "To Do" column, **When** a user moves it to "In Progress",
   **Then** the task disappears from "To Do" and appears in "In Progress".
2. **Given** a task is moved to a column, **When** the board refreshes or another user views
   it, **Then** the task remains in the new column.
3. **Given** a user attempts to move a task to its current column, **When** the action
   completes, **Then** no visible change occurs and no error is shown.
4. **Given** a task is in any column, **When** a user moves it, **Then** it can be placed
   into any of the other three columns (not restricted to adjacent columns only).

---

### User Story 4 - Comment on Tasks (Priority: P4)

A team member opens a task's detail view and adds a text comment. The comment is
attributed to the currently selected user and appears in chronological order below
any existing comments.

**Why this priority**: Comments enable team communication within tasks. They are valuable
but the platform is functional for task tracking without them, making this lower priority
than creation and movement.

**Independent Test**: Can be fully tested by opening a task, adding a comment, and
confirming it appears with the correct author attribution and timestamp — delivers
team communication value independently.

**Acceptance Scenarios**:

1. **Given** a user views a task's detail, **When** they submit a non-empty comment,
   **Then** the comment appears at the bottom of the comment list attributed to the
   currently selected user with a timestamp.
2. **Given** multiple comments exist on a task, **When** a user views the task detail,
   **Then** comments are displayed in chronological order (oldest first).
3. **Given** a user submits a comment containing only whitespace, **When** they attempt
   to save, **Then** a validation error is displayed and the comment is not created.
4. **Given** a task has no comments, **When** a user opens the task detail, **Then** an
   appropriate empty-state message is shown instead of a blank comments area.

---

### Edge Cases

- What happens when a project has no tasks? (All four columns are displayed with an
  empty-state message indicating no tasks exist yet.)
- What happens when a user submits a comment with only whitespace? (Validation rejects
  it with an inline error; the comment is not saved.)
- What happens when a task title exceeds the maximum allowed length? (Validation shows
  an inline error; the task is not created.)
- What happens when no user identity has been selected and they attempt to create a
  task or comment? (The user is prompted to select their identity before proceeding.)
- What happens when a task is moved to its current column? (No change occurs; no error
  is surfaced to the user.)
- What happens when two users attempt to rename the same task at the same time? (The
  last write wins; no conflict resolution is required in this phase.)
- What happens when a user deletes a comment? (The comment is permanently removed with
  no undo; a brief confirmation prompt MUST be shown before deletion to prevent accidental
  loss.)
- What happens when a user edits a comment to be empty or whitespace-only? (Validation
  rejects the edit; the original comment body is preserved.)

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST present a predefined list of exactly five team members (one
  product manager and four engineers) for identity selection when the app is first opened,
  with no login or registration required.
- **FR-002**: System MUST display all existing projects as a navigable list on the main
  screen after a user selects their identity.
- **FR-003**: System MUST allow any user to create a new project by providing a name
  (required) and an optional description.
- **FR-004**: System MUST display a Kanban board for each project with exactly four
  columns in this fixed order: To Do, In Progress, In Review, Done.
- **FR-005**: System MUST allow any user to create a task within a project by specifying
  a title (required), a description (optional), and an assignee chosen from the five
  predefined team members.
- **FR-006**: System MUST allow any user to move any task from its current column to any
  other column without restriction, including backward movement (e.g., Done → In Progress).
  No transition rules or mandatory reasons are required for any direction of movement.
- **FR-007**: System MUST display a task detail view showing the task's title, description,
  assignee, creator, current column, creation timestamp, and all associated comments.
- **FR-007a**: Each task card on the Kanban board MUST display, without requiring the
  detail view to be opened: the task title, the assignee's name, and the creator's name.
- **FR-008**: System MUST allow any user to add a text comment to a task; the comment
  MUST be attributed to the currently selected user identity and timestamped.
- **FR-008a**: System MUST allow any user to edit the body text of any comment; the
  original author attribution and timestamp MUST be preserved, and an "edited" indicator
  MUST be displayed on the comment after modification.
- **FR-008b**: System MUST allow any user to delete any comment; deleted comments MUST
  be permanently removed with no recovery mechanism.
- **FR-009**: System MUST display comments on a task in ascending chronological order
  (oldest first).
- **FR-010**: System MUST pre-seed three sample projects with representative tasks
  distributed across columns on initial load, requiring no user setup.
- **FR-011**: All user-provided inputs MUST be validated for required fields,
  non-whitespace content where required, and the following maximum character lengths
  before being accepted; validation errors MUST be shown inline adjacent to the
  offending field:
  - Project name: required, max 100 characters.
  - Task title: required, max 100 characters.
  - Task description: optional, max 2000 characters.
  - Comment body: required (non-whitespace), max 2000 characters.
- **FR-012**: System MUST allow any user to reassign any existing task to any other
  team member from the predefined list, regardless of who created the task or who is
  currently assigned. No role restriction applies in this phase.
- **FR-013**: System MUST allow any user to edit an existing task's title and description.
- **FR-014**: The currently selected user identity MUST be persistently visible (e.g.,
  in a header or sidebar) and switchable at any time without losing the current view.

### Key Entities *(include if feature involves data)*

- **User**: A predefined team member with a name and role (product manager or engineer).
  There are exactly five users; they cannot be created or deleted by end users.
- **Project**: A named container for a set of tasks. Attributes: name (required,
  max 100 chars), optional description, creator (User), creation timestamp.
- **Task**: A unit of work belonging to a project. Attributes: title (required,
  max 100 chars), description (optional, max 2000 chars), assignee (User), creator
  (User), Kanban column (one of the four fixed values), creation timestamp.
- **Comment**: A text note attached to a task. Attributes: body text (required,
  max 2000 chars), author (User), created timestamp, edited flag (boolean). Any user
  may edit or delete any comment; original author attribution is preserved after edits.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Any user can move a task from one Kanban column to another in under 5
  seconds from opening the board.
- **SC-002**: The Kanban board for any project is fully visible and operable without
  horizontal scrolling on a 1280px-wide desktop viewport.
- **SC-003**: All five predefined team members are selectable at session start; their
  identity is correctly reflected on every task and comment they create.
- **SC-004**: Three sample projects with tasks distributed across columns are visible
  within 3 seconds of first launch with zero setup steps by the user.
- **SC-005**: A new task can be created, assigned, and confirmed on the board in under
  60 seconds from clicking the "create task" action.
- **SC-006**: All user-created content (projects, tasks, comments, column moves) persists
  and remains visible for the full duration of the browser session without data loss.
- **SC-007**: Input validation errors are displayed inline adjacent to the offending field
  within 1 second of a failed submission attempt.
- **SC-008**: 100% of user inputs (task titles, descriptions, comments, project names)
  are validated before acceptance; no unvalidated data is stored or displayed.

## Clarifications

### Session 2026-11-17

- Q: Should users be able to move a task to any column in any direction (e.g., from "Done" back to "In Progress"), or should column transitions be restricted to forward-only movement? → A: Any column to any column, any direction, no restrictions.
- Q: Should users be able to edit or delete comments after they are posted, and if so, is that permission restricted to the comment's own author or open to any user? → A: Any user can edit and delete any comment.
- Q: Should any user be able to reassign a task to a different team member, or should reassignment be restricted to specific actors such as the task creator or the product manager? → A: Any user can reassign any task.
- Q: What are the maximum character limits for task title, task description, comment body, and project name? → A: Task title 100, task description 2000, comment body 2000, project name 100.
- Q: What information should be visible on a task card directly on the Kanban board, without requiring the user to open the task detail view? → A: Title + assignee name + creator name.

## Assumptions

- No user authentication or registration is required in this phase; users select their
  identity from the predefined list at session start. Identity switching is permitted at
  any time.
- Data persistence is scoped to the browser session (e.g., in-memory or local storage);
  no server-side database or backend API is required for this phase.
- All five users share unrestricted access to all projects, tasks, and comments; no
  per-project or per-role permissions are enforced in this phase.
- The product manager and engineers have identical feature capabilities; role distinctions
  are for labeling and identity purposes only in this phase.
- The three sample projects are pre-seeded at first load and cannot be deleted by users
  in this phase.
- Mobile and tablet support is out of scope for this phase; the platform targets
  desktop browsers only.
- Task ordering within a column is by creation date (ascending) unless a user explicitly
  reorders; drag-to-reorder within a column is out of scope for this phase.
- There is no real-time multi-user synchronization in this phase; each browser session
  operates independently.
