# Taskify Constitution

## Core Principles

### I. Security-First (NON-NEGOTIABLE)

Security is the primary design constraint of Taskify. Every feature, API endpoint, data
model, and dependency decision MUST be evaluated through a security lens before implementation.

- Security requirements MUST be addressed at design time, not retrofitted post-implementation.
- Threat modeling MUST be performed for every new service or cross-service integration.
- Known vulnerabilities (OWASP Top 10, CVEs in dependencies) MUST be remediated before
  release; no exceptions without a documented, time-bounded risk-acceptance record.
- All secrets (API keys, tokens, credentials) MUST be stored in a secrets manager and MUST NOT
  be committed to version control or emitted in logs.
- Authentication and authorization MUST be enforced server-side on every request; client-side
  enforcement alone is never a sufficient security boundary.

### II. Input Validation (NON-NEGOTIABLE)

All user-supplied and externally sourced input MUST be validated before any processing occurs.

- Validation MUST occur server-side regardless of any client-side checks already in place.
- Every input field MUST be validated against an allow-list schema covering type, range, length,
  and format; requests that fail validation MUST be rejected immediately with a generic error.
- No partial processing of invalid payloads is permitted.
- Validation logic MUST be centralized per service; duplicated or ad-hoc validation scattered
  across handlers is prohibited.
- Structured parsers MUST be preferred over regular expressions; where regex is unavoidable,
  patterns MUST use a non-backtracking engine and cap input length to prevent ReDoS.

### III. Microservices Architecture

Taskify is built as a system of independently deployable, loosely coupled services, each
encapsulating a single bounded domain context.

- Each service MUST own its data store; direct cross-service database access is prohibited.
- Service-to-service communication MUST use well-defined, versioned contracts (OpenAPI,
  AsyncAPI, Protobuf, or equivalent); ad-hoc integrations are not permitted.
- Services MUST be independently deployable and scalable without coordinated shutdowns of
  other services.
- Inter-service trust MUST be explicitly verified (mTLS, signed JWTs, or equivalent); no
  service implicitly trusts another based solely on network location.
- Every service MUST expose health-check and readiness endpoints to support orchestration
  platforms (e.g., Kubernetes liveness/readiness probes).

### IV. Full Documentation

All production code MUST be fully documented at the point of authorship; documentation debt
MUST NOT be deferred.

- Every public function, method, class, module, and service API MUST include a documentation
  comment describing its purpose, parameters, return values, and error conditions.
- Each service MUST maintain an up-to-date README covering: purpose, local setup instructions,
  API contract reference, and known operational constraints.
- Undocumented code MUST NOT be merged; CI gates MUST enforce documentation coverage where
  tooling supports it.
- Inline comments MUST explain non-obvious logic; trivial or self-evident code MUST NOT be
  over-commented to avoid noise.

## Architecture Standards

The following cross-cutting standards apply to all services within the Taskify ecosystem:

- **Service boundaries**: Domain-driven; each service encapsulates exactly one bounded context
  with no overlap into another service's domain.
- **API style**: RESTful HTTP/JSON for synchronous request-response flows; event-driven
  messaging via a durable message broker for asynchronous, cross-service workflows.
- **Security perimeter**: An API gateway MUST handle inbound authentication and coarse-grained
  authorization before requests reach internal services.
- **Observability**: Every service MUST emit structured logs, metrics, and distributed traces.
  PII and secrets MUST be redacted before emission to any log sink or monitoring system.
- **Dependency management**: All third-party dependencies MUST be pinned with integrity hashes
  in a lockfile and scanned by a software-composition-analysis (SCA) tool on every build.
- **HLD compliance**: Every implementation plan MUST conform to and explicitly reference
  `docs/hld.md`. Plans that deviate from the documented architecture without an approved
  Architecture Decision Record (ADR) MUST be rejected before implementation begins.
- **Architecture Decision Records (ADR)**: Any change to the system architecture documented
  in `docs/hld.md` MUST be preceded by an ADR that is created and approved before the
  corresponding plan or implementation proceeds. ADRs MUST be stored in `docs/adr/` and
  MUST follow this structure: **title**, **status** (Proposed / Accepted / Deprecated /
  Superseded), **context** (the problem or constraint driving the decision), **decision**
  (the chosen approach), **consequences** (trade-offs, risks, and follow-up actions).

## Development Workflow

- **Security review**: Every pull request touching authentication, authorization, data storage,
  external integrations, or input handling MUST receive an explicit security review before merge.
- **Test requirements**: Unit tests MUST cover happy-path and edge-case validation logic.
  Integration tests MUST cover inter-service contracts and all security-sensitive flows.
- **Documentation gate**: Pull requests that add or change public APIs, functions, or service
  boundaries MUST include corresponding documentation updates; PRs without them MUST be blocked.
- **Dependency updates**: Third-party dependencies MUST be reviewed for security advisories
  within 7 days of a disclosed CVE; critical or high severity issues MUST be patched within
  24 hours of confirmation.
- **Secrets hygiene**: Secret scanning MUST run in CI on every push; commits containing
  plaintext secrets MUST be blocked from merging.

## Governance

This constitution supersedes all other development guidelines and practices within the Taskify
project. All implementation decisions MUST be traceable to one or more principles stated above.

- **Amendment procedure**: Any amendment MUST be proposed as a pull request to this file,
  reviewed and approved by at least one project lead, and accompanied by a migration plan when
  existing code becomes non-compliant with the new or changed principle.
- **Versioning policy**: Amendments follow semantic versioning — MAJOR for backward-incompatible
  principle removals or redefinitions; MINOR for new principles or materially expanded guidance;
  PATCH for clarifications, wording corrections, and non-semantic refinements.
- **Compliance review**: Compliance with this constitution MUST be verified during every code
  review. Non-compliant code MUST NOT be merged; waivers require a documented, time-bounded
  risk-acceptance record signed off by a project lead.
- **Conflict resolution**: When implementation requirements conflict with a principle stated
  here, the principle takes precedence unless a formal waiver (or an approved ADR for
  architecture deviations) is in place.

**Version**: 1.1.0 | **Ratified**: 2026-11-17 | **Last Amended**: 2026-11-19
