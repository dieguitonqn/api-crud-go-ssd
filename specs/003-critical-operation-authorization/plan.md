# Implementation Plan: Critical Operation Authorization

**Branch**: `003-critical-operation-authorization` | **Date**: 2026-10-20 | **Spec**: [spec.md](./spec.md)

**Input**: Approved feature specification from `/specs/003-critical-operation-authorization/spec.md`

## Summary

This feature defines `operationauth`, the independent operation-credential capability used to confirm critical actions. It enforces strict separation from login authentication and session lifecycle, supports secure credential lifecycle states, and guarantees single-use, resource-bound confirmations with replay/idempotency protection.

## Technical Context

**Language/Version**: Go 1.23+  
**Primary Dependencies**: Gin, GORM, PostgreSQL, hashing/security primitives, auditing infrastructure  
**Storage**: PostgreSQL with versioned migrations  
**Testing**: Unit + integration + API tests for lifecycle transitions and race conditions  
**Target Platform**: Linux single deployable backend behind Next.js BFF  
**Project Type**: Backend authorization capability module  

**Constraints**
- Operation password lifecycle independent from login credential lifecycle.
- No session creation/renewal/elevation from operation-password verification.
- Single-use confirmation bound to action + resource + request fingerprint + versions.
- Automatic lockout expiration when security conditions allow; no automatic clearing of RESET_REQUIRED/DISABLED.

## Constitution Check

*GATE: Must pass before implementation.*

- **Modular Monolith by Business Capability**: Pass. `operationauth` owns operation credentials and confirmations only.
- **Pragmatic Clean Architecture in Go**: Pass. Domain policies are independent from frameworks.
- **Secure by Default**: Pass. Explicit checks for active account, valid session, role/permission, and resource state.
- **Persistence Integrity and Transactional Clarity**: Pass. Confirmation consumption and execution consistency are transaction-bound.
- **Quality, Testing, and Explicit Error Handling**: Pass. Lockout/reset/race paths are explicit and testable.

## Cross-Feature Architecture Alignment

### Module Ownership

- **operationauth**: operation credential state machine + confirmation lifecycle.
- **auth**: login/session validity and actor identity context.
- **users**: account role/active status and directory lifecycle.

### Dependency Direction (Application Contracts)

- `operationauth` consumes:
  - authenticated actor/session context from `auth`.
  - current account status/role from `users`.
- `operationauth` does not import `auth`/`users` infrastructure.
- Critical business modules consume confirmation result via contract; confirmation records remain owned by `operationauth`.

### Shared Infrastructure

- `shared/security`: password hashing/compare and secure random helpers.
- `shared/authorization`: policy contracts for role/permission evaluation.
- `shared/auditing`: append-only audit envelope and writer abstraction.

## Project Structure

```text
internal/
├── modules/
│   ├── operationauth/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── interfaces/
│   ├── auth/
│   ├── users/
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

1. **Operation credential reset and lockout**
   - Reset invalidates current credential version and outstanding confirmations atomically.
   - Failed-attempt increments and lockout transitions are atomic and race-safe.
   - Lockout expiration automatically returns to ACTIVE only if state is eligible.
   - Account reactivation never auto-restores operation authorization when credential state remains `RESET_REQUIRED` or `DISABLED`; previously imposed restrictions remain effective until the required lifecycle action completes.

2. **Single-use confirmation issuance/consumption**
   - Confirmation row includes exact operation/action/resource/fingerprint/version data.
   - Consumption uses conditional update (`status=PENDING`) to enforce single-use under concurrency.

3. **Critical operation execution and audit consistency**
   - Persist three distinct audit outcomes correlated by `operation_identifier`: credential verification, authorization decision, and business execution.
   - Execute confirmation-consumption + business operation + mandatory security-audit persistence in one transaction where possible.
   - If business operation is in another module/transaction boundary, use transactional outbox only for secondary continuation, while mandatory security audit events are still durably persisted at each decision point.

4. **Replay and idempotency protection**
   - Unique constraints on operation identifier + user + request fingerprint + confirmation state window.
   - Repeated or stale requests resolve to deterministic rejection or idempotent no-op.

## Security Alignment

- Preserves independent login and operation passwords.
- Preserves session revocation behavior by requiring current valid auth context.
- Denies confirmation on inactive accounts, insufficient permissions, revoked/expired sessions, or stale resource versions.
- Preserves restriction continuity across account deactivation/reactivation: operation authorization is never auto-restored when reset/security restrictions remain unresolved.
- BFF may collect operation password but must not persist it; authoritative verification is backend-only.

## Complexity Tracking

No constitution exception required. Complexity is security-centric and managed through explicit state transitions and transactional safeguards.
