---
name: "speckit-hld"
description: "Generate a High-Level Design (HLD) document for the active feature from spec, plan, data-model, and contracts artifacts."
argument-hint: "Optional focus areas, e.g. 'emphasise security architecture' or 'include sequence diagrams for all user flows'"
compatibility: "Requires spec-kit project structure with .specify/ directory; run after /speckit-plan"
metadata:
  author: "taskify-project"
  source: ".claude/skills/speckit-hld/SKILL.md"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Use it to weight
which sections or diagrams to expand. If the user asks to emphasise a specific area
(e.g., security, data flow, deployment), give that section more depth.

## Pre-Execution Checks

**Check for extension hooks (before HLD generation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_hld` key.
- If the YAML cannot be parsed or is invalid, do not skip silently: tell the user that
  `.specify/extensions.yml` could not be read (include the parser error) and that no hooks
  were checked, including any mandatory (`optional: false`) hooks registered there,
  then continue normally.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled`
  field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition`
  expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable.
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation
    to the HookExecutor implementation.
- When constructing command invocations from hook command names, replace dots (`.`) with
  hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to
    finish before continuing.
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently.

## Outline

### Step 1 — Locate Active Feature

Run `.specify/scripts/bash/check-prerequisites.sh --json --paths-only` from the repo root
and parse the JSON payload. Required fields:

- `REPO_ROOT` — absolute path to the repository root
- `FEATURE_DIR` — absolute path to the feature directory
- `FEATURE_SPEC` — absolute path to `spec.md`

Derive:
- `IMPL_PLAN` = `FEATURE_DIR/plan.md`
- `DATA_MODEL` = `FEATURE_DIR/data-model.md`
- `RESEARCH` = `FEATURE_DIR/research.md`
- `CONTRACTS_DIR` = `FEATURE_DIR/contracts/`
- `DOCS_DIR` = `REPO_ROOT/docs`
- `HLD_FILE` = `REPO_ROOT/docs/hld.md`

Create `DOCS_DIR` if it does not exist (`mkdir -p`).

If JSON parsing fails or `FEATURE_DIR` is missing, stop and instruct the user to run
`/speckit-specify` first.

### Step 2 — Load Input Artifacts

Read each of the following files if they exist. Note which are absent; absent optional
artifacts leave the corresponding HLD section shallower but do not block execution.

| File | Required | Used for |
|------|----------|----------|
| `spec.md` | YES | Functional requirements, entities, success criteria |
| `plan.md` | YES | Technical context, constitution check, project structure |
| `data-model.md` | Recommended | Entity definitions, relationships, schema constraints |
| `research.md` | Recommended | Integration decisions, rationale, alternatives |
| `contracts/` (all files) | Recommended | API endpoint shapes, auth conventions |
| `.specify/memory/constitution.md` | YES | Governance principles for alignment section |

If `spec.md` or `plan.md` is missing, stop with:
> ERROR: `spec.md` and `plan.md` must exist before generating an HLD.
> Run `/speckit-specify` then `/speckit-plan` first.

### Step 3 — Build the HLD

Generate `HLD_FILE` with the structure below. Every section is mandatory unless marked
*(omit if not applicable)*. Replace all placeholder text with concrete content derived
from the loaded artifacts. Do not leave section headings empty.

Use **Mermaid** syntax for all diagrams. Mermaid blocks render in GitHub, GitLab, and
most modern markdown viewers. Use `graph TD` for component/flow diagrams, `sequenceDiagram`
for interaction flows, and `erDiagram` for entity relationships.

---

#### Required HLD Sections

**1. Overview**
- One paragraph: what this feature is, who uses it, and what problem it solves.
- Reference the spec's primary user stories and success criteria (SC-001 … SC-N).
- State any explicitly deferred concerns (e.g., authentication in phase 1).

**2. System Architecture**
- Mermaid `graph TD` diagram showing all runtime components and their connections.
  Include: user browser, frontend service, backend service, database, any external
  services. Label arrows with protocol/direction (e.g., `HTTP/JSON`, `SQL`, `HMR`).
- Component table: one row per box in the diagram with its responsibility and
  technology.

**3. Component Descriptions**
One subsection per major component (e.g., Frontend, Backend API, Database):
- Purpose and boundaries.
- Internal structure (key modules/layers from `plan.md` project structure).
- What it owns; what it does NOT own.
- Key design decisions (from `research.md` if available).

**4. Data Flow — Key User Journeys**
For each P1 and P2 user story in the spec, produce a Mermaid `sequenceDiagram`
showing the actors, components, and messages involved. Label each diagram with the
user story title and priority.

Minimum diagrams (derive from spec user stories):
- P1 flow (highest priority story)
- P2 flow (second priority story)
- If more user stories exist, add diagrams for P3+ or combine minor flows.

**5. Data Model Summary**
- Mermaid `erDiagram` covering all entities, their key attributes, and relationships.
  Derive from `data-model.md` if present; otherwise derive from spec Key Entities.
- Note: this is a summary — refer to `data-model.md` for full column definitions,
  constraints, and indexes.

**6. API Surface** *(omit if no external API)*
- Summary table of all endpoints grouped by resource (derive from `contracts/`).
- Authentication / identity mechanism (from `research.md` or `plan.md`).
- Note: full request/response shapes live in `contracts/`; do not duplicate them here.

**7. Deployment Architecture**
- Mermaid `graph TD` diagram of the deployment topology: hosts, containers/services,
  ports, volumes, networks. Derive from Docker Compose configuration described in
  `plan.md` or `research.md`.
- Startup sequence: describe the order services must start and any health-check
  dependencies.
- Note any environment-specific differences (local dev vs. production intent).

**8. Cross-Cutting Concerns**
One subsection each for:
- **Security**: how each constitution Security-First requirement is addressed
  (or explicitly deferred with justification). Reference `plan.md` Constitution Check.
- **Input Validation**: how the Input Validation principle is enforced end-to-end
  (client → transport → server → DB).
- **Observability**: logging strategy, health endpoints, any metrics or tracing planned.
- **Error Handling**: how errors propagate from DB → service → API → frontend → user.

**9. Constitution Alignment**
Table with one row per constitution principle. Columns: Principle name, Status
(PASS / JUSTIFIED DEVIATION / DEFERRED / N/A), and a one-line evidence or rationale.
Derive from `plan.md` Constitution Check; do not contradict it.

**10. Constraints and Known Tradeoffs**
Bulleted list of explicit constraints (from spec Assumptions and plan.md Technical
Context) and design tradeoffs made. For each tradeoff: what was chosen, what was
rejected, and why.

**11. Open Questions and Future Considerations** *(omit if none)*
Bulleted list of items deliberately out of scope for this phase but which will affect
the architecture when addressed (e.g., authentication, real-time sync, microservice
extraction). Reference the phase or spec section where each is deferred.

---

#### HLD File Header

The first lines of `HLD_FILE` must be:

```markdown
# High-Level Design: [FEATURE NAME]

**Branch**: `[branch]` | **Date**: [YYYY-MM-DD] | **Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

**Status**: Draft

---
```

Replace `[FEATURE NAME]`, `[branch]`, and `[YYYY-MM-DD]` with values from the loaded
artifacts (`plan.md` branch line and today's date).

### Step 4 — Validate Before Writing

Before writing `HLD_FILE`, self-check:

- [ ] All Mermaid blocks are syntactically valid (no unclosed brackets, valid node IDs).
- [ ] Sequence diagram message labels contain no `\n` escape sequences, double-quote characters (`"`), or curly braces (`{`, `}`) — these break the Mermaid parser; use plain prose instead.
- [ ] No section heading is empty or contains only placeholder text.
- [ ] Component table covers every node in the architecture diagram.
- [ ] Sequence diagrams exist for every P1 and P2 user story.
- [ ] Constitution Alignment table covers every principle in `constitution.md`.
- [ ] No implementation code (function bodies, SQL queries, full configs) is present —
      HLD describes structure and behaviour, not implementation.
- [ ] Cross-cutting Concerns section explicitly addresses each constitution principle.
- [ ] Artifact references in the HLD use paths relative to `docs/` (e.g.,
      `../specs/NNN-feature/data-model.md`, `../specs/NNN-feature/contracts/api.md`).

If any check fails, fix the content before writing.

### Step 5 — Write HLD File

Write the completed document to `HLD_FILE` (overwrite if exists).

### Step 6 — Update Feature Plan Reference (if exists)

If `FEATURE_DIR/plan.md` exists, check whether it already references `docs/hld.md`.
If not, add the following line to its Documentation section (or append a note at the
end if no such section exists):

```
docs/hld.md          # High-Level Design (/speckit-hld command output — project-level)
```

Do not modify any other section of `plan.md`.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_hld`, skip to the
  Completion Report.
- If it exists, read it and look for entries under the `hooks.after_hld` key.
- If the YAML cannot be parsed or is invalid, do not skip silently: tell the user that
  `.specify/extensions.yml` could not be read (include the parser error) and that no
  hooks were checked, including any mandatory (`optional: false`) hooks registered there,
  then continue to the Completion Report.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled`
  field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition`
  expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable.
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation
    to the HookExecutor implementation.
- When constructing command invocations from hook command names, replace dots (`.`) with
  hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Mandatory hook** (`optional: false`) — **You MUST emit `EXECUTE_COMMAND:` for each
    mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to
    finish before continuing.
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

Report to the user:

- `HLD_FILE` path
- Sections generated (list each section heading)
- Diagrams produced (list each Mermaid diagram type and subject)
- Any artifacts that were absent and left sections shallower
- Suggested next command: `/speckit-tasks` (if not yet run) or `/speckit-implement`

## Done When

- [ ] Input artifacts loaded (spec.md, plan.md, constitution.md minimum)
- [ ] HLD written to `FEATURE_DIR/hld.md` and self-validated
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks rules
- [ ] Completion reported with file path, sections, and diagrams produced
