<!--
Sync Impact Report
Version change: 1.0.1 -> 1.1.0
Modified principles:
- I. Modular Monolith by Business Capability
- III. Secure by Default
Added sections:
- None
Removed sections:
- None
Deferred items:
- None
-->

# Go Modular API Constitution

## Core Principles

### I. Modular Monolith by Business Capability
The system MUST be organized as a modular monolith with clear business-capability boundaries and a single deployable application. Modules MUST own their domain logic, interfaces, and persistence concerns without creating circular dependencies or shared implementation leaks. Business data MUST have a single owning module, and cross-module interactions MUST use explicit application-level contracts instead of direct access to another module's repositories, persistence models, or database access code. This structure preserves delivery speed while keeping the codebase understandable, reviewable, and evolvable.

### II. Pragmatic Clean Architecture in Go
The project MUST follow pragmatic Clean Architecture adapted to idiomatic Go. Domain models MUST remain independent from Gin, GORM, PostgreSQL-specific code, and framework concerns. Dependencies MUST be injected explicitly through constructors, and interfaces MUST be defined at their point of use when they provide real architectural value. Unnecessary abstractions, generic repositories, and excessive layers are prohibited unless the team can justify a concrete need. This keeps business rules stable while minimizing accidental coupling.

### III. Secure by Default
The API MUST be secure by default. Authentication and authorization are distinct responsibilities. Authentication MUST verify identity, support session expiration and revocation, and reject invalid or stale sessions. Authorization MUST follow least-privilege principles, enforce access decisions server-side for every protected action, and deny access by default. Passwords MUST be hashed with Argon2id, and all external inputs MUST be validated before use. CORS policies MUST be explicit, rate limiting MUST be configurable, secret material MUST never be logged, and HTTP server defaults MUST include sane timeouts and secure configuration. Security decisions MUST be deliberate; insecure defaults are not acceptable.

### IV. Persistence Integrity and Transactional Clarity
The application MUST use PostgreSQL with GORM. Persistence models MUST be separated from domain models when architectural boundaries require it, schema changes MUST be managed through versioned migrations, and database errors MUST be handled explicitly. Business operations that require atomicity MUST use transactions. This ensures correctness, recoverability, and predictable behavior under failure.

### V. Quality, Testing, and Explicit Error Handling
The codebase MUST follow idiomatic Go conventions and prefer the standard library when practical. Errors MUST be handled explicitly, request-scoped operations MUST use context.Context, and premature optimization or unnecessary dependencies MUST be avoided. Business logic MUST be covered by unit tests, persistence behavior MUST be verified by integration tests, and API behavior MUST be validated by HTTP tests. All tests MUST pass before an implementation is accepted. This maintains correctness and keeps the codebase maintainable.

## Architecture & Technical Constraints
The project MUST use Go, Gin, GORM, and PostgreSQL in a single deployable application backed by one shared PostgreSQL database. Domain logic MUST remain decoupled from transport and persistence frameworks, and module boundaries MUST be explicit. Dependency injection through constructors is required, but not every dependency needs a framework-level abstraction. Teams MUST avoid circular dependencies, hidden cross-module access, and architectural layering that adds ceremony without a measurable benefit.

## Development Workflow
The project MUST use Spec-Driven Development with GitHub Spec Kit. Specifications MUST precede implementation, and implementation MUST follow approved specifications and technical plans. Agents and contributors MUST NOT introduce architectural changes without documenting and reviewing them. Work MUST be broken into small, independently verifiable tasks, with each change validated against the governing specification and the quality gates in this constitution. This keeps changes traceable and reduces accidental drift from the approved design.

## Governance
This constitution supersedes ad hoc practices for this project. Any amendment MUST be proposed in writing, reviewed for architectural impact, and documented before implementation proceeds. Versioning MUST follow semantic versioning: MAJOR for backward-incompatible governance or principle changes, MINOR for new principles or materially expanded guidance, and PATCH for clarifications, wording fixes, or non-semantic refinements. Compliance review MUST verify that implementation, design decisions, and review comments align with this constitution; any deviation requires explicit justification and approval before merge.

**Version**: 1.1.0 | **Ratified**: 2026-10-12 | **Last Amended**: 2026-10-12
