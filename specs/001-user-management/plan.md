# Implementation Plan: User Management

**Branch**: `001-user-management` | **Date**: 2026-10-20 | **Spec**: [spec.md](./spec.md)

**Input**: Approved feature specification from `/specs/001-user-management/spec.md`

## Summary

This feature defines the `users` business capability for account directory and lifecycle governance. The module owns account records, administrative status, role assignments, listing/search, and lifecycle invariants (including last-active-administrator protection). Login credentials, activation invitations, and session lifecycle remain owned by `auth`; operation credentials and critical-operation confirmation remain owned by `operationauth`.

## Technical Context

**Language/Version**: Go 1.23+  
**Primary Dependencies**: Gin, GORM, PostgreSQL, structured logging, validation helpers  
**Storage**: PostgreSQL with versioned migrations  
**Testing**: Unit + integration + HTTP authorization tests  
**Target Platform**: Linux single deployable backend (modular monolith) behind Next.js BFF  
**Project Type**: Backend API capability module  

**Constraints**
- Admin-only account creation; no public registration.
- Domain independent from Gin/GORM/PostgreSQL types.
- No duplicated user domain models across modules.
- No circular imports; module interaction only through application contracts/ports.
- No generic repositories; focused repositories per aggregate/use case.

## Constitution Check

*GATE: Must pass before implementation.*

- **Modular Monolith by Business Capability**: Pass. `users` owns account lifecycle and directory operations; no credential storage in this module.
- **Pragmatic Clean Architecture in Go**: Pass. Domain rules live in `users/domain`; adapters isolated in infrastructure.
- **Secure by Default**: Pass. Least privilege, self-protection safeguards, and secret exclusion are mandatory.
- **Persistence Integrity and Transactional Clarity**: Pass. Last-admin and concurrent mutations are transaction-bound.
- **Quality, Testing, and Explicit Error Handling**: Pass. Invariants are directly testable at domain and persistence levels.

## Cross-Feature Architecture Alignment

### Module Ownership

- **users**: user records, role, active status, directory listing/filter/search, account lifecycle invariants.
- **auth**: login credentials, activation invitations, initial password provisioning, authentication, sessions/revocation.
- **operationauth**: operation credential lifecycle and critical-operation confirmation.

### Dependency Direction (Application Contracts)

- `users` depends on:
  - auth-provided authenticated actor context contract (read-only identity/role/session context).
  - auth session-revocation contract (command) for deactivation-triggered invalidation.
- `users` does **not** import auth/operationauth infrastructure implementations.
- `users` publishes account-state events/commands consumed by `auth` and `operationauth` through internal application contracts.

### Shared Infrastructure (Narrow Scope)

- `shared/security`: hashing interfaces and secret-safe comparison primitives (no business rules).
- `shared/authorization`: policy contracts and actor context abstractions.
- `shared/auditing`: append-only audit writer contract and event envelope schema.

## Project Structure

```text
internal/
├── modules/
│   ├── users/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── interfaces/
│   ├── auth/
│   ├── operationauth/
│   └── shared/
│       ├── security/
│       ├── authorization/
│       └── auditing/
└── platform/
    ├── db/
    ├── http/
    └── logger/
```

## Transaction and Concurrency Strategy

1. **Last-active-administrator protection**
   - Perform role/deactivation mutation in a DB transaction.
   - Re-check active-admin count with lock-protected query before commit.
   - Reject commit if resulting count is zero.

2. **Concurrent user creation with unique email**
   - Enforce case-insensitive unique index at DB level.
   - Normalize email before persistence.
   - Translate uniqueness violation into deterministic business error.

3. **Account deactivation and session invalidation**
   - Deactivation executes as a transactionally coordinated workflow in the shared PostgreSQL monolith: set user `active=false` and revoke all sessions for that user before commit.
   - Outbox is allowed only for secondary asynchronous effects (notifications, downstream projections), never as the sole enforcement path for session invalidation.

4. **Administrative self-protection**
   - Reject self role-change/self-deactivation before persistence.
   - Re-validate actor identity from authoritative auth context, fail closed if unavailable.

## Security Alignment

- Password material never stored or returned by `users`.
- Activation invitation and credential provisioning status are external to `users`, but `users` must not bypass those constraints.
- Current-role authorization evaluated server-side on each admin operation.
- Every protected request must validate current administrative account status authoritatively from server-side state; stale sessions cannot authorize when account status is inactive.
- All failures return explicit business errors without secret leakage.

## Complexity Tracking

No constitution exception required. Complexity is bounded by strict ownership and explicit cross-module contracts.
