# Tasks: Critical Operation Authorization

**Input**: Design documents from `/specs/003-critical-operation-authorization/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, and approved cross-feature task boundaries in `/specs/001-user-management/tasks.md` and `/specs/002-authentication-session-management/tasks.md`

**Scope**: This feature covers only the `operationauth` capability: operation credentials, credential lifecycle, critical-operation confirmation, authorization decisions, and operation-specific audit events. `auth` and `users` remain authoritative owners of their respective business domains and are consumed only through explicit contracts.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: `US1`, `US2`, `US3`, `US4`, `US5`, `US6`, or `US7`
- Include exact file paths in every task description

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the operationauth module structure and test/migration scaffolding before lifecycle and authorization logic.

- [ ] T001 Create the operationauth package structure under `internal/modules/operationauth/` with `domain/`, `application/`, `infrastructure/`, and `interfaces/` aligned to the approved architecture
- [ ] T002 [P] Initialize operationauth module wiring and explicit dependency injection entrypoints in `internal/modules/operationauth/wiring.go` and `internal/platform/http/router.go`
- [ ] T003 [P] Configure migration and test scaffolding for operationauth in `internal/platform/db/migrations/`, `tests/integration/operationauth/`, and `tests/unit/operationauth/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Implement security-critical foundations, core state models, contract boundaries, credential-abuse protections, and transaction policies before exposing critical-operation endpoints.

- [ ] T004 Define the `OperationCredential` domain model in `internal/modules/operationauth/domain/operation_credential.go` with constraints quoted verbatim from data-model.md: `status` enum `NOT_CONFIGURED | ACTIVE | TEMPORARILY_LOCKED | RESET_REQUIRED | DISABLED`, `credential_version` required, `failed_attempt_count` required default 0, and `row_version` required
- [ ] T005 [P] Define the `OperationConfirmation` domain model in `internal/modules/operationauth/domain/operation_confirmation.go` with constraints quoted verbatim from data-model.md: `status` enum `PENDING | CONSUMED | REJECTED | EXPIRED`, `operation_identifier` required, `request_fingerprint` required, `resource_version` required, `credential_version` required, and `expires_at` required
- [ ] T006 [P] Define persistence models and repositories (no generic repository) for credentials and confirmations in `internal/modules/operationauth/infrastructure/persistence/operation_credential_model.go`, `internal/modules/operationauth/infrastructure/persistence/operation_confirmation_model.go`, and `internal/modules/operationauth/application/ports.go`
- [ ] T007 [P] Add PostgreSQL migrations for `operation_credentials` and `operation_confirmations` in `internal/platform/db/migrations/<timestamp>_create_operationauth_tables.sql` with unique constraints for replay protection on operation identifier/fingerprint windows and row-version columns for concurrency control
- [ ] T008 Extend shared security primitives as needed for operation-password hashing and comparison in `internal/modules/shared/security/password.go` while keeping Argon2id usage consistent with auth
- [ ] T009 [P] Define auth integration contracts for current session validation and login-password verification in `internal/modules/operationauth/application/contracts.go` and `internal/modules/shared/authorization/actor_context.go` without duplicating auth session logic
- [ ] T010 [P] Define users integration contracts for current account status and role checks in `internal/modules/operationauth/application/contracts.go` and `internal/modules/shared/authorization/authorization_policy.go` without duplicating user lifecycle logic
- [ ] T011 [P] Define bidirectional password-inequality contracts (`operation != login` and `login != operation`) between operationauth and auth in `internal/modules/operationauth/application/contracts.go` and `internal/modules/auth/application/contracts.go`
- [ ] T012 Implement fail-closed authorization preconditions (valid session, active account, current role, required contracts available) in `internal/modules/operationauth/application/authorization_preconditions.go` before any critical-operation endpoint is introduced
- [ ] T013 Implement canonical request-fingerprint strategy and operation identifier normalization in `internal/modules/operationauth/application/request_fingerprint_service.go`
- [ ] T014 Implement transaction coordination policy for atomic confirmation consumption + business execution coupling in `internal/modules/operationauth/application/transaction_policy.go` and `internal/modules/shared/transactions/coordination.go`
- [ ] T015 Define operationauth audit event semantics correlated by `operation_identifier` in `internal/modules/operationauth/application/audit_events.go`, distinguishing credential verification, authorization decision, and business execution outcomes
- [ ] T068 [P] Implement durable, append-only operation audit persistence primitives and correlation storage (including failure-path persistence requirements) in `internal/modules/operationauth/application/operation_audit_service.go` and `internal/modules/shared/auditing/audit.go` before critical-operation execution is enabled
- [ ] T069 [P] Implement operation-password failed-attempt tracking, abuse-threshold enforcement, and `TEMPORARILY_LOCKED` transition in `internal/modules/operationauth/application/credential_lockout_service.go` so critical-operation verification cannot be attempted without limits
- [ ] T070 [P] Write failing pre-endpoint lockout-gate tests proving verification attempts are rate/threshold constrained before confirmation endpoints are made usable in `tests/integration/operationauth/credential_lockout_pre_endpoint_gate_test.go`

**Checkpoint**: Security foundations, credential-abuse protections, durable audit persistence, contract boundaries, and transaction guarantees exist before any operation-confirmation endpoint is usable.

---

## Phase 3: User Story 1 - User establishes an operation password (Priority: P1) 🎯 MVP

**Goal**: Allow initial operation-password provisioning with strict separation from login credentials and explicit identity verification.

**Independent Test**: An eligible user with valid session/account status and fresh login-password verification can provision an operation password; invalid preconditions are denied without secret exposure.

### Tests for User Story 1

- [ ] T016 [P] [US1] Write failing provisioning tests for required valid session, active account, and fresh login-password verification in `tests/integration/operationauth/credential_provisioning_test.go`
- [ ] T017 [P] [US1] Write failing password-inequality tests proving operation password must differ from login password in `tests/integration/operationauth/password_inequality_test.go`
- [ ] T018 [P] [US1] Write failing tests for disabled/reset-required precondition handling and fail-closed behavior in `tests/integration/operationauth/credential_provisioning_security_test.go`

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement initial credential provisioning service in `internal/modules/operationauth/application/provision_operation_credential_service.go` enforcing `NOT_CONFIGURED -> ACTIVE` transition and Argon2id hash persistence
- [ ] T020 [US1] Integrate fresh login-password verification contract calls (auth-owned) in `internal/modules/operationauth/application/provision_operation_credential_service.go` without reimplementing login verification
- [ ] T021 [US1] Implement bidirectional password-inequality verification contract calls in `internal/modules/operationauth/application/password_inequality_policy.go`
- [ ] T022 [US1] Add provisioning HTTP endpoint and request validation in `internal/modules/operationauth/interfaces/http/operation_credential_handlers.go` only after T012 precondition guards are wired
- [ ] T023 [US1] Emit provisioning and credential-verification audit events without secret leakage in `internal/modules/operationauth/application/operation_credential_audit_emitter.go`

**Checkpoint**: Initial provisioning is secure, contract-driven, and independent from login-session lifecycle mutation.

---

## Phase 4: User Story 2 - User changes their operation password (Priority: P1)

**Goal**: Support credential rotation with current operation-password verification while preserving session and login-password separation.

**Independent Test**: A user with ACTIVE credential and valid preconditions can rotate operation password; stale or invalid verification is rejected.

### Tests for User Story 2

- [ ] T024 [P] [US2] Write failing operation-password change tests for current-password verification and credential-version increment in `tests/integration/operationauth/credential_change_test.go`
- [ ] T025 [P] [US2] Write failing inequality-direction tests for login-password interactions during operation-password change in `tests/integration/operationauth/password_inequality_change_test.go`

### Implementation for User Story 2

- [ ] T026 [P] [US2] Implement operation-password change service with current operation-password verification in `internal/modules/operationauth/application/change_operation_credential_service.go`
- [ ] T027 [US2] Implement credential-version increment and invalidation of pending confirmations tied to older versions in `internal/modules/operationauth/application/credential_versioning_service.go`
- [ ] T028 [US2] Integrate bidirectional password-inequality contract checks for credential changes in `internal/modules/operationauth/application/password_inequality_policy.go`
- [ ] T029 [US2] Emit credential-change audit events for accepted and rejected attempts in `internal/modules/operationauth/application/operation_credential_audit_emitter.go`

**Checkpoint**: Credential rotation is secure and version-safe without altering login credential or auth session ownership boundaries.

---

## Phase 5: User Story 3 - User confirms a critical operation (Priority: P1)

**Goal**: Enforce single-use, exact-target, short-lived confirmations with replay prevention and transactional consistency.

**Independent Test**: A valid actor can confirm and execute only the exact operation/resource/version once; stale, replayed, or mismatched attempts fail.

### Tests for User Story 3

- [ ] T030 [P] [US3] Write failing confirmation tests for unique operation identifier, canonical fingerprint matching, and short-lived expiry in `tests/integration/operationauth/operation_confirmation_test.go`
- [ ] T031 [P] [US3] Write failing concurrent confirmation consumption tests proving atomic single-use behavior in `tests/integration/operationauth/operation_confirmation_concurrency_test.go`
- [ ] T032 [P] [US3] Write failing replay and request-fingerprint mismatch tests in `tests/integration/operationauth/operation_confirmation_replay_test.go`
- [ ] T033 [P] [US3] Write failing resource-version mismatch and stale-version rejection tests in `tests/integration/operationauth/operation_confirmation_resource_version_test.go`
- [ ] T034 [P] [US3] Write failing transaction rollback and failed-business-execution coupling tests in `tests/integration/operationauth/operation_execution_transaction_test.go`
- [ ] T071 [P] [US3] Write failing authorization-denial and fail-closed confirmation tests (session invalid, account inactive, role mismatch, unavailable dependency) in `tests/integration/operationauth/operation_confirmation_authorization_failure_test.go`
- [ ] T072 [P] [US3] Write failing tests for confirmation state after rollback, replay behavior after failed execution, and concurrent execution attempts against the same operation identifier in `tests/integration/operationauth/operation_confirmation_failure_replay_test.go`
- [ ] T073 [P] [US3] Write failing indeterminate-outcome handling tests that verify deterministic replay/idempotency handling and required audit persistence for unresolved execution outcomes in `tests/integration/operationauth/operation_execution_indeterminate_outcome_test.go`

### Implementation for User Story 3

- [ ] T035 [P] [US3] Implement confirmation issuance service binding action/resource/request fingerprint/resource version/credential version in `internal/modules/operationauth/application/issue_operation_confirmation_service.go`
- [ ] T036 [US3] Implement atomic single-use confirmation consumption with conditional `PENDING -> CONSUMED` transition in `internal/modules/operationauth/application/consume_operation_confirmation_service.go`
- [ ] T037 [US3] Implement transactional coupling between confirmation consumption and business execution outcome in `internal/modules/operationauth/application/critical_operation_authorization_service.go`, including explicit rollback semantics, confirmation-state handling after rollback/failure, and clear separation between PostgreSQL atomicity and external-system side effects
- [ ] T038 [US3] Implement replay prevention and idempotency checks using operation identifier + fingerprint constraints in `internal/modules/operationauth/application/replay_prevention_service.go`
- [ ] T039 [US3] Add critical-operation confirmation endpoints and middleware wiring in `internal/modules/operationauth/interfaces/http/operation_confirmation_handlers.go` only after foundational security controls are complete, durable audit persistence is implemented (T068), and lockout abuse controls are active (T069/T070)
- [ ] T040 [US3] Emit and durably persist distinct audit events for credential verification, authorization decision, and business execution outcomes correlated by `operation_identifier` in `internal/modules/operationauth/application/operation_confirmation_audit_emitter.go`, including failure and rollback paths so failed operations are never silently lost

**Checkpoint**: Critical operations enforce fresh, exact, single-use confirmation with race-safe transaction semantics and correlated auditing, and endpoints are exposed only after lockout abuse controls and durable audit prerequisites are active.

---

## Phase 6: User Story 4 - Administrator initiates operation-password reset (Priority: P1)

**Goal**: Support administrative reset that invalidates current credential usage without exposing or selecting replacement passwords.

**Independent Test**: Authorized administrators can set `RESET_REQUIRED` and invalidate stale confirmations; unauthorized actors are denied.

### Tests for User Story 4

- [ ] T041 [P] [US4] Write failing administrative reset authorization and outcome tests in `tests/integration/operationauth/admin_reset_test.go`
- [ ] T042 [P] [US4] Write failing tests proving reset invalidates pending confirmations and requires reprovisioning in `tests/integration/operationauth/admin_reset_confirmation_invalidation_test.go`

### Implementation for User Story 4

- [ ] T043 [P] [US4] Implement administrative reset service setting `RESET_REQUIRED` and incrementing `credential_version` in `internal/modules/operationauth/application/admin_reset_operation_credential_service.go`
- [ ] T044 [US4] Implement reset-driven invalidation of pending confirmations in `internal/modules/operationauth/application/credential_versioning_service.go`
- [ ] T045 [US4] Add administrator reset endpoint and authorization guards in `internal/modules/operationauth/interfaces/http/admin_operation_credential_handlers.go`
- [ ] T046 [US4] Emit reset-related security audit events in `internal/modules/operationauth/application/operation_credential_audit_emitter.go`

**Checkpoint**: Administrative reset is secure, auditable, and does not allow credential reuse or admin-managed secret disclosure.

---

## Phase 7: User Story 5 - User establishes a new password after reset (Priority: P1)

**Goal**: Require explicit reprovisioning after reset/reactivation and prevent automatic restoration of operation authorization.

**Independent Test**: A reset-required or reactivated user must complete explicit reprovisioning before operation authorization is available.

### Tests for User Story 5

- [ ] T047 [P] [US5] Write failing reset-required reprovisioning tests including fresh login-password verification and inequality checks in `tests/integration/operationauth/reset_reprovisioning_test.go`
- [ ] T048 [P] [US5] Write failing account deactivation/reactivation continuity tests proving no automatic restoration of operation credential usability in `tests/integration/operationauth/account_reactivation_credential_state_test.go`

### Implementation for User Story 5

- [ ] T049 [P] [US5] Implement reset-required reprovisioning flow to transition `RESET_REQUIRED -> ACTIVE` only after all security preconditions pass in `internal/modules/operationauth/application/reprovision_operation_credential_service.go`
- [ ] T050 [US5] Integrate account deactivation/reactivation contract handling so reactivation never auto-restores restricted credential states in `internal/modules/operationauth/application/account_state_restriction_service.go`
- [ ] T051 [US5] Emit reprovisioning and reactivation-restriction audit events in `internal/modules/operationauth/application/operation_credential_audit_emitter.go`

**Checkpoint**: Reprovisioning is explicit and secure; reactivation does not bypass reset/disable restrictions.

---

## Phase 8: User Story 6 - System temporarily blocks operation authorization after repeated failures (Priority: P1)

**Goal**: Validate and harden lockout recovery, state precedence, and regression behavior after foundational lockout-abuse controls are already in place.

**Independent Test**: Repeated failures trigger `TEMPORARILY_LOCKED`; eligible lockout expiry may restore `ACTIVE` only when no stronger restriction applies.

### Tests for User Story 6

- [ ] T052 [P] [US6] Write failing recovery and lockout-expiry eligibility regression tests in `tests/integration/operationauth/credential_lockout_test.go` (foundational threshold/abuse gate behavior is covered by T070)
- [ ] T053 [P] [US6] Write failing tests proving `RESET_REQUIRED` and `DISABLED` are never auto-cleared by lockout expiry in `tests/integration/operationauth/credential_lockout_restriction_test.go`
- [ ] T054 [P] [US6] Write failing disabled-credential authorization denial tests in `tests/integration/operationauth/credential_disabled_test.go`

### Implementation for User Story 6

- [ ] T055 [P] [US6] Finalize lockout recovery logic and regression protections in `internal/modules/operationauth/application/credential_lockout_service.go` after foundational failed-attempt tracking and threshold enforcement (T069)
- [ ] T056 [US6] Implement eligibility-gated automatic lockout recovery to `ACTIVE` while preserving `RESET_REQUIRED`/`DISABLED` precedence in `internal/modules/operationauth/application/credential_lockout_service.go`
- [ ] T057 [US6] Emit lockout and unlock audit events in `internal/modules/operationauth/application/operation_credential_audit_emitter.go`

**Checkpoint**: Lockout behavior is deterministic, secure, and compatible with higher-priority credential restrictions.

---

## Phase 9: User Story 7 - System audits critical operation attempts and outcomes (Priority: P1)

**Goal**: Persist append-only, correlated, security-grade audit records for credential verification, authorization, and execution outcomes.

**Independent Test**: Every critical-operation attempt produces correctly correlated audit records without exposing passwords, tokens, or secrets.

### Tests for User Story 7

- [ ] T058 [P] [US7] Write failing audit-correlation tests requiring shared `operation_identifier` across verification/authorization/execution outcomes in `tests/integration/operationauth/audit_correlation_test.go`
- [ ] T059 [P] [US7] Write failing tests proving append-only audit behavior and secret redaction in `tests/integration/operationauth/audit_integrity_test.go`
- [ ] T060 [P] [US7] Write failing authorization-denial audit tests for rejected session/account/role/resource conditions in `tests/integration/operationauth/audit_denial_test.go`

### Implementation for User Story 7

- [ ] T061 [P] [US7] Harden and verify correlated audit persistence with immutable envelope semantics (implemented foundationally in T068) in `internal/modules/operationauth/application/operation_audit_service.go` and `internal/modules/shared/auditing/audit.go`
- [ ] T062 [US7] Implement explicit authorization-denial and fail-closed audit emission paths in `internal/modules/operationauth/application/critical_operation_authorization_service.go`
- [ ] T063 [US7] Add audit query/read models needed for operational verification without exposing sensitive fields in `internal/modules/operationauth/infrastructure/persistence/operation_audit_read_model.go`

**Checkpoint**: Audit coverage is complete, correlated, append-only, and safe for security investigations.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Final hardening, consistency checks, and cross-feature integration gates.

- [ ] T064 [P] Validate that operationauth consumes (and does not duplicate) auth session/login-password contracts and users status/role contracts in `internal/modules/operationauth/application/contracts.go`
- [ ] T065 [P] Review migration constraints and concurrency controls for confirmations and credential transitions in `internal/platform/db/migrations/`
- [ ] T066 [P] Run operationauth unit/integration/security regression suites and verify all user-story checkpoints independently in `tests/unit/operationauth/` and `tests/integration/operationauth/`
- [ ] T067 Execute cross-feature integration gate validation plan for auth/users/operationauth contract availability and block full E2E sign-off when required contracts are unavailable in `specs/003-critical-operation-authorization/quickstart.md`
- [ ] T074 Record unresolved execution-semantics decisions (only where not specified by approved spec/plan/research), including handling of indeterminate outcomes and external side-effect reconciliation boundaries, in `specs/003-critical-operation-authorization/quickstart.md` without inventing new product behavior

---

## Dependencies & Execution Order

### Cross-feature dependencies

- **Feature 002 (`auth`)**:
  - current session validation contract
  - login-password verification contract for provisioning/reprovisioning
  - bidirectional password-inequality contract interactions with login-password changes
  - password-change interaction contract required to preserve login/operation password separation
- **Feature 001 (`users`)**:
  - current user account status and role contract
  - account deactivation/reactivation lifecycle signals consumed by operationauth
  - lifecycle restrictions that must remain effective across reactivation
- **Do not declare full integration complete** until required `auth` and `users` contracts exist and are verified in the shared PostgreSQL modular monolith.

### User story dependency order

- **US1** depends on foundational security controls and cross-feature contracts; it is the MVP.
- **US2** depends on US1 credential state foundations and versioning behavior.
- **US3** depends on US1/US2 plus transaction coupling and replay defenses, and also on foundational lockout-abuse controls (T069/T070) and foundational durable audit persistence (T068).
- **US4** depends on credential versioning and authorization policy from earlier phases.
- **US5** depends on US4 reset behavior and account-state integration contracts.
- **US6** depends on credential verification flows and status precedence rules from US1/US5, and focuses on recovery/precedence/regression after foundational abuse controls are already in place.
- **US7** depends on all previous stories to validate complete audit correlation and failure-path durability coverage.

### Parallel opportunities

- Setup tasks T001-T003 can run in parallel.
- Foundational tasks T004-T015 plus T068-T070 can be partitioned across domain modeling, contracts, migrations, and security/audit foundations.
- Within each user story, tasks marked [P] can run in parallel across tests, domain/application services, and persistence boundaries.

---

## Parallel Example: User Story 3

```bash
# Run in parallel after foundational work is complete:
Task: "T030 [US3] confirmation validity tests in tests/integration/operationauth/operation_confirmation_test.go"
Task: "T031 [US3] concurrency single-use tests in tests/integration/operationauth/operation_confirmation_concurrency_test.go"
Task: "T033 [US3] resource-version mismatch tests in tests/integration/operationauth/operation_confirmation_resource_version_test.go"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational security controls and contract boundaries
3. Complete Phase 3: US1 provisioning with fresh login-password verification and password-inequality checks
4. Validate US1 independently before opening critical-operation confirmation endpoints, and keep those endpoints gated on foundational lockout-abuse controls (T069/T070) and durable audit persistence (T068)

### Incremental Delivery

- Add US2 for safe credential rotation and versioning.
- Add US3 for critical-operation confirmation and transactional execution coupling only after foundational abuse-protection and audit prerequisites are complete.
- Add US4 + US5 for reset/reprovisioning and reactivation restriction continuity.
- Add US6 for lockout recovery, precedence, and regression hardening.
- Finish with US7 correlated auditing and final cross-feature gate checks.

---

## Architectural guardrails for this feature

- `operationauth` owns operation credentials, lifecycle transitions, confirmation, and operation-specific audit semantics.
- `auth` remains the sole owner of login credentials, session validity, and login-password verification.
- `users` remains the sole owner of user identity, role, account status, and lifecycle governance.
- Domain logic must remain independent from Gin, GORM, and PostgreSQL-specific concerns.
- Use explicit dependency injection and focused repositories only; no generic repositories and no circular dependencies.
- Fail closed whenever required cross-feature checks are unavailable, and never expose passwords, tokens, or secrets in logs or API responses.
