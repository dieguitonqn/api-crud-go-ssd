# Data Model: Authentication and Session Management

## Entity: LoginCredential

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Credential record |
| user_id | UUID | Required, unique FK to `users.user` | One login credential per user |
| password_hash | secret | Required when provisioned | Argon2id hash; never returned |
| provisioning_state | enum | `PENDING_ACTIVATION` \| `ACTIVE` \| `RESET_REQUIRED` | Explicit provisioning lifecycle |
| password_version | bigint | Required | Increment on credential reset/change |
| last_password_change_at | timestamp | Optional | Security/audit support |
| created_at | timestamp | Required | Creation time |
| updated_at | timestamp | Required | Last update |

## Entity: ActivationInvitation

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Invitation record |
| user_id | UUID | Required FK | Invitation owner |
| issued_by_user_id | UUID | Required FK | Administrator issuer |
| token_hash | secret | Required, unique | Store only hash of opaque token |
| expires_at | timestamp | Required | Automatic expiration |
| consumed_at | timestamp | Optional | Single-use consumption marker |
| revoked_at | timestamp | Optional | Revocation marker |
| revoked_by_user_id | UUID | Optional FK | Administrator revoker |
| replacement_for_invitation_id | UUID | Optional FK self-reference | Links replacement chain |
| created_at | timestamp | Required | Issuance time |

## Entity: Session

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Session identifier |
| user_id | UUID | Required FK | Session owner |
| token_hash | secret | Required, unique | Opaque token hash only |
| created_at | timestamp | Required | Issue time |
| last_seen_at | timestamp | Required | Last validated/renewed time |
| idle_expires_at | timestamp | Required | Sliding expiration boundary |
| absolute_expires_at | timestamp | Required | Absolute expiration boundary |
| revoked_at | timestamp | Optional | Revocation marker |
| revoked_reason | enum | Optional | `LOGOUT_SELF`, `LOGOUT_ALL`, `ADMIN_REVOKE`, `ACCOUNT_DEACTIVATED`, `SECURITY_EVENT` |
| row_version | bigint | Required | Concurrency-safe renewal/revocation |

> `is_active` is intentionally removed. Session validity derives from timestamps + revocation fields.

## Entity: AuthenticationAttempt

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | Attempt record |
| user_id | UUID | Optional FK | Populated when account exists |
| normalized_email | string | Optional | For analysis/abuse controls |
| source_fingerprint | string | Optional | Client/source dimension |
| success | boolean | Required | Authentication result |
| occurred_at | timestamp | Required | Attempt time |
| outcome_reason | enum | Required | Generic classification without account-existence leakage |

## Business Rules

- Only active user accounts with `LoginCredential.provisioning_state=ACTIVE` may authenticate.
- Initial provisioning requires valid unexpired invitation and consumes invitation exactly once.
- Revoked, replaced, expired, or already consumed invitations are invalid.
- Session renewal is mandatory once threshold is reached and session remains eligible.
- Expired/revoked sessions are never renewed and never authorize protected operations.
- Revocation is authoritative and takes precedence over renewal races.
- Account deactivation must immediately revoke all existing sessions via transactionally coordinated deactivation+revocation in the shared PostgreSQL monolith.
- Every protected request must validate both session validity and current account active status authoritatively before authorization.
- Password material and raw tokens are never logged or returned in API responses.

## State Transitions

### LoginCredential
- `PENDING_ACTIVATION` → `ACTIVE` (successful invitation redemption + password setup)
- `ACTIVE` → `RESET_REQUIRED` (security/admin reset trigger)
- `RESET_REQUIRED` → `ACTIVE` (completed reset flow)

### ActivationInvitation
- `ISSUED` → `CONSUMED` (successful one-time redemption)
- `ISSUED` → `REVOKED` (admin action)
- `ISSUED` → `EXPIRED` (time-based invalidation)
- `ISSUED` → `REPLACED` (replacement flow; old marked revoked/replaced)

### Session
- `ISSUED` → `VALID` → `RENEWED` (threshold-based, conditional)
- `VALID/RENEWED` → `REVOKED` (logout/admin/deactivation/security)
- `VALID/RENEWED` → `EXPIRED` (idle/absolute expiration)

## Transaction and Concurrency Notes

1. **Invitation redemption**
   - lock invitation row; check `revoked_at IS NULL`, `consumed_at IS NULL`, `expires_at > now()`.
   - set `consumed_at` and activate credential in same transaction.
2. **Invitation replacement/revocation**
   - revoke prior invitation and issue replacement atomically.
3. **Session renewal**
   - conditional update on `row_version`, non-revoked, non-expired predicates.
4. **Session revocation**
   - idempotent update setting `revoked_at` if null; safe under retries.
5. **Deactivation + revocation coordination**
   - deactivation workflow must guarantee that account deactivation and all-session revocation are committed as one security-consistent unit of work.
   - outbox may publish secondary effects but cannot be the sole authorization-control path.

## Shared Audit Infrastructure Contract (Used by `auth`)

- Shared envelope: `audit_event(id, module, event_type, actor_id, subject_id, occurred_at, outcome, metadata)`.
- `auth` event semantics include:
  - `ACTIVATION_INVITATION_ISSUED`
  - `ACTIVATION_INVITATION_REVOKED`
  - `ACTIVATION_INVITATION_REPLACED`
  - `ACTIVATION_INVITATION_REDEEMED`
  - `AUTHENTICATION_SUCCEEDED`
  - `AUTHENTICATION_FAILED`
  - `SESSION_RENEWED`
  - `SESSION_REVOKED`
