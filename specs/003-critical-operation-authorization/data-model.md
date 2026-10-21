# Data Model: Critical Operation Authorization

## Entity: OperationCredential

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Credential record |
| user_id | UUID | Required, unique FK to `users.user` | Credential owner |
| password_hash | secret | Nullable when `NOT_CONFIGURED`; required when provisioned | Independent from login credential hash |
| status | enum | `NOT_CONFIGURED` \| `ACTIVE` \| `TEMPORARILY_LOCKED` \| `RESET_REQUIRED` \| `DISABLED` | Approved lifecycle states |
| credential_version | bigint | Required | Increment on reset/change; used to invalidate stale confirmations |
| failed_attempt_count | int | Required, default 0 | Lockout tracking |
| lockout_expires_at | timestamp | Optional | End of temporary lockout |
| last_verified_at | timestamp | Optional | Last successful operation-password verification |
| created_at | timestamp | Required | Provisioning timestamp |
| updated_at | timestamp | Required | Last state change |
| row_version | bigint | Required | Concurrency-safe updates |

## Entity: OperationConfirmation

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Confirmation record |
| user_id | UUID | Required FK | Confirming actor |
| operation_identifier | string | Required | Exact operation identifier |
| action | string | Required | Semantic action (e.g., deactivate-user, close-period) |
| resource_type | string | Required | Target aggregate type |
| resource_identifier | string | Required | Target resource id |
| request_fingerprint | string | Required | Deterministic anti-replay/idempotency key |
| resource_version | string/bigint | Required | Resource state/version at confirmation time |
| credential_version | bigint | Required | Must match current credential version at execution |
| status | enum | `PENDING` \| `CONSUMED` \| `REJECTED` \| `EXPIRED` | Single-use lifecycle |
| expires_at | timestamp | Required | Confirmation validity window |
| consumed_at | timestamp | Optional | Consumption timestamp |
| created_at | timestamp | Required | Creation time |

## Business Rules

- Operation credential lifecycle is independent from login credential lifecycle.
- Operation password must never be equal to login password.
- Account deactivation immediately blocks operation authorization usage.
- Account reactivation never auto-restores operation authorization if credential state is `RESET_REQUIRED`, `DISABLED`, or otherwise security-restricted; required lifecycle actions must complete first.
- Reset invalidates credential version and all pending confirmations tied to previous version.
- Temporary lockout auto-expires only when eligibility conditions permit (active account, not reset-required, not disabled, no additional blocking policy).
- RESET_REQUIRED and DISABLED are never auto-cleared by lockout expiry.
- Every confirmation is single-use and bound to exact operation/action/resource/fingerprint/version context.

## State Transitions

### OperationCredential
- `NOT_CONFIGURED` → `ACTIVE` (initial provisioning)
- `ACTIVE` → `TEMPORARILY_LOCKED` (failed-attempt threshold)
- `TEMPORARILY_LOCKED` → `ACTIVE` (automatic expiry when eligible)
- `ACTIVE`/`TEMPORARILY_LOCKED` → `RESET_REQUIRED` (admin/user reset trigger)
- Any state → `DISABLED` (administrative/security disable)
- `RESET_REQUIRED` → `ACTIVE` (successful re-provisioning)

### OperationConfirmation
- `PENDING` → `CONSUMED` (single successful use)
- `PENDING` → `REJECTED` (authorization/resource/version mismatch)
- `PENDING` → `EXPIRED` (time-based expiration)

## Transaction and Concurrency Notes

1. **Reset and version bump**
   - In one transaction: set `status=RESET_REQUIRED`, increment `credential_version`, mark pending confirmations stale/rejected.
2. **Failed-attempt lockout**
   - Atomic increment and transition to `TEMPORARILY_LOCKED` when threshold reached.
3. **Confirmation consumption**
   - Conditional status transition `PENDING -> CONSUMED` with row lock/version check to enforce single-use.
4. **Execution consistency**
   - Re-check active session context, role/permission, resource version, credential version at execution time.
   - Persist distinct security-audit records for credential verification outcome, authorization decision outcome, and business execution outcome, all correlated by `operation_identifier`.

## Shared Audit Infrastructure Contract (Used by `operationauth`)

- Shared envelope: `audit_event(id, module, event_type, actor_id, subject_id, occurred_at, outcome, metadata)`.
- Shared envelope is append-only and immutable through ordinary application interfaces.
- `metadata` must include `operation_identifier` for correlation when an operation confirmation flow is involved.
- `operationauth` event semantics include:
  - `OPERATION_CREDENTIAL_PROVISIONED`
  - `OPERATION_CREDENTIAL_CHANGED`
  - `OPERATION_CREDENTIAL_RESET_REQUIRED`
  - `OPERATION_CREDENTIAL_LOCKED`
  - `OPERATION_CREDENTIAL_UNLOCKED`
  - `OPERATION_CREDENTIAL_VERIFICATION_SUCCEEDED`
  - `OPERATION_CREDENTIAL_VERIFICATION_FAILED`
  - `OPERATION_AUTHORIZATION_APPROVED`
  - `OPERATION_AUTHORIZATION_DENIED`
  - `CRITICAL_OPERATION_EXECUTION_SUCCEEDED`
  - `CRITICAL_OPERATION_EXECUTION_FAILED`
