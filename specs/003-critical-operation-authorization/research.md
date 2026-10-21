# Research: Critical Operation Authorization

## Decision 1: Dedicated `operationauth` Capability
- **Decision**: Keep independent module ownership for operation credentials and confirmations.
- **Rationale**: Prevents coupling with login/session or user-directory concerns.
- **Alternatives considered**:
  - Merge into `auth` (rejected: would blur independent lifecycle and broaden blast radius).

## Decision 2: Explicit Credential State Machine
- **Decision**: Support states `NOT_CONFIGURED`, `ACTIVE`, `TEMPORARILY_LOCKED`, `RESET_REQUIRED`, `DISABLED` with credential versioning.
- **Rationale**: Required by approved specification and needed for deterministic transitions under reset/lockout.
- **Alternatives considered**:
  - Boolean active/inactive only (rejected: insufficient for required lifecycle/security semantics).

## Decision 3: Confirmation as Single-Use, Resource-Bound Authorization Artifact
- **Decision**: confirmation stores operation identifier, action, resource type/id, request fingerprint, resource version, credential version, expiration, and consumption state.
- **Rationale**: Enforces replay protection, idempotency, and exact-target authorization.
- **Alternatives considered**:
  - Reusable short-lived confirmation token (rejected: creates privilege window and replay risk).

## Decision 4: Lockout Expiration Behavior
- **Decision**: lockout auto-expires to ACTIVE when eligibility conditions are satisfied; RESET_REQUIRED and DISABLED are never auto-cleared by lockout expiry.
- **Rationale**: Aligns security resilience with approved mandatory behavior.
- **Alternatives considered**:
  - Manual unlock only (rejected: conflicts with mandatory automatic expiration requirement).

## Decision 5: Reactivation Restriction Continuity
- **Decision**: account reactivation does not automatically restore operation authorization; prior reset/security restrictions remain effective until their explicit lifecycle resolution.
- **Rationale**: Preserves approved separation between account active status and operation-credential usability.
- **Alternatives considered**:
  - Auto-restoring operation authorization on account reactivation (rejected: violates approved security behavior).

## Decision 6: Shared Audit Envelope with Module-Owned Event Semantics
- **Decision**: shared append-only audit infrastructure; `operationauth` defines semantic event types.
- **Rationale**: consistent storage and compliance queryability without cross-module business leakage.
- **Alternatives considered**:
  - Generic undifferentiated audit message stream (rejected: weak incident-response value).

## Remaining Technical Decisions

- TTL and retention windows for confirmation/audit records by compliance tier.
- Canonical request fingerprint inputs (fields included/excluded) across critical operation types.
