# ADR-0001: Defer Authentication to Phase 2 (X-User-Id Header for Phase 1)

**Status**: Accepted
**Date**: 2026-11-19
**Deciders**: Taskify project lead
**Constitution reference**: Principle I — Security-First (justified deviation)

---

## Context

The Taskify phase 1 feature spec (`specs/001-team-kanban-board/spec.md`) explicitly scopes
out login for the first phase. The five team members (Alice Chen, Bob Kumar, Carol Santos,
David Park, Eva Müller) are predefined internal users operating in a trusted local development
environment accessed only via Docker Compose. There is no network exposure beyond localhost.

The project constitution (Principle I — Security-First) requires authentication and
authorization to be enforced server-side on every request. Implementing OAuth2/JWT for a
local-only, single-team prototype before the core Kanban domain is validated would delay
delivery without commensurate risk reduction given the deployment context.

---

## Decision

For phase 1 only, user identity is conveyed via an `X-User-Id: <int>` request header on all
write operations (`POST`, `PATCH`, `DELETE`). The backend validates that the header value is
one of the five seeded user IDs (1–5) and rejects with HTTP 401 if missing or invalid.
Read operations (`GET`) require no identity header.

Authentication (OAuth2/JWT or equivalent) MUST be implemented before any network-exposed or
multi-tenant deployment. This deviation is explicitly time-bounded to phase 1.

---

## Consequences

**Accepted risks**:
- Any client can impersonate any of the five users by supplying an arbitrary `X-User-Id`
  value; there is no cryptographic proof of identity.
- This pattern MUST NOT be used beyond the localhost Docker Compose context.

**Mitigations in place**:
- The backend rejects header values that are not in the seeded user set (1–5); unknown IDs
  return HTTP 401.
- The deviation is documented here, in `plan.md` Constitution Check, and in
  `docs/hld.md` Section 8 (Cross-Cutting Concerns — Security).
- A tracking obligation exists: authentication MUST be revisited in phase 2 before any
  deployment beyond localhost.

**Follow-up actions**:
- Phase 2: replace `X-User-Id` with JWT Bearer token; update `deps.py` resolver and all
  write endpoints; add token issuance endpoint or OAuth2 provider integration.
- Phase 2: `deps.py` is already structured to swap the identity resolver without touching
  route handlers — the coupling surface is isolated.
