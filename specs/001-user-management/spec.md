# Feature Specification: User Management

**Feature Branch**: `001-user-management`

**Created**: 2026-10-12

**Status**: Draft

**Input**: User description: "Specify the User Management feature for an enterprise REST API. Context: The project follows the approved constitution in .specify/memory/constitution.md. Feature objective: Provide secure administration of user accounts. Functional requirements: 1. Only authenticated administrators can create user accounts. 2. Public self-registration is not supported. 3. Administrators can list registered users with pagination. 4. Administrators can retrieve individual user accounts. 5. Administrators can update permitted user information. 6. Administrators can deactivate user accounts. 7. Authenticated users can retrieve their own account information. 8. Email addresses must be unique. 9. User accounts have two roles: admin and user. 10. Newly created accounts are active by default. 11. Passwords must never be returned in API responses. 12. Password changes are managed separately from general user updates. 13. Deactivated accounts must not be allowed to authenticate. 14. The system must prevent administrators from accidentally removing the last active administrator. User information: - Unique identifier - Email - Name - Role - Active status - Creation timestamp - Last modification timestamp Scope exclusions: - Public registration - Password recovery - Multi-factor authentication - Multi-tenancy - User profiles - External identity providers Requirements: - Define clear user stories. - Define functional requirements. - Define business rules. - Define measurable acceptance criteria. - Include relevant edge cases. - Identify ambiguities requiring clarification. - Avoid implementation details and framework-specific decisions. - Do not generate application code. - Do not create the technical implementation plan. Focus on WHAT the system must do, not HOW it will be built."

## Clarifications

### Session 2026-10-13
- Q: Which user fields can administrators modify, and may they change a user's role? → A: Administrators may update name, email, role (admin/user), and active status; passwords are managed separately; identifier, creation timestamp, and last modification timestamp are system-managed and immutable. Administrators cannot deactivate or demote themselves if doing so would leave the system without an active administrator.
- Q: When a user account is deactivated, what happens to that user's previously authenticated sessions? → A: Existing sessions are immediately revoked so the deactivated user loses access right away.
- Q: How should the system treat duplicate email addresses when the same address is entered with different case or belongs to a deactivated account? → A: Email uniqueness is case-insensitive and applies to both active and deactivated accounts; no duplicate email address may exist across any user record.
- Q: Can deactivated accounts be reactivated, and how should the last-active-administrator safeguard behave during concurrent deactivation or role-change attempts? → A: Deactivated accounts may be reactivated by administrators only; reactivation does not restore prior sessions, and a fresh authentication is required. Concurrent deactivation or role-change attempts must be rejected if they would leave zero active administrators, and that invariant must be enforced before the operation is committed.
- Q: What are the authorization, pagination, filters, sorting, search, and secret-exclusion rules for the administrator user-listing endpoint? → A: The user list endpoint is restricted to authenticated administrators only. Results are always paginated with a default page size of 20 and a maximum page size of 100. Administrators may filter by role (admin/user) and active status (active/inactive), and search by name or email. Results are sorted by creation timestamp descending with the unique user identifier as a secondary sort key for deterministic ordering. Pagination metadata is always included. Non-administrators cannot access the listing endpoint, and authenticated regular users may access only their own account through the dedicated self-service endpoint. All responses exclude passwords, password hashes, session tokens, and other authentication secrets, and authorization and visibility rules are enforced server-side.
- Q: May an administrator change their own role or deactivate their own account? → A: Historical (superseded clarification retained for traceability): this answer remains limited to role/account-state changes and does not govern administrator session-revocation authority defined in the authentication/session specification. Administrators must never change their own role or deactivate their own account, regardless of the number of active administrators. This restriction is independent of the last-active-administrator safeguard.

### Session 2026-10-20
- Q: How should the user-management domain treat an independently managed operation credential while preserving the login-password and account-governance rules? → A: User accounts may carry an independently managed operation credential, but operation-password lifecycle, validation, and reset flows are outside this feature. Creating a user account MUST NOT automatically activate an operation password. Deactivation MUST immediately disable the use of any associated operation credential. Reactivation MUST NOT automatically restore operation authorization. Administrators MUST NOT retrieve, inspect, or use another user's operation password. User-management responses MUST exclude operation credential hashes, reset secrets, and related sensitive material. Login-password changes and operation-password changes remain separate workflows; successful login-password changes revoke all existing user sessions.
- Q: How is initial login-password provisioning governed for administrator-created accounts? → A: Login-password provisioning is separate from administrative account status. Administrators issue secure activation invitations that are single-use and expire. Invitations can be replaced or revoked by authorized administrators, and all invitation lifecycle actions are auditable. Account activation status, role, and profile updates MUST remain independent from invitation/provisioning state transitions.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Admin creates and manages accounts (Priority: P1)
Administrators must be able to create, review, update, and deactivate user accounts in a controlled and secure way. This protects the organization from unauthorized or accidental account changes while maintaining a reliable user directory.

**Why this priority**: This is the core operational capability for enterprise user administration and underpins secure access control for the platform.

**Independent Test**: An administrator can create a new account, confirm the account appears in the list, update the account details, and deactivate it without affecting unrelated users.

**Acceptance Scenarios**:

1. **Given** an authenticated administrator and a valid set of user information, **When** the administrator creates a new account, **Then** the system creates an active account with the provided fields and unique email.
2. **Given** an authenticated administrator, **When** they attempt to create a user using an email already assigned to another account, **Then** the system rejects the request and prevents a duplicate account.
3. **Given** an authenticated administrator, **When** they deactivate a user account, **Then** the account remains recorded but cannot be used to authenticate.
4. **Given** an authenticated administrator, **When** they attempt to change their own role or deactivate their own account, **Then** the system rejects the operation regardless of the number of active administrators.

---

### User Story 2 - Administrator reviews the user directory (Priority: P1)
Administrators need a reliable way to search and review registered accounts with pagination so they can operate an enterprise-scale user list without exposing private details beyond the established access rules.

**Why this priority**: This lets administrators monitor access status, audit account activity, and validate that each user is managed within policy.

**Independent Test**: An administrator can retrieve a paginated list of users and view a single user record without seeing password data.

**Acceptance Scenarios**:

1. **Given** an authenticated administrator, **When** they request the user list with pagination parameters, **Then** the system returns only the requested slice of accounts and the pagination metadata.
2. **Given** an authenticated administrator, **When** they request a single account by identifier, **Then** the system returns the account details without including a password value.
3. **Given** a request for a user record that does not exist, **When** the administrator asks for it, **Then** the system rejects the request with a clear not-found outcome.

---

### User Story 3 - User manages their own account safely (Priority: P1)
Authenticated users must be able to retrieve their own account information while the system prevents disclosure of sensitive account data such as password values and blocks access to other users' information.

**Why this priority**: This supports self-service visibility and reduces operational friction without weakening the authorization model.

**Independent Test**: A signed-in user can view their own account record and cannot retrieve another user's details.

**Acceptance Scenarios**:

1. **Given** an authenticated user, **When** they request their own account information, **Then** the system returns their account details without exposing any password value.
2. **Given** an authenticated user, **When** they attempt to retrieve another user's account record, **Then** the system denies access.
3. **Given** a deactivated user account, **When** the user attempts to authenticate or access protected resources, **Then** the system refuses access.

---

### User Story 4 - System enforces privileged access and account safety (Priority: P1)
The platform must enforce identity verification, role-based permissions, and safeguards that prevent unsafe administrative actions such as disabling the final active administrator or making a forbidden account state change.

**Why this priority**: This is the core security control that protects the administration layer and preserves continuity of system ownership.

**Independent Test**: An unauthenticated or non-administrator actor cannot create, update, or deactivate accounts except as explicitly allowed by their own account access.

**Acceptance Scenarios**:

1. **Given** an unauthenticated user, **When** they attempt to create or manage user accounts, **Then** access is denied.
2. **Given** a non-admin authenticated user, **When** they attempt an administrator-only action, **Then** the system denies access.
3. **Given** only one active administrator remains, **When** an administrator attempts to deactivate or otherwise remove that final active administrator, **Then** the system blocks the operation.
4. **Given** an authenticated administrator, **When** they attempt to change their own role or deactivate their own account, **Then** the system rejects the request even if other active administrators remain available.

---

### Edge Cases

- What happens when a user attempts to create an account using an email that already exists in a different active or inactive state?
- How does the system handle duplicate email values during bulk or concurrent account creation attempts?
- What happens when an administrator attempts to change a user role to a value outside the defined admin and user roles?
- What happens when an administrator attempts to deactivate the final active administrator?
- What happens when an account is deactivated but a request is still made with a valid session or credential set?
- What happens when an account is updated with a password change request in the same operation as a general profile update?
- What happens when an administrator attempts to demote or deactivate themselves while another active administrator exists or does not exist?
- What controlled, non-public mechanism provisions the initial administrator without enabling unrestricted account creation after startup?
- What happens when a user account has an associated operation credential and that account is deactivated, reactivated, or subject to a password change?
- What happens when an administrator attempts to retrieve or use another user's operation credential, even if the user is authorized to manage the account?
- What server-side controls are required to ensure operation-credential state remains independent of the login-password lifecycle and user-profile updates?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST require authentication for all user creation actions and MUST deny anonymous creation attempts.
- **FR-002**: The system MUST allow only authenticated administrators to create user accounts.
- **FR-003**: The system MUST NOT support public self-registration.
- **FR-004**: The system MUST allow only authenticated administrators to list registered users, and MUST deny non-administrator access to the user-listing endpoint.
- **FR-005**: The system MUST allow authenticated administrators to retrieve a specific user account by identifier.
- **FR-006**: The system MUST allow authenticated administrators to update permitted user fields (name, email, role, and active status) while enforcing role-based constraints, immutable system-managed fields, and the minimum active administrator safeguard.
- **FR-007**: The system MUST allow authenticated administrators to deactivate user accounts without destroying the account record.
- **FR-008**: The system MUST allow authenticated users to retrieve their own account information only.
- **FR-009**: The system MUST enforce unique email addresses across all user accounts, regardless of active or deactivated state, using a case-insensitive comparison.
- **FR-010**: The system MUST support exactly two user roles: admin and user.
- **FR-011**: The system MUST create new user accounts in an active state by default.
- **FR-012**: The system MUST never return password values in API responses.
- **FR-013**: The system MUST manage password changes separately from general user information updates.
- **FR-014**: The system MUST prevent deactivated accounts from authenticating successfully.
- **FR-015**: The system MUST prevent administrators from accidentally removing the last active administrator.
- **FR-016**: The system MUST maintain the required user record fields: unique identifier, email, name, role, active status, creation timestamp, and last modification timestamp.
- **FR-017**: The system MUST treat authentication and authorization as separate security checks and MUST require both to succeed for protected account administration actions.
- **FR-018**: The system MUST reject requests that attempt to create, update, or deactivate a user account without valid authorization.
- **FR-019**: The system MUST prevent an administrator from changing their own role or deactivating their own account under any circumstance, regardless of how many active administrators remain.
- **FR-020**: The system MUST revoke all active sessions for a deactivated account immediately upon deactivation and MUST deny access to that account until it is reactivated or otherwise re-provisioned.
- **FR-021**: The system MUST allow authenticated administrators to reactivate a deactivated user account, but MUST require that user to authenticate again after reactivation.
- **FR-022**: The system MUST reject any concurrent deactivation or role-change operation that would leave the system without at least one active administrator, even if the operation is otherwise valid in isolation.
- **FR-023**: The system MUST provide a controlled, non-public mechanism for provisioning the initial administrator account, and MUST prevent unrestricted account creation after that initial provisioning step.
- **FR-024**: The system MUST return paginated user list results with a default page size of 20, a maximum page size of 100, and pagination metadata in every response.
- **FR-025**: The system MUST support optional filtering by role and active status, and MUST support searching by name or email for administrator user-list queries.
- **FR-026**: The system MUST sort administrator user-list results by creation timestamp descending, using the unique user identifier as a secondary sort key to ensure deterministic order.
- **FR-027**: The system MUST exclude passwords, password hashes, session tokens, and other authentication secrets from all user-list and user-detail responses, and MUST enforce authorization and visibility rules server-side.
- **FR-028**: The system MUST allow authenticated regular users to access only their own account through the dedicated self-service endpoint and MUST deny access to the administrator listing endpoint.
- **FR-029**: The system MUST allow user accounts to carry an independently managed operation credential distinct from the login password, without merging the two credential lifecycles.
- **FR-030**: The system MUST NOT automatically activate an operation credential when a new user account is created.
- **FR-031**: The system MUST immediately prevent the use of any associated operation credential when a user account is deactivated.
- **FR-032**: The system MUST NOT automatically restore the ability to authorize critical operations when a user account is reactivated.
- **FR-033**: The operation-password lifecycle, including provisioning, validation, reset, and revocation, belongs to the separate feature specification for critical operation authorization and is out of scope for this user-management feature.
- **FR-034**: The system MUST prevent administrators from retrieving, inspecting, or using another user's operation credential or operation-password material through user-management APIs or administrative account flows.
- **FR-035**: The system MUST exclude operation credential hashes, reset secrets, and any other sensitive operation-password material from all user-management responses and logs.
- **FR-036**: The system MUST keep login-password changes separate from other user-profile updates and MUST reject requests that combine profile changes with password changes in a single general-purpose update flow.
- **FR-037**: A successful login-password change MUST invalidate all previously established sessions for that user and MUST require the user to authenticate again before resuming authenticated access.
- **FR-038**: Login-password changes and operation-password changes MUST remain separate workflows, and the system MUST not change one credential as a side effect of changing the other.
- **FR-039**: Administrator-created user accounts MUST start with login credential provisioning pending and MUST require completion of the approved activation-invitation flow before successful password-based authentication is possible.
- **FR-040**: The system MUST model account administrative status and login-credential provisioning status as separate governance dimensions, and changes in one MUST NOT implicitly mutate the other.
- **FR-041**: Authorized administrators MUST be able to issue activation invitations for pending accounts, and each invitation MUST be single-use and expire automatically after a configured validity window.
- **FR-042**: Authorized administrators MUST be able to replace or revoke outstanding activation invitations, and replaced or revoked invitations MUST become unusable immediately.
- **FR-043**: The system MUST audit invitation issuance, replacement, revocation, successful redemption, and expiration events with actor, target account, timestamp, and outcome.
- **FR-044**: A valid invitation redemption MUST allow the user to provision an initial compliant login password without exposing the password to administrators or normal administrative interfaces.
- **FR-045**: The initial bootstrap administrator provisioning mechanism MUST be secure, non-public, auditable, and constrained to explicit initialization controls that prevent reuse after bootstrap completion.

### Business Rules

- User email values MUST be unique across the system using a case-insensitive comparison, including deactivated records.
- A newly created account MUST default to an active status.
- A deactivated account MUST be retained as an account record but MUST fail authentication.
- Deactivation MUST immediately revoke all active sessions for the affected account.
- Reactivated accounts MUST require fresh authentication; prior sessions must not be restored automatically.
- Password data MUST be excluded from all user-facing responses.
- Password change operations MUST be distinct from profile update operations.
- Login-password changes MUST invalidate all existing sessions for the affected user, and the user MUST authenticate again before continuing with protected operations.
- Operation-password state MUST remain separate from the login-password lifecycle and MUST never be activated, restored, or exposed through the user-profile management domain.
- User accounts MAY have a separately managed operation credential, but that credential MUST be treated as a distinct authorization artifact whose lifecycle belongs to the critical-operation authorization feature rather than the general user-management feature.
- Deactivating an account MUST immediately prevent use of any associated operation credential, and reactivation MUST NOT restore the ability to authorize critical operations automatically.
- Administrators MUST not be able to change user roles or account states in a way that leaves the system without any active administrator.
- Administrators MUST never change their own role or deactivate their own account, regardless of the number of active administrators. This restriction is independent of the last-active-administrator safeguard.
- Administrators may update name, email, role, and active status only within the authorized policy; identifier, creation timestamp, and last modification timestamp are system-managed and immutable.
- The system MUST provide a controlled, non-public initial administrator provisioning mechanism and MUST prevent unrestricted account creation after that initial provisioning step.
- Newly created administrator-managed accounts MUST remain unable to authenticate with password credentials until a valid activation invitation is redeemed and initial password provisioning is completed.
- Activation invitations for initial password provisioning MUST be single-use, expiring, administratively replaceable/revocable, and fully auditable.
- Administrative account status management (active/inactive, role changes) MUST remain independent from login-credential provisioning status and invitation lifecycle state.
- The initial bootstrap administrator provisioning mechanism MUST be secure, explicit, auditable, and non-reusable once completed.
- The administrator user-listing endpoint MUST be restricted to authenticated administrators, while authenticated regular users may access only their own account through the dedicated self-service endpoint.
- User listing MUST always be paginated, defaulting to 20 results and capping each page at 100; results MUST include pagination metadata and MUST be searchable by name or email and filterable by role and active status.
- Listing results MUST default to creation timestamp descending order with the unique user identifier as a tie-breaker to maintain deterministic ordering.
- Public self-registration MUST be excluded from all supported user-management flows.
- Account status, role, and email are core business data that MUST be protected by authorization rules.
- Operation credential hashes, reset secrets, and related sensitive material MUST never be exposed by user-management APIs or administrative responses.

### Key Entities *(include if feature involves data)*

- **User Account**: The authoritative record for a person with a unique identifier, email address, name, role, active status, creation timestamp, and last modification timestamp.
- **Administrator**: An authenticated actor with privileges to manage user accounts subject to authorization rules and safeguards.
- **Session**: The authenticated context for a user that must expire or be revoked when invalid or deactivated.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Administrators can create, review, update, and deactivate accounts without exposing password values in any user-facing response.
- **SC-002**: 100% of account creation and update attempts with duplicate emails are rejected before a new or modified account is persisted, including case-insensitive duplicates and duplicates on deactivated records.
- **SC-003**: All valid, authorized self-service account requests succeed under normal operating conditions, and all unauthorized requests are rejected.
- **SC-004**: The system blocks any action that would result in zero active administrators and records the denial as a failed operation, including self-deactivation and concurrent final-admin changes.
- **SC-005**: Deactivated accounts fail all authentication attempts while remaining identifiable in the user directory for administrative review, and reactivated accounts require a fresh authenticated session.
- **SC-006**: Pagination for the user list returns consistent results across repeated requests for the same dataset within normal operating conditions, with a default page size of 20 and a maximum page size of 100, and includes pagination metadata in each response.
- **SC-007**: All user-management operations enforce the distinction between authentication and authorization, with protected actions denied when either requirement fails.
- **SC-008**: The user-listing endpoint is accessible only to authenticated administrators, while non-administrator authenticated users are denied access and can access only their own account through the self-service endpoint; no password material or authentication secrets are included in any response.
- **SC-009**: The system provides a controlled, non-public mechanism for initial administrator provisioning and prevents unrestricted account creation after that provisioning step.
- **SC-010**: 100% of administrator-created accounts require successful redemption of a valid, single-use, unexpired activation invitation before first password-based authentication succeeds.
- **SC-011**: 100% of revoked, replaced, expired, or previously redeemed activation invitations are rejected for initial login-password provisioning.
- **SC-012**: Invitation issuance, replacement, revocation, expiration, and redemption events are always recorded with actor, target account, timestamp, and outcome.
- **SC-013**: Account active/inactive state changes and login-credential provisioning changes remain independent, and account-state changes alone never produce successful authentication for accounts with pending credential provisioning.

## Assumptions

- The organization has a single, authoritative user registry for the application.
- User records are managed within a controlled administrative environment without public self-registration.
- Account lifecycle decisions are governed by the same approval and audit expectations as other enterprise access controls.
- Password recovery, MFA, and external identity integration are out of scope for this version and are addressed separately.
- The business expects a clear distinction between administrative account management and user self-service account access.
- Initial login-password provisioning for administrator-created accounts is controlled through secure, administrator-governed activation invitations rather than public registration or administrator-known passwords.
