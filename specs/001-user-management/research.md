# Research: User Management

## Decision 1: Strict Module Ownership for Account Governance
- **Decision**: `users` owns account records, roles, administrative status, and directory operations only.
- **Rationale**: Matches approved specs and prevents credential leakage/coupling.
- **Alternatives considered**:
  - Store login/operation credential data in `users` for convenience (rejected: violates ownership and separation of concerns).

## Decision 2: Separation of Account State vs Credential Provisioning State
- **Decision**: `users.active` and role state are governed in `users`; login-credential provisioning state is governed in `auth`.
- **Rationale**: Approved requirements explicitly separate administrative status from credential provisioning lifecycle.
- **Alternatives considered**:
  - Single combined state machine in `users` (rejected: collapses responsibilities and increases coupling).

## Decision 3: Last-Active-Administrator Safety Under Concurrency
- **Decision**: enforce with transaction-scoped recheck and lock-protected count before commit.
- **Rationale**: Required to prevent race conditions in concurrent role/deactivation operations.
- **Alternatives considered**:
  - Application-only pre-check without transactional recheck (rejected: race-prone).

## Decision 4: Deactivation-Driven Session Invalidity Coordination
- **Decision**: account deactivation and session revocation are coordinated synchronously in one shared-DB transaction boundary; outbox is reserved for secondary effects only.
- **Rationale**: Approved security requirements demand immediate revocation effectiveness and must not depend solely on asynchronous processing.
- **Alternatives considered**:
  - Outbox-only revocation propagation (rejected: non-immediate security enforcement).
  - `users` directly updating session tables through infrastructure coupling (rejected: cross-module persistence breach).

## Decision 5: Shared Audit Infrastructure with Module Semantics
- **Decision**: Use shared append-only audit envelope infrastructure, while event semantics remain module-owned.
- **Rationale**: Ensures consistent audit storage without centralizing business meaning.
- **Alternatives considered**:
  - Module-specific audit tables without shared envelope (rejected: inconsistent retention/query semantics).

## Remaining Technical Decisions

- Exact indexing strategy for directory search fields (name/email) for target data volumes.
- Whether active-admin protection uses pessimistic locking, optimistic retries, or hybrid under high write contention.
