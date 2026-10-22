# Tasks: User Management

**Input**: Design documents from `/specs/001-user-management/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md

**Scope**: This feature only covers the `users` business capability. Login credentials, activation invitations, session lifecycle, and operation-credential processes remain owned by `auth` and `operationauth` and must be integrated only through explicit contracts.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: `US1`, `US2`, `US3`, or `US4`
- Include exact file paths in every task description

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the user-management module skeleton and shared contracts required before feature work can begin.

- [ ] T001 Create the feature package structure under `internal/modules/users/`, `internal/modules/shared/authorization/`, and `internal/modules/shared/auditing/` per the approved architecture
- [ ] T002 [P] Initialize the Go module, GORM/PostgreSQL configuration, and environment/test setup in `go.mod`, `internal/platform/db/`, and `internal/platform/config/`
- [ ] T003 [P] Configure migration, linting, and test tooling in `Makefile`, `internal/platform/db/migrations/`, and `.golangci.yml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define the core user aggregate, repository contracts, migration, shared transaction coordination, and authorization gating requirements that all protected routes and business flows depend on.

- [ ] T004 Define the `User` aggregate, role enum, lifecycle invariants, and secret-exclusion rules in `internal/modules/users/domain/user.go`
- [ ] T005 [P] Define the persistence model and repository ports for user records in `internal/modules/users/infrastructure/persistence/user_model.go` and `internal/modules/users/application/ports.go`
- [ ] T006 [P] Add the PostgreSQL migration for the `users` table, unique case-insensitive email index, row versioning, and lifecycle columns in `internal/platform/db/migrations/<timestamp>_create_users_table.sql`
- [ ] T007 Implement the user lifecycle service and last-active-administrator invariant in `internal/modules/users/application/user_service.go`
- [ ] T008 Define the shared actor-context and deactivation/session-revocation contract consumed by `users` in `internal/modules/shared/authorization/actor_context.go` and `internal/modules/users/application/contracts.go`
- [ ] T009 Define the shared audit envelope and `users` event semantics in `internal/modules/shared/auditing/audit.go` and `internal/modules/users/application/audit_events.go`
- [ ] T010 [P] Establish a shared transaction coordination mechanism for account deactivation and session revocation in `internal/modules/shared/transactions/coordination.go` and `internal/modules/users/application/contracts.go` so atomicity is enforced by the database transaction boundary rather than by a Go interface alone
- [ ] T011 [P] Implement the authoritative route-level authorization policy and current-account-status validation for admin-only access, active-account enforcement, self-protection, and last-active-admin protection in `internal/modules/shared/authorization/authorization_policy.go` and `internal/modules/users/application/authorization_policy.go`; this must be complete before any protected HTTP endpoint is created
- [ ] T012 [P] Define the user-owned bootstrap policy, initialization invariants, and integration contract for initial administrator bootstrap in `internal/modules/users/application/bootstrap_policy.go` and `internal/modules/shared/authorization/bootstrap_contract.go`; keep the bootstrap implementation responsibilities in feature 002 (`auth`) and document the dependency explicitly
- [ ] T013 [Blocked by feature 002] Define the feature 002 dependency for the auth-owned bootstrap implementation and credential-provisioning contract required for initial administrator bootstrap; no auth implementation work is created here in feature 001
- [ ] T014 [Blocked by feature 002] Define the feature 002 dependency for the auth-owned bootstrap audit and security event handling required for bootstrap outcomes; no auth audit implementation work is created here in feature 001

**Checkpoint**: User-management foundational contracts, transaction coordination, authorization gates, and bootstrap invariants are ready before any protected HTTP endpoint is implemented; bootstrap service and credential provisioning remain feature 002 responsibilities.

---

## Phase 3: User Story 1 - Admin creates and manages accounts (Priority: P1) 🎯 MVP

**Goal**: Permit authorized administrators to create, update, and deactivate user accounts while enforcing the last-active-admin invariant and duplicate-email protections.

**Independent Test**: An administrator can create a new account, confirm it appears in the directory, update profile fields, and deactivate it without affecting unrelated users or collapsing the admin set.

### Tests for User Story 1

- [ ] T015 [P] [US1] Write failing integration tests for admin create/update/deactivate flows and mocked auth/session contract behavior in `tests/integration/users_admin_lifecycle_test.go`
- [ ] T016 [P] [US1] Add concurrency tests for last-active-administrator protection, concurrent account creation, and account deactivation/session-revocation races in `tests/integration/users_concurrency_test.go`
- [ ] T017 [P] [US1] Add bootstrap tests for initial administrator bootstrap, concurrent bootstrap attempts, and idempotent safe retry behavior in `tests/integration/users_bootstrap_test.go`

### Implementation for User Story 1

- [ ] T018 [P] [US1] Implement user creation validation for required fields, default active state, and case-insensitive unique-email enforcement in `internal/modules/users/domain/user_validator.go`
- [ ] T019 [P] [US1] Implement the create-user use case and persistence flow in `internal/modules/users/application/create_user.go`
- [ ] T020 [US1] Implement the update-user use case, permitted-field validation, and self-protection checks in `internal/modules/users/application/update_user.go`
- [ ] T021 [US1] Implement transactional deactivation and session-revocation coordination through the auth contract in `internal/modules/users/application/deactivate_user.go`
- [ ] T022 [US1] Add protected admin HTTP handlers, request validation, and response serialization for create, update, and deactivate in `internal/modules/users/interfaces/http/user_handlers.go` only after T011 and T012 are complete
- [ ] T023 [US1] Add audit emission for `USER_CREATED`, `USER_UPDATED`, `USER_DEACTIVATED`, and `LAST_ADMIN_MUTATION_REJECTED` in `internal/modules/users/application/audit_emitter.go`

**Checkpoint**: The core admin account lifecycle is functional, authorization is enforced from the first usable endpoint, and auth/session logic remains outside the `users` module.

---

## Phase 4: User Story 2 - Administrator reviews the user directory (Priority: P1)

**Goal**: Deliver paginated, role-aware, administrator-only directory access with secure field filtering and deterministic ordering.

**Independent Test**: An authenticated administrator can list users with pagination and search filters, and an ordinary authenticated user cannot access the listing endpoint.

### Tests for User Story 2

- [ ] T024 [P] [US2] Write failing HTTP authorization and pagination tests, using mocks/fakes only for isolated contract checks, in `tests/integration/users_directory_test.go`

### Implementation for User Story 2

- [ ] T025 [P] [US2] Implement directory-query validation, filter parsing, and pagination metadata in `internal/modules/users/application/list_users.go`
- [ ] T026 [P] [US2] Implement search/sort behavior by email/name and creation timestamp descending in `internal/modules/users/application/user_query_service.go`
- [ ] T027 [US2] Implement single-user retrieval and not-found handling in `internal/modules/users/application/get_user.go`
- [ ] T028 [US2] Add admin-only HTTP routes and response serialization without exposing secrets in `internal/modules/users/interfaces/http/user_directory_handlers.go`

**Checkpoint**: The user directory and single-record retrieval are independently testable and consistent with the documented access rules.

---

## Phase 5: User Story 3 - User manages their own account safely (Priority: P1)

**Goal**: Allow any authenticated user to read only their own account record while preventing account disclosure and preserving secret exclusions.

**Independent Test**: A signed-in user can retrieve their own record, but cannot access another user's details, and deactivated accounts are denied self-service access.

### Tests for User Story 3

- [ ] T029 [P] [US3] Write failing self-service visibility tests, including unauthorized and inactive account access checks, in `tests/integration/users_self_service_test.go`

### Implementation for User Story 3

- [ ] T030 [P] [US3] Implement identity-bound account retrieval logic in `internal/modules/users/application/get_self_user.go`
- [ ] T031 [US3] Add self-account HTTP endpoint and permission checks in `internal/modules/users/interfaces/http/user_self_handler.go`
- [ ] T032 [US3] Enforce secret redaction and account status checks before allowing self-account reads in `internal/modules/users/application/self_access_policy.go`

**Checkpoint**: Users can safely access only their own record while the system refuses unauthorized or inactive access.

---

## Phase 6: User Story 4 - System enforces privileged access and account safety (Priority: P1)

**Goal**: Enforce role checks, administrator self-protection, zero-admin prevention, reactivation constraints, and cross-feature security regressions without permitting insecure admin operations.

**Independent Test**: Unauthorized actors cannot manage accounts; administrators cannot demote or deactivate themselves; a final active administrator cannot be removed; reactivation requires fresh authentication outside this module.

### Tests for User Story 4

- [ ] T033 [P] [US4] Write failing adversarial authorization, self-protection, and unauthorized-administrative-request tests in `tests/integration/users_authorization_test.go`
- [ ] T034 [P] [US4] Add last-active-admin protection, self-demotion, self-deactivation, and reactivation-without-session-restoration concurrency regressions in `tests/integration/users_security_regression_test.go`

### Implementation for User Story 4

- [ ] T035 [US4] Keep the final authoritative route guard and protection logic aligned with the earlier foundational authorization work in `internal/modules/users/application/authorization_policy.go`; no new first implementation should be introduced here
- [ ] T036 [US4] Implement or finalize self-protection and zero-admin enforcement in `internal/modules/users/application/admin_safety.go`
- [ ] T037 [US4] Implement reactivation policy that requires fresh authentication and does not restore prior sessions in `internal/modules/users/application/reactivation_policy.go`
- [ ] T038 [US4] Add final authorization middleware and route guards for all protected endpoints in `internal/modules/users/interfaces/http/admin_protection_middleware.go`

**Checkpoint**: The system enforces privileged access and safe account lifecycle transitions without permitting insecure admin operations, and the authorization checks remain authoritative even when the full auth feature is not yet integrated. US4 focuses on adversarial, concurrency, and regression coverage rather than initial security enforcement.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final validation, migration review, and consistency checks across the feature, including explicit cross-feature integration gates for auth and operationauth.

- [ ] T039 [P] Review migration ordering, bootstrap constraints, and schema invariants against `spec.md`, `plan.md`, and `data-model.md` in `internal/platform/db/migrations/`
- [ ] T040 [P] Review audit event semantics, secret redaction coverage, and the authoritative authorization boundary across all user endpoints in `internal/modules/users/`
- [ ] T041 Validate the feature against the approved modular-monolith architecture and cross-module contracts, ensuring no temporary login/session logic or operation credential logic is added inside `users`
- [ ] T042 Run the `users` integration and security regression suites, reserving full end-to-end validation for feature 002 and feature 003 integration gates once those modules exist

---

## Dependencies & Execution Order

### Cross-feature dependencies

- **Mocked implementation allowed for isolated `users` work**:
  - the `users` module may use fake or mocked authentication/session contracts while driving domain and HTTP authorization logic
  - this is only for unit/integration tests that do not depend on feature 002 behavior or database-backed session semantics
- **Blocked by feature 002 (`auth`)**:
  - initial login-password provisioning via secure activation invitations
  - initial administrator bootstrap integration with auth-owned password provisioning and the bootstrap audit trail
  - account-level authentication checks and session revocation contract
  - active/inactive account status enforcement for authentication and session validation
  - reactivation flow that requires fresh authentication after a deactivation
  - any end-to-end session invalidation validation or login/session-based acceptance scenario
- **Blocked by feature 003 (`operationauth`)**:
  - operation credential state and reset/disable lockout rules
  - operation-credential deactivation and reactivation restrictions
  - critical-operation authorization checks required for account-level policy enforcement
- **Do not declare end-to-end integration complete** until feature 002 and feature 003 exist and their contracts are verified in the shared PostgreSQL modular monolith.

### User story dependency order

- **User Story 1 (US1)** depends on the foundational schema, policy, transaction, bootstrap invariant, and authorization guard work and is the MVP.
- **User Story 2 (US2)** depends on the foundational layer and can run in parallel with US1 once the core entities, access contracts, and authorization policy exist.
- **User Story 3 (US3)** depends on the same foundational layer and can run in parallel with US2 after the identity-bound access model is in place.
- **User Story 4 (US4)** depends on the foundational authorization work and the completed lifecycle flows; it is reserved for adversarial and concurrency regression coverage rather than first implementation of security checks.

### Parallel opportunities

- Setup tasks T001-T003 can run in parallel.
- Foundational tasks T004-T014 can be partitioned across domain, persistence, migration, transaction coordination, authorization enforcement, and bootstrap contract work.
- Story-level test tasks for US1-US4 can be written concurrently once foundational tasks are complete.
- The data model, persistence, security policy, and HTTP layer tasks within each story can proceed in parallel where there are no direct file-level conflicts.

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational, including T011 authorization guard implementation and T012 bootstrap policy contract definition before any protected route exists
3. Complete Phase 3: User Story 1
4. Validate admin create/update/deactivate flows and last-admin safety before expanding into directory access or self-service tasks

### Incremental delivery

- After US1, add US2 for reviewer access and pagination.
- After US2, add US3 for self-service visibility.
- US4 remains focused on adversarial, concurrency, and regression coverage after the route-level protections are already implemented.
- Run the final polish pass only after all user stories are passing independently.

---

## Architectural guardrails for this feature

- `users` owns only account directory, role, active status, listing/search, lifecycle invariants, and bootstrap coordination contracts that do not provision login credentials.
- Login credential provisioning, activation invitations, and session management are not implemented in this module.
- Initial administrator bootstrap is constrained to a non-public, transactional, idempotent, auth-owned workflow that does not expose a reusable bootstrap endpoint or default credentials.
- Operation credential lifecycle and critical-operation authorization are not implemented in this module.
- Shared infrastructure remains narrow: authorization contracts, transaction coordination, audit envelope, and security primitives only.
- No generic repository pattern, no duplicated domain models, no circular imports, and no direct access to another module's infrastructure implementations.
- A Go interface alone does not guarantee transaction atomicity; the implementation must rely on a shared transaction coordination strategy using the PostgreSQL boundary and explicit DB transaction semantics.
