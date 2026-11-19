# ADR-0004: Generated TypeScript Client from OpenAPI Schema (@hey-api/openapi-ts)

**Status**: Accepted
**Date**: 2026-11-19
**Deciders**: Taskify project lead

---

## Context

The frontend (React 18 + TypeScript strict mode) must call the FastAPI backend. Options
for maintaining the API contract between backend and frontend:

| Option | Type safety | Backend drift detection | Maintenance |
|--------|-------------|------------------------|-------------|
| Hand-written fetch wrappers | Manual casts | None — silent drift | High |
| OpenAPI-generated client | Fully typed | Build-time TS errors | Minimal |
| GraphQL | Fully typed | Schema enforcement | High (schema layer) |
| tRPC | Fully typed | Shared types only | Requires Node backend |

The backend is FastAPI, which generates a standards-compliant OpenAPI 3.x schema at
`/openapi.json` automatically from Pydantic v2 response models. This schema is the
single source of truth for the API contract.

---

## Decision

Use **`@hey-api/openapi-ts`** to generate a fully typed TypeScript client from
`http://localhost:8000/openapi.json` into `frontend/src/client/`. Generation is triggered
via `npm run generate-client` and MUST be re-run after any backend schema change.

The generated `src/client/` directory is **never hand-edited**. All frontend API calls go
through `src/lib/apiClient.ts`, which configures the generated client's base URL and
injects the `X-User-Id` header from the user store.

---

## Consequences

**Benefits**:
- TypeScript compile errors immediately surface when backend response shapes change and
  the client has not been regenerated — breaking contract drift is caught at build time,
  not runtime.
- Zero hand-maintained fetch boilerplate; all endpoint signatures, request/response types,
  and error shapes are derived from the backend's Pydantic schemas.

**Constraints**:
- The backend MUST be running on port 8000 to regenerate the client; CI must start the
  backend before the TypeScript type-check step.
- Every Pydantic response model on the backend MUST have explicit field declarations (no
  `model_config = ConfigDict(arbitrary_types_allowed=True)` without documented fields)
  to produce a complete OpenAPI schema.

**Follow-up actions**:
- Add `npm run generate-client` as a pre-build step in CI after the backend health check
  passes.
- If the backend is later split into multiple services, generate separate clients per
  service and compose them in `apiClient.ts`.
