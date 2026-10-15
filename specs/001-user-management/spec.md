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
- Q: May an administrator change their own role or deactivate their own account? → A: No. Administrators must never change their own role or deactivate their own account, regardless of how many active administrators exist. This restriction is independent of the last-active-administrator safeguard.

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
- **FR-019**: The system MUST prevent an administrator from deactivating or demoting themselves when doing so would leave no active administrator in the system.
- **FR-020**: The system MUST prevent an administrator from changing their own role or deactivating their own account under any circumstance, regardless of how many active administrators remain.
- **FR-021**: The system MUST revoke all active sessions for a deactivated account immediately upon deactivation and MUST deny access to that account until it is reactivated or otherwise re-provisioned.
- **FR-022**: The system MUST allow authenticated administrators to reactivate a deactivated user account, but MUST require that user to authenticate again after reactivation.
- **FR-023**: The system MUST reject any concurrent deactivation or role-change operation that would leave the system without at least one active administrator, even if the operation is otherwise valid in isolation.
- **FR-024**: The system MUST provide a controlled, non-public mechanism for provisioning the initial administrator account, and MUST prevent unrestricted account creation after that initial provisioning step.
- **FR-025**: The system MUST return paginated user list results with a default page size of 20, a maximum page size of 100, and pagination metadata in every response.
- **FR-026**: The system MUST support optional filtering by role and active status, and MUST support searching by name or email for administrator user-list queries.
- **FR-027**: The system MUST sort administrator user-list results by creation timestamp descending, using the unique user identifier as a secondary sort key to ensure deterministic order.
- **FR-028**: The system MUST exclude passwords, password hashes, session tokens, and other authentication secrets from all user-list and user-detail responses, and MUST enforce authorization and visibility rules server-side.
- **FR-029**: The system MUST allow authenticated regular users to access only their own account through the dedicated self-service endpoint and MUST deny access to the administrator listing endpoint.

### Business Rules

- User email values MUST be unique across the system using a case-insensitive comparison, including deactivated records.
- A newly created account MUST default to an active status.
- A deactivated account MUST be retained as an account record but MUST fail authentication.
- Deactivation MUST immediately revoke all active sessions for the affected account.
- Reactivated accounts MUST require fresh authentication; prior sessions must not be restored automatically.
- Password data MUST be excluded from all user-facing responses.
- Password change operations MUST be distinct from profile update operations.
- Administrators MUST not be able to change user roles or account states in a way that leaves the system without any active administrator.
- Administrators MUST never change their own role or deactivate their own account, regardless of the number of active administrators. This restriction is independent of the last-active-administrator safeguard.
- Administrators may update name, email, role, and active status only within the authorized policy; identifier, creation timestamp, and last modification timestamp are system-managed and immutable.
- The system MUST provide a controlled, non-public initial administrator provisioning mechanism and MUST prevent unrestricted account creation after that initial provisioning step.
- The administrator user-listing endpoint MUST be restricted to authenticated administrators, while authenticated regular users may access only their own account through the dedicated self-service endpoint.
- User listing MUST always be paginated, defaulting to 20 results and capping each page at 100; results MUST include pagination metadata and MUST be searchable by name or email and filterable by role and active status.
- Listing results MUST default to creation timestamp descending order with the unique user identifier as a tie-breaker to maintain deterministic ordering.
- Public self-registration MUST be excluded from all supported user-management flows.
- Account status, role, and email are core business data that MUST be protected by authorization rules.

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

## Assumptions

- The organization has a single, authoritative user registry for the application.
- User records are managed within a controlled administrative environment without public self-registration.
- Account lifecycle decisions are governed by the same approval and audit expectations as other enterprise access controls.
- Password recovery, MFA, and external identity integration are out of scope for this version and are addressed separately.
- The business expects a clear distinction between administrative account management and user self-service account access.
