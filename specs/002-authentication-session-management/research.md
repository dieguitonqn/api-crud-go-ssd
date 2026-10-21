# Research: Authentication and Session Management

## Decision 1: Auth as Sole Owner of Login Credentials and Sessions
- **Decision**: `auth` owns login credential records, provisioning state, invitation lifecycle, and session lifecycle.
- **Rationale**: Keeps business ownership clear and prevents user-model duplication.
- **Alternatives considered**:
  - Split sessions into a separate business module (rejected for this scope: ownership must remain in auth per approved module map).

## Decision 2: Initial Password Provisioning via Activation Invitations
- **Decision**: administrator-issued invitation with hashed token storage, expiration, single-use consumption, and replacement/revocation support.
- **Rationale**: Satisfies security requirements without exposing credentials to administrators.
- **Alternatives considered**:
  - Admin-generated initial passwords (rejected: violates no-admin-knowledge requirement).
  - Non-expiring setup links (rejected: weak security posture).

## Decision 3: Session Model and Renewal Semantics
- **Decision**: opaque token with hashed storage, absolute + sliding expiration, explicit revocation state, mandatory renewal at threshold.
- **Rationale**: Required by approved spec and secure-by-default constraints.
- **Alternatives considered**:
  - JWT/self-contained tokens (rejected by approved requirements).
  - Optional renewal only (rejected: renewal must be mandatory at threshold).

## Decision 4: Concurrency-Safe Session Renewal and Revocation
- **Decision**: conditional updates with row version/locking and precedence for revocation over renewal.
- **Rationale**: Prevents race-induced resurrection or shortening of validity.
- **Alternatives considered**:
  - Last-write-wins without guard conditions (rejected: race-prone and insecure).

## Decision 5: Immediate Revocation on Account Deactivation
- **Decision**: deactivation-triggered session revocation is enforced synchronously in a transactionally coordinated shared-DB workflow; outbox is secondary only.
- **Rationale**: Security-critical invalidation must be immediate and not depend solely on asynchronous delivery.
- **Alternatives considered**:
  - Outbox-only revocation propagation (rejected: timing gap for protected request authorization).

## Decision 6: Shared Audit Envelope, Auth-Owned Event Semantics
- **Decision**: use shared append-only audit infrastructure; auth defines event semantics for invitation/auth/session events.
- **Rationale**: consistent storage and querying with module-local meaning.
- **Alternatives considered**:
  - Centralized cross-module event taxonomy only (rejected: blurs ownership semantics).

## Remaining Technical Decisions

- Exact renewal-throttle interval and abuse thresholds by environment.
- Final CSRF token transport pattern at BFF boundary (double-submit vs server-state strategy) while preserving approved constraints.
