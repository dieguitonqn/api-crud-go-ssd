# Data Model: User Management

## Aggregate: User

> `users` is the authoritative owner of account directory and lifecycle metadata.  
> Login credentials and operation credentials are **not** stored in this aggregate.

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK, immutable | User identifier |
| email | string | Required, normalized, case-insensitive unique | Unique across active and inactive users |
| name | string | Required | Display name |
| role | enum | `admin` \| `user` | Authorization role |
| active | boolean | Required, default `true` | Administrative account status |
| created_at | timestamp | Required, immutable | Creation timestamp |
| updated_at | timestamp | Required | Last update timestamp |
| row_version | bigint | Required | Optimistic concurrency token |

## Explicit Non-Ownership

- `password_hash` and login-credential provisioning state belong to `auth`.
- `operation_credential_hash` and operation credential state belong to `operationauth`.

## Business Rules

- Only authenticated administrators can create/update/deactivate accounts.
- Public registration is not supported.
- Email uniqueness is enforced case-insensitively at persistence boundary.
- Administrators cannot change their own role or deactivate their own account.
- System must reject any mutation that would leave zero active administrators.
- Deactivation must immediately revoke all existing sessions for the account through a transactionally coordinated `users` + `auth` workflow in the shared PostgreSQL monolith.
- Protected requests must re-validate current account active status authoritatively; deactivated accounts cannot authorize requests even with previously issued session tokens.

## State Transitions

### Account Lifecycle
- **Created** (`active=true`) → **Active**
- **Active** → **Deactivated** (`active=false`)
- **Deactivated** → **Reactivated** (`active=true`, requires fresh authentication in auth flow)

## Transaction and Concurrency Notes

1. **Last-active-admin invariant**
   - Lock-protected count/recheck in same transaction as role/deactivation mutation.
2. **Concurrent creation**
   - Unique email index guarantees correctness; app maps DB conflict to business error.
3. **Deactivation side effects**
   - In one coordinated transaction boundary: persist user deactivation and revoke all user sessions before commit.
   - Outbox/audit events may be emitted for secondary asynchronous processing but are not the primary security-control path.

## Shared Audit Infrastructure Contract (Used by `users`)

- `audit_event` envelope fields (shared): `id`, `module`, `event_type`, `actor_id`, `subject_id`, `occurred_at`, `outcome`, `metadata`.
- `users`-owned event semantics include:
  - `USER_CREATED`
  - `USER_UPDATED`
  - `USER_DEACTIVATED`
  - `USER_REACTIVATED`
  - `LAST_ADMIN_MUTATION_REJECTED`
