# Implementation Plan: Authentication and Session Management

**Branch**: `002-authentication-session-management` | **Date**: 2026-10-20 | **Spec**: [spec.md](./spec.md)

**Input**: Approved feature specification from `/specs/002-authentication-session-management/spec.md`

## Summary

This feature defines the `auth` capability for login credentials, activation invitations, authentication, and stateful opaque sessions. It keeps Go authoritative for authentication/session validity and integrates with Next.js BFF for cookie transport without delegating authorization decisions to the BFF.

## Technical Context

**Language/Version**: Go 1.23+  
**Primary Dependencies**: Gin, GORM, PostgreSQL, crypto utilities, rate-limiting utilities, structured logging  
**Storage**: PostgreSQL with versioned migrations  
**Testing**: Unit + integration + HTTP/BFF boundary tests  
**Target Platform**: Linux single deployable backend with Next.js BFF and reverse proxy  
**Project Type**: Backend authentication and session capability  

**Constraints**
- Stateful opaque sessions only (no JWT/JWE/refresh-token/OAuth flow in initial scope).
- Session renewal threshold configurable and mandatory when reached.
- CSRF protection for state-changing browser requests.
- Session token secrecy: hash-at-rest, never log raw token.
- Fail-closed behavior if authoritative session state cannot be validated.

## Constitution Check

*GATE: Must pass before implementation.*

- **Modular Monolith by Business Capability**: Pass. `auth` owns credentials/invitations/sessions end-to-end.
- **Pragmatic Clean Architecture in Go**: Pass. Domain is framework-independent; adapters are isolated.
- **Secure by Default**: Pass. Active-account enforcement, session revocation, non-disclosing failures, and CSRF/cookie constraints are mandatory.
- **Persistence Integrity and Transactional Clarity**: Pass. Invitation and session transitions are transaction-safe.
- **Quality, Testing, and Explicit Error Handling**: Pass. Security-critical branches are explicitly testable.

## Cross-Feature Architecture Alignment

### Module Ownership

- **auth**: `LoginCredential`, `ActivationInvitation`, `Session`, login attempts, auth/session audit semantics.
- **users**: account status/roles and directory operations.
- **operationauth**: operation-credential and confirmation lifecycle.

### Dependency Direction (Application Contracts)

- `auth` consumes:
  - account status and role context from `users` contract.
- `auth` publishes:
  - authenticated actor/session context contract for `users` and `operationauth`.
  - session revocation contract callable by `users` on deactivation.
- No module imports another module’s infrastructure package.

### Shared Infrastructure

- `shared/security`: Argon2id, token hashing, secure compare, random generation primitives.
- `shared/authorization`: actor context contract and policy contract interfaces.
- `shared/auditing`: append-only audit writer with module/event metadata.

## Project Structure

```text
internal/
├── modules/
│   ├── auth/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── interfaces/
│   ├── users/
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

1. **Invitation replacement and redemption**
   - Replace/revoke and create-new invitation in one transaction.
   - Redemption uses token-hash lookup + row lock; consume only if not revoked/expired/consumed.
   - Redemption and initial password provisioning commit atomically.

2. **Session renewal and revocation**
   - Renewal executes conditional update: only non-revoked, non-expired sessions below absolute expiry.
   - Use row version or lock to prevent concurrent renewal/regression.
   - Revocation (logout/admin/deactivation) sets `revoked_at` and reason atomically; renewal must reject revoked rows.

3. **Account deactivation and session invalidation**
   - Account deactivation and session revocation are transactionally coordinated in the shared PostgreSQL monolith so all sessions are revoked before the deactivation transaction commits.
   - Outbox can be used for secondary effects (notifications/integration side effects) but not as the sole security mechanism for revocation enforcement.
   - Revocation remains idempotent for retries and race safety.

4. **Concurrent authentication attempts**
   - Abuse controls persisted per actor/source dimensions.
   - Generic failure responses regardless of account existence.

## Security Alignment

- Admin-only account creation and no public registration preserved via `users` + `auth` contract.
- Initial password provisioning via secure, single-use, expiring activation invitations.
- No administrator knowledge of user passwords.
- BFF cookie settings enforce HttpOnly, Secure, host-only, and appropriate SameSite.
- CSRF enforced for state-changing requests.
- Current-role authorization validated from authoritative server-side state.
- Every protected request validates both session state and current account active status authoritatively, failing closed if either check cannot be verified.

## Complexity Tracking

No constitution exception required. Complexity comes from security/concurrency, addressed through explicit state machines and transactional guards.
