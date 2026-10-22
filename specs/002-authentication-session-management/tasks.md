# Tasks: Authentication and Session Management

**Input**: Design documents from `/specs/002-authentication-session-management/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, and the approved feature 001 task boundary in `/specs/001-user-management/tasks.md`

**Scope**: This feature only covers the `auth` business capability for login credentials, activation invitations, server-side session management, login abuse controls, and auth-owned bootstrap responsibilities. Users owns account state, roles, and lifecycle invariants; operationauth owns operation credentials and critical-operation confirmation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: `US1`, `US2`, `US3`, or `US4`
- Include exact file paths in every task description

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the authentication module skeleton, security primitives, shared contracts, and persistence scaffolding required before auth functionality can be implemented.

- [ ] T001 Create the feature package structure under `internal/modules/auth/`, `internal/modules/shared/security/`, and `internal/modules/shared/authorization/` per the approved architecture
- [ ] T002 [P] Initialize the Go module, GORM/PostgreSQL configuration, and environment/test setup in `go.mod`, `internal/platform/db/`, and `internal/platform/config/`
- [ ] T003 [P] Configure migration, linting, and test tooling in `Makefile`, `internal/platform/db/migrations/`, and `.golangci.yml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define the core auth domain, persistence boundaries, transaction coordination, security primitives, and authorization policy that all protected flows depend on.

- [ ] T004 Define the `LoginCredential`, `ActivationInvitation`, `Session`, and `AuthenticationAttempt` aggregates and state invariants in `internal/modules/auth/domain/login_credential.go`, `internal/modules/auth/domain/activation_invitation.go`, `internal/modules/auth/domain/session.go`, and `internal/modules/auth/domain/authentication_attempt.go`
- [ ] T005 [P] Define the persistence models and repository ports for credentials, invitations, sessions, and attempts in `internal/modules/auth/infrastructure/persistence/login_credential_model.go`, `internal/modules/auth/infrastructure/persistence/activation_invitation_model.go`, `internal/modules/auth/infrastructure/persistence/session_model.go`, and `internal/modules/auth/application/ports.go`
- [ ] T006 [P] Add the PostgreSQL migrations for the `login_credentials`, `activation_invitations`, `sessions`, and `authentication_attempts` tables, including unique token-hash indexes, row versioning, expiry fields, and revocation markers in `internal/platform/db/migrations/<timestamp>_create_auth_tables.sql`
- [ ] T007 Implement the shared security primitives for Argon2id hashing, secure token generation, and constant-time comparison in `internal/modules/shared/security/password.go` and `internal/modules/shared/security/tokens.go`
- [ ] T008 Define the auth-user contract, actor/session context, and authorization boundaries consumed by `auth` and `operationauth` in `internal/modules/shared/authorization/actor_context.go` and `internal/modules/auth/application/contracts.go`
- [ ] T009 Extend the existing shared audit envelope only where required for auth lifecycle events, and define `auth` event semantics in `internal/modules/shared/auditing/audit.go` and `internal/modules/auth/application/audit_events.go`
- [ ] T010 [P] Establish the PostgreSQL transaction coordination strategy for account deactivation plus session revocation and for session renewal versus revocation precedence in `internal/modules/shared/transactions/coordination.go` and `internal/modules/auth/application/session_transaction_policy.go`
- [ ] T011 [P] Extend the existing route-level authorization and active-account validation policy to enforce current account status, role checks, and fail-closed behavior before any protected auth endpoint is exposed in `internal/modules/shared/authorization/authorization_policy.go` and `internal/modules/auth/application/authorization_policy.go`
- [ ] T012 [P] Consume and document the existing feature 001 user-owned bootstrap policy and integration contract (`specs/001-user-management/tasks.md` T012) in `internal/modules/auth/application/bootstrap_contract.go` and `internal/modules/auth/application/bootstrap_service.go` without reimplementing user-owned account lifecycle rules
- [ ] T013 [P] Define the auth-owned bootstrap responsibilities for initial administrator setup, idempotency, concurrency safety, and non-public bootstrap coordination with `users` in `internal/modules/auth/application/bootstrap_contract.go`
- [ ] T014 Implement the auth-owned bootstrap service, including transaction-safe, idempotent, concurrency-safe initialization and explicit non-public bootstrap constraints in `internal/modules/auth/application/bootstrap_service.go`
- [ ] T015 Add auth-owned bootstrap audit and failure handling for authorized bootstrap attempts, duplicate/retry rejection, and concurrent bootstrap conflict outcomes in `internal/modules/auth/application/bootstrap_audit.go`
- [ ] T045 [P] Define the auth↔BFF secure server-to-server session issuance/validation contract, including that the login endpoint may return a newly issued bearer token only through this contract and never via logs, URLs, or ordinary API responses, in `internal/modules/auth/application/bff_contract.go` and `internal/modules/auth/interfaces/http/auth_handlers.go`

**Checkpoint**: Auth foundations, security primitives, transaction coordination, authorization policy, and bootstrap ownership are in place before any protected auth endpoint or session flow is implemented.

---

## Phase 3: User Story 1 - Secure sign-in for enterprise users (Priority: P1) 🎯 MVP

**Goal**: Authenticate valid active users with generic denial semantics and abuse controls while issuing secure opaque sessions.

**Independent Test**: A valid active user can authenticate, receive a valid server-managed session, and invalid or inactive requests fail without leaking account existence or privileges.

### Tests for User Story 1

- [ ] T016 [P] [US1] Write failing sign-in, inactive-account, and generic failure tests in `tests/integration/auth_login_test.go`
- [ ] T017 [P] [US1] Write failing password hashing and verification tests in `tests/unit/auth_password_hash_test.go`
- [ ] T018 [P] [US1] Write failing login abuse control and concurrent abuse-threshold tests in `tests/integration/auth_login_abuse_test.go`
- [ ] T046 [P] [US1] Write failing activation invitation lifecycle tests for administrator issuance, expiration, revocation, atomic replacement, and replay rejection in `tests/integration/auth_activation_invitation_lifecycle_test.go`
- [ ] T047 [P] [US1] Write failing concurrent invitation redemption tests proving single-use enforcement under races in `tests/integration/auth_activation_invitation_concurrency_test.go`
- [ ] T048 [P] [US1] Write failing invitation redemption tests for Argon2id password setup, `LoginCredential` activation, and no automatic session issuance in `tests/integration/auth_activation_redemption_test.go`

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement login credential validation, password verification, and provisioning-state checks in `internal/modules/auth/domain/login_credential_rules.go` and `internal/modules/auth/application/login_service.go`
- [ ] T020 [US1] Implement authentication attempt tracking, generic failure classification, and abuse-threshold enforcement in `internal/modules/auth/application/auth_attempt_service.go`
- [ ] T021 [US1] Implement secure session issuance with opaque token generation, hashing, and session persistence in `internal/modules/auth/application/session_service.go`
- [ ] T022 [US1] Add auth sign-in HTTP handler, request validation, and secure session issuance response behavior in `internal/modules/auth/interfaces/http/auth_handlers.go` only after T011 and invitation-activation tasks (T049-T054), allowing bearer token return only through the approved server-to-server BFF contract and never through logs, URLs, or ordinary API responses
- [ ] T023 [US1] Emit mandatory audit events for authentication success and failure in `internal/modules/auth/application/auth_audit_emitter.go`
- [ ] T049 [P] [US1] Implement administrator-issued activation invitation creation with cryptographically secure random token generation and token-hash-only persistence in `internal/modules/auth/application/activation_invitation_issue_service.go`
- [ ] T050 [US1] Implement activation invitation expiration validation, administrator revocation, and explicit revoked-state handling in `internal/modules/auth/application/activation_invitation_revoke_service.go`
- [ ] T051 [US1] Implement atomic invitation replacement (revoke old + issue new) in one transaction in `internal/modules/auth/application/activation_invitation_replace_service.go`
- [ ] T052 [US1] Implement single-use invitation redemption with row-lock/conditional guards for concurrent redemption protection in `internal/modules/auth/application/activation_invitation_redeem_service.go`
- [ ] T053 [US1] Implement invitation redemption password setup using Argon2id, `LoginCredential` activation, and password version initialization in `internal/modules/auth/application/credential_activation_service.go`
- [ ] T054 [US1] Enforce that successful invitation redemption does not auto-create or auto-renew any session in `internal/modules/auth/application/activation_invitation_redeem_service.go` and `internal/modules/auth/application/session_service.go`
- [ ] T055 [US1] Emit invitation lifecycle audit events (`ACTIVATION_INVITATION_ISSUED`, `ACTIVATION_INVITATION_REVOKED`, `ACTIVATION_INVITATION_REPLACED`, `ACTIVATION_INVITATION_REDEEMED`) in `internal/modules/auth/application/activation_invitation_audit_emitter.go`

**Checkpoint**: Invitation provisioning and credential activation are complete, and sign-in works with generic denial semantics, active-user enforcement, and secure session issuance, without exposing raw credentials or tokens.

---

## Phase 4: User Story 2 - Session lifecycle and renewal (Priority: P1)

**Goal**: Enforce lifecycle controls for session validity, sliding renewal, revocation precedence, and logout/logout-all flows.

**Independent Test**: A valid session can renew at threshold while expired or revoked sessions fail, and users can terminate their current or all sessions without resurrecting stale sessions.

### Tests for User Story 2

- [ ] T024 [P] [US2] Write failing session renewal, expiration, revocation, and logout tests in `tests/integration/auth_session_lifecycle_test.go`
- [ ] T025 [P] [US2] Write failing concurrent renewal/revocation race tests in `tests/integration/auth_session_race_test.go`
- [ ] T056 [P] [US2] Write failing authenticated login-password change tests for current-password verification, password policy, password version increment, and forced session invalidation in `tests/integration/auth_password_change_test.go`
- [ ] T057 [P] [US2] Write failing password-change concurrency and revocation-order tests ensuring no revoked session can be reused after a successful change in `tests/integration/auth_password_change_concurrency_test.go`

### Implementation for User Story 2

- [ ] T026 [P] [US2] Implement session validity checks, sliding/absolute expiration logic, and renewal threshold enforcement in `internal/modules/auth/application/session_validation_service.go`
- [ ] T027 [US2] Implement atomic conditional renewal with revocation precedence and row-version protection in `internal/modules/auth/application/session_renewal_service.go`
- [ ] T028 [US2] Implement logout and logout-all flows with immediate session invalidation in `internal/modules/auth/application/session_revoke_service.go`
- [ ] T029 [US2] Add session lifecycle HTTP handlers and middleware for validation and protected request gating in `internal/modules/auth/interfaces/http/session_handlers.go` and `internal/modules/auth/interfaces/http/session_middleware.go`
- [ ] T030 [US2] Emit `SESSION_RENEWED` and `SESSION_REVOKED` audit events with correlation to the session and subject in `internal/modules/auth/application/session_audit_emitter.go`
- [ ] T058 [US2] Implement authenticated login-password change flow that verifies the current login password, enforces password policy, updates Argon2id hash, and increments `password_version` in `internal/modules/auth/application/password_change_service.go`
- [ ] T059 [US2] Integrate password-change inequality validation against operation credentials through a feature 003 contract (without implementing operation credential logic in `auth`) in `internal/modules/auth/application/contracts.go` and `internal/modules/auth/application/password_change_service.go`
- [ ] T060 [US2] Implement transactionally atomic revoke-all-sessions on successful password change, requiring fresh authentication afterward per specification in `internal/modules/auth/application/password_change_service.go` and `internal/modules/auth/application/session_revoke_service.go`
- [ ] T061 [US2] Emit password-change security audit events for both successful and rejected attempts in `internal/modules/auth/application/password_change_audit_emitter.go`

**Checkpoint**: Session validity, renewal, and revocation operate consistently under concurrency without allowing a revoked session to be resurrected.

---

## Phase 5: User Story 3 - Authorized session administration (Priority: P1)

**Goal**: Support authorized admin revocation of other users' sessions and enforce immediate deactivation-driven session invalidation without changing account role or active status.

**Independent Test**: An authorized administrator can revoke another user's session, unauthorized actors are blocked, and deactivation immediately invalidates all associated sessions.

### Tests for User Story 3

- [ ] T031 [P] [US3] Write failing admin session revocation and account deactivation integration tests in `tests/integration/auth_admin_session_revoke_test.go`
- [ ] T032 [P] [US3] Write failing deactivation-and-revocation race tests in `tests/integration/auth_account_deactivation_test.go`
- [ ] T062 [P] [US3] Write failing tests proving administrators cannot revoke their own sessions through the administrative revocation endpoint while user-owned logout/logout-all remain available in `tests/integration/auth_admin_session_revoke_test.go`

### Implementation for User Story 3

- [ ] T033 [P] [US3] Implement authorized session-revocation logic for administrators and role-aware access checks in `internal/modules/auth/application/admin_session_revoke_service.go`
- [ ] T063 [US3] Implement an explicit guard rejecting admin self-session revocation through the administrative endpoint in `internal/modules/auth/application/admin_session_revoke_service.go`
- [ ] T034 [US3] Implement the deactivation-triggered revoke-all-sessions flow using a transactional coordination boundary in `internal/modules/auth/application/account_deactivation_invalidator.go`
- [ ] T035 [US3] Add admin session revocation HTTP handlers and protected route guards in `internal/modules/auth/interfaces/http/admin_session_handlers.go`
- [ ] T036 [US3] Emit `ACCOUNT_DEACTIVATED` and `SESSION_REVOKED` audit events tied to the user and revocation reason in `internal/modules/auth/application/session_audit_emitter.go`

**Checkpoint**: Session administration is enforced without conflating session revocation with account-state changes or role mutation.

---

## Phase 6: User Story 4 - Critical operations require operation-specific authorization (Priority: P1)

**Goal**: Publish the authentication contract consumed by operationauth without duplicating operation-auth business logic or creating new login-session privileges.

**Independent Test**: The auth module exposes only the authenticated-user context, authorization boundary, and current-session validation required by operationauth; the operation-specific confirmation remains owned by feature 003.

### Tests for User Story 4

- [ ] T037 [P] [US4] Write failing contract and security-boundary tests for operationauth integration in `tests/integration/auth_operationauth_contract_test.go`

### Implementation for User Story 4

- [ ] T038 [P] [US4] Define the auth-to-operationauth session context, current-account-status contract, and actor identity interfaces in `internal/modules/auth/application/contracts.go` and `internal/modules/shared/authorization/actor_context.go`
- [ ] T039 [US4] Implement the auth contract service that validates the current session and role before an operationauth authorization check is accepted in `internal/modules/auth/application/operationauth_context_service.go`
- [ ] T040 [US4] Ensure protected routes check the latest active account status and current role before any operation-specific confirmation is accepted in `internal/modules/auth/interfaces/http/session_middleware.go`

**Checkpoint**: Auth exposes the required authoritative identity and session context without implementing operationauth business logic.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final validation, migration review, and explicit cross-feature integration gates for users and operationauth.

- [ ] T041 [P] Review migration ordering, token hashing, expiration constraints, and bootstrap invariants against `spec.md`, `plan.md`, and `data-model.md` in `internal/platform/db/migrations/`
- [ ] T042 [P] Review audit event semantics, secret redaction coverage, and fail-closed request validation across auth routes in `internal/modules/auth/`
- [ ] T043 Validate the feature against the approved modular-monolith architecture and cross-module contracts, ensuring no generic repositories, no circular imports, and no duplicated login/session business rules are introduced
- [ ] T044 Run the auth integration and security regression suites, reserving full end-to-end validation for feature 002 + users + operationauth integration after the required contracts exist

---

## Dependencies & Execution Order

### Cross-feature dependencies

- **Feature 001 (`users`)**:
  - account identity, role, active/inactive status, lifecycle invariants, and account deactivation logic
  - current account status validation contract consumed by auth
  - bootstrap invariant coordination and explicit integration contract for initial administrator bootstrap
  - last-active-admin protection and self-protection rules that auth must respect before issuing or maintaining sessions
- **Feature 003 (`operationauth`)**:
  - operation credential lifecycle, confirmation flow, reset/disable/lockout semantics
  - password-change inequality contract (login password MUST differ from operation credential) consumed by auth without implementing operationauth logic in auth
  - the auth-issued session context and active-account validation contract used by operation-specific authorization
  - operation-specific confirmation checks that remain distinct from login-session validity
- **Do not declare full integration complete** until the required `users` and `operationauth` contracts exist and are verified in the shared PostgreSQL modular monolith.

### User story dependency order

- **User Story 1 (US1)** depends on login credential hashing, activation invitation lifecycle (issue/revoke/replace/redeem), provisioning-state activation, audit, and session issuance foundations and is the MVP.
- **User Story 2 (US2)** depends on the same foundational layer and adds lifecycle controls for renewal and revocation.
- **User Story 3 (US3)** depends on US1 and US2 and adds administrative revocation and deactivation-driven invalidation.
- **User Story 4 (US4)** depends on the auth/session contract and current-account-status validation but does not implement operationauth logic itself.

### Parallel opportunities

- Setup tasks T001-T003 can run in parallel.
- Foundational tasks T004-T015 can be partitioned across domain, persistence, security primitives, audit, transaction coordination, and bootstrap integration work.
- Story-level tests for US1-US4 can be written concurrently once foundational tasks are complete.
- The persistence, security policy, and HTTP handler tasks within each story can proceed in parallel when no direct file conflicts exist.

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational, including route-level authorization and active-account validation before any protected auth endpoint exists
3. Complete Phase 3: User Story 1
4. Validate invitation provisioning, credential activation, sign-in, hash verification, abuse controls, and issued session semantics before implementing lifecycle and admin flows

### Incremental delivery

- After US1, add US2 for session lifecycle, renewal, and logout flows.
- After US2, add US3 for admin revocation and account-deactivation invalidation.
- US4 remains a contract/integration phase that validates the auth boundary consumed by operationauth without reimplementing operationauth logic.
- Run the final polish pass only after all user stories are passing independently, then gate the full end-to-end integration once the required users and operationauth modules are present.

---

## Architectural guardrails for this feature

- `auth` owns login credentials, password hashing and verification, activation invitations, authentication attempts, server-side sessions, renewal/revocation, and auth-related audit events.
- `users` owns account identity, role, active/inactive status, and lifecycle invariants; `auth` must consume these authoritative state contracts rather than duplicating their rules.
- `operationauth` owns operation credentials and critical-operation confirmation; `auth` exposes only the authenticated-user context required for those checks.
- Initial administrator bootstrap is non-public, idempotent, concurrency-safe, transactionally coordinated, and auth-owned, while the users module owns the account lifecycle contract and bootstrap policy only.
- Go issues and validates opaque bearer sessions; the future Next.js BFF owns browser cookie handling (Secure, HttpOnly, host-only) and CSRF protections, and this feature does not implement the frontend.
- No generic repositories, no circular imports, no duplicated user/session business rules, no JWT or refresh-token model, and no raw token or password exposure in logs or API responses.
- A Go interface alone does not guarantee transaction atomicity; all security-critical revocation and renewal decisions must rely on the shared PostgreSQL transaction boundary and explicit DB semantics.
