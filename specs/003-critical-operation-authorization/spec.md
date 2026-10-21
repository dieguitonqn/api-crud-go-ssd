# Feature Specification: Critical Operation Authorization

**Feature Branch**: `003-critical-operation-authorization`

**Created**: 2026-10-20

**Status**: Draft

**Input**: User description: "Provide an independent operation credential that allows authorized users to explicitly confirm critical actions. The operation password is separate from the login password. This feature must support the complete lifecycle of operation credentials and secure operation-specific authorization. The system must not invent insecure provisioning or recovery methods, must keep the credential lifecycle separate from login sessions, and must remain consistent with the user-management and authentication/security requirements already approved for the project."

## Clarifications

### Session 2026-10-20
- Q: What lifecycle and security rules are mandatory for the independent operation credential? → A: Initial provisioning requires an active account, a valid authenticated login session, re-entry of the current login password, a new compliant operation password, and backend verification of the login password. The new operation password must differ from the current login password, must not be generated or disclosed by an administrator, and activates the credential upon successful provisioning. Voluntary change requires a valid authenticated session, the current operation password, and a compliant new operation password; successful change invalidates outstanding confirmations from the previous credential and does not alter the login password or session state. Administrative reset invalidates the current credential and outstanding confirmations, marks the credential as RESET_REQUIRED, emits an auditable security event, and prohibits administrators from choosing, learning, or retrieving the replacement password. The affected user must authenticate again, re-enter the current login password, and choose a new compliant operation password before the replacement credential becomes active. Lockout after repeated failures is temporary, must not revoke the login session, and must not clear RESET_REQUIRED or DISABLED states. Account deactivation disables operation authorization immediately; reactivation requires fresh authentication and re-verification of the login password before the credential is usable again. Every critical operation must have a valid authenticated session, active account, current permissions, an active usable operation credential, explicit operation-password verification, and exact resource-state validation. Confirmations are bound to the exact operation and resource and must not create or extend sessions, elevate permissions, or expose a reusable privilege window. Operation-password reuse, duplicate requests, and stale confirmations are rejected by operation identifier and current state checks. Audit records distinguish between successful credential verification, successful authorization, and successful business execution, and normal application interfaces cannot modify or delete audit data.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - User establishes an operation password (Priority: P1)
Users must be able to establish their own operation credential through an approved, controlled process before any critical operation can rely on it. This gives users a separate confirmation control for sensitive actions without mixing it with normal sign-in.

**Why this priority**: Critical operation authorization depends on a deliberate, separate credential that can be validated independently of the login session.

**Independent Test**: A user with an active account and valid identity verification can establish an operation password, after which the user can use it for operation-specific confirmation.

**Acceptance Scenarios**:

1. **Given** a user with a valid active account, a valid authenticated login session, and the correct current login password, **When** they complete the operation-password provisioning flow with a compliant new operation password, **Then** the system activates the operation credential and allows future operation confirmations.
2. **Given** a user with a valid active account and a valid authenticated login session, **When** they submit an operation-password provisioning request with an incorrect current login password, **Then** the system rejects the request and does not activate the operation credential.
3. **Given** a user with a valid active account and a valid authenticated login session, **When** they attempt to set the operation password equal to the current login password, **Then** the system rejects the change and keeps the operation credential unconfigured or unchanged.
4. **Given** a new user account, **When** the account is created, **Then** it does not receive an automatically activated operation password.
5. **Given** a user who has not completed initial operation-password provisioning, **When** they attempt a critical operation, **Then** the system rejects the action because operation authorization is unavailable until provisioning is completed.

---

### User Story 2 - User changes their operation password (Priority: P1)
Users need to rotate their operation credential without affecting their login password or session state. This keeps the two controls independent and supports ongoing security hygiene.

**Why this priority**: Password rotation is a core lifecycle control for preventing stale or compromised operation credentials.

**Independent Test**: A user with a valid session and current operation credential can change the operation password, and any prior operation confirmation tied to the old credential is invalidated.

**Acceptance Scenarios**:

1. **Given** an authenticated user with a valid active session and a valid current operation credential, **When** they request an operation-password change with the current operation password and a compliant new password, **Then** the system updates the operation credential and invalidates any outstanding confirmations tied to the previous credential.
2. **Given** a user who attempts to change the operation password without providing the current operation password, **When** they submit the request, **Then** the system rejects the request.
3. **Given** a user who attempts to set the operation password equal to the current login password, **When** they submit the change, **Then** the system rejects the change.
4. **Given** a user who successfully changes the operation password, **When** the login password and session are checked afterward, **Then** the login password remains unchanged and the login session remains valid.

---

### User Story 3 - User confirms a critical operation (Priority: P1)
Users must be able to confirm a specific action with the operation password without creating a new session or extending their login session. The confirmation must be scoped to the exact requested action and target resource.

**Why this priority**: This is the primary purpose of the feature: securing high-risk actions with an explicit second control that does not broaden user privileges.

**Independent Test**: A user with a valid session, current role permissions, and a valid operation password can confirm a single critical operation, while invalid or stale confirmations are denied.

**Acceptance Scenarios**:

1. **Given** an authenticated user with a valid active session, the required permissions, and a valid operation credential, **When** they confirm a critical operation for a specific resource, **Then** the system authorizes only that operation and resource for execution.
2. **Given** a user whose operation credential is missing, disabled, locked, or reset-required, **When** they attempt a critical operation, **Then** the system rejects the action.
3. **Given** a successful operation confirmation, **When** the user attempts to reuse it for a different action or resource, **Then** the system rejects the reuse.
4. **Given** a critical operation whose target record or permission state changed after confirmation, **When** the operation is executed, **Then** the system revalidates the current state and denies execution if the operation is no longer valid.
5. **Given** a successful confirmation for a critical operation, **When** the request is duplicated by retry or concurrent submission using the same operation identifier, **Then** the system rejects the duplicate and the operation executes only once.
6. **Given** a valid confirmation and a valid resource state at confirmation time, **When** the business operation fails after confirmation, **Then** the system records the confirmation as valid but records the business execution failure as a separate audit outcome.

---

### User Story 4 - Administrator initiates operation-password reset (Priority: P1)
Administrators may initiate a reset of another user's operation credential when policy permits. The reset must invalidate the existing credential without exposing it or assigning a new secret to the user in plain text.

**Why this priority**: Reset workflows are essential operational controls for credential recovery without permitting administrators to learn or assign user secrets.

**Independent Test**: An authorized administrator can trigger a reset for another user, and the previous credential is invalidated until the affected user completes a new approved credential setup.

**Acceptance Scenarios**:

1. **Given** an authorized administrator and a user with an existing operation credential, **When** the administrator initiates a reset, **Then** the system invalidates the current credential, invalidates outstanding confirmations, marks the credential as RESET_REQUIRED, and emits an auditable security event.
2. **Given** the previous operation credential after a reset, **When** the user attempts to confirm a critical operation with the old credential, **Then** the system denies the attempt because the old credential is no longer valid.
3. **Given** an unauthorized actor, **When** they attempt a reset for another user's operation credential, **Then** the system denies the request.
4. **Given** a reset request, **When** the user attempts to authorize a critical operation before establishing the new credential, **Then** the system denies the operation.

---

### User Story 5 - User establishes a new password after reset (Priority: P1)
After an authorized reset, the affected user must recover the ability to authorize critical actions through a controlled identity-verification flow. This protects the system from unauthorized credential recovery while preserving account continuity.

**Why this priority**: A reset without a secure re-establishment flow would create an unsafe bypass of the operation-control mechanism.

**Independent Test**: A user with a reset operation credential can complete an approved re-provisioning flow and then perform operation confirmations again only after the new credential becomes active.

**Acceptance Scenarios**:

1. **Given** a user whose operation credential is in reset-required state, **When** they authenticate again, re-enter the current login password, and choose a new compliant operation password, **Then** the system activates the replacement credential and the prior version is no longer usable.
2. **Given** a reset request that does not complete the approved identity and verification step, **When** the user attempts to authorize a critical operation, **Then** the system denies the request.
3. **Given** an administrator attempts to pick or learn the user's replacement credential, **When** the reset flow is processed, **Then** the system rejects the operation because no administrator may assign or inspect the user's operation password.

---

### User Story 6 - System temporarily blocks operation authorization after repeated failures (Priority: P1)
Repeated wrong operation-password attempts must trigger safe throttling or temporary lockout. This protects critical operations from brute force while keeping the user account and login session separate from the lockout.

**Why this priority**: Repeated guesses against a critical control need a protective, auditable response without compromising the rest of the session model.

**Independent Test**: After repeated invalid attempts, the system denies operation authorization for a controlled interval while preserving the user session and login account state.

**Acceptance Scenarios**:

1. **Given** repeated invalid operation-password submissions from the same credential, **When** the configured attempt threshold is reached, **Then** the system places the operation credential in a temporary lockout state.
2. **Given** a temporarily locked operation credential, **When** the user attempts to confirm a critical operation, **Then** the system denies access until the lockout window ends or a recovery action is approved.
3. **Given** the expiry of the configured lockout period, **When** the credential is still eligible for use and the account remains active, **Then** the system MUST automatically restore the credential to ACTIVE without revoking the login session or clearing reset-required or disabled states.
4. **Given** a lockout event, **When** the system records the event, **Then** the audit trail captures the time, actor, outcome, and relevant security context without storing the password.

---

### User Story 7 - System audits critical operation attempts and outcomes (Priority: P1)
Security stakeholders must be able to review attempts to use the operation credential, whether successful or failed, and determine whether a critical action was accepted or blocked. This supports accountability and incident response.

**Why this priority**: Auditability is essential for governance and for proving that critical actions were explicitly confirmed and executed under policy.

**Independent Test**: Every critical operation attempt produces an audit record with the acting user, resource, action, timestamp, outcome, and security context, while never exposing secrets.

**Acceptance Scenarios**:

1. **Given** a successful operation confirmation, **When** the system executes the critical action, **Then** the system records the confirmation and the execution outcome without logging the credential or secret values.
2. **Given** a failed operation confirmation, **When** the system rejects the action, **Then** the audit record captures the failure reason, actor, target, and outcome.
3. **Given** a credential reset, lockout, or unlock event, **When** the event occurs, **Then** the system records the relevant security event without recording the credential contents.
4. **Given** a normal administrative or application action, **When** it tries to modify or delete an existing audit event, **Then** the system rejects the attempt because audit records are immutable to normal application interfaces.

---

### Edge Cases

- What happens when a user changes the operation password while another critical operation is awaiting confirmation?
- How does the system behave when a user attempts repeated invalid confirmations after the credential is reset but before a replacement is activated?
- What happens when an account is deactivated while the operation credential is active, locked, or reset-required, and a critical operation is being confirmed at the same time?
- What happens when a user attempts a critical operation after their login session expires or is revoked?
- What happens when another user or administrator attempts to replay a previously accepted confirmation for a different target or action?
- How does the system prevent duplicate execution when a critical action is submitted twice concurrently, or when a reset and operation execution race each other?
- What happens when the same user resets their operation password while a critical operation is mid-flight?
- What happens when a user changes the login password to match the current operation password, or attempts to set the operation password to match the login password?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST provide a separate operation credential for critical operation authorization that is independent from the normal login password.
- **FR-002**: The system MUST support the conceptual lifecycle states of operation credentials: NOT_CONFIGURED, ACTIVE, TEMPORARILY_LOCKED, RESET_REQUIRED, and DISABLED.
- **FR-003**: The system MUST consider these states conceptual only and MUST NOT prescribe database or storage structures in the specification.
- **FR-004**: New user accounts MUST NOT receive an automatically activated operation password during initial creation.
- **FR-005**: Users MUST establish their own operation password through an approved provisioning process before operation authorization becomes available.
- **FR-006**: The initial provisioning process MUST require an active user account, a valid authenticated login session, re-entry of the current login password, a new compliant operation password, and a successful backend verification of the current login password. The new operation password MUST differ from the current login password and MUST NOT be generated, assigned, or disclosed by an administrator.
- **FR-007**: An administrator MUST NOT choose, assign, view, or learn a user's operation password.
- **FR-008**: Operation authorization MUST remain unavailable until the operation credential has been successfully established and activated.
- **FR-009**: Login and operation passwords MUST be independently managed and MUST NOT be treated as a shared credential pair.
- **FR-010**: Changing the login password MUST NOT automatically change the operation password, and changing the operation password MUST NOT automatically change the login password.
- **FR-011**: The system MUST reject any operation password that is identical to the user's current login password.
- **FR-012**: The system MUST reject any login-password change that would make the login password identical to the current operation password.
- **FR-013**: Password equality checks MUST NOT require reversible storage of either credential.
- **FR-014**: The system MUST never expose either login or operation passwords in API responses or application logs.
- **FR-015**: Operation passwords MUST be stored using a modern, password-specific, salted hashing algorithm suitable for password verification.
- **FR-016**: Plaintext operation passwords MUST never be persisted.
- **FR-017**: Operation-password verification MUST occur in the authoritative Go backend.
- **FR-018**: Operation-password verification MUST use secure comparison and secure error-handling practices, without leaking whether a value is correct beyond the authorized result.
- **FR-019**: Operation-password hashing parameters MUST be configurable and MUST support future upgrades without requiring a redesign of the feature model.
- **FR-020**: A user MUST have an active authenticated session before an operation can be confirmed.
- **FR-021**: A user MUST possess the required role and current permissions for the target action and resource before operation confirmation can authorize the critical action.
- **FR-022**: The operation credential MUST be ACTIVE and usable for the targeted user before confirming a critical operation.
- **FR-023**: The user MUST provide the operation password for each critical operation, even when the user is already authenticated.
- **FR-024**: Successful operation-password verification MUST authorize only the specific operation, action, and target resource being confirmed.
- **FR-025**: Successful operation verification MUST NOT create a new authenticated session.
- **FR-026**: Successful operation verification MUST NOT extend, renew, or otherwise lengthen the current login session.
- **FR-027**: Successful operation verification MUST NOT elevate user roles or grant additional permissions beyond what the user already has under the current authorization model.
- **FR-028**: Each operation confirmation MUST be single-use and MUST NOT be reusable for another operation or another resource.
- **FR-029**: The system MUST revalidate permissions and current resource state before executing a critical operation.
- **FR-030**: Concurrent changes MUST NOT allow a critical operation to execute when it is no longer valid, authorized, or current.
- **FR-031**: Operation confirmation MUST be bound to the exact action and resource being modified.
- **FR-032**: Critical operations MUST NOT execute if the operation confirmation fails, is stale, or cannot be authoritatively validated.
- **FR-033**: The system MUST define and enforce replay protection or equivalent idempotency safeguards for critical operations so the same logical request cannot duplicate business effects. A repeated request using the same operation identifier MUST NOT execute the operation again.
- **FR-034**: A user with an active session may request an operation-password change, but the request MUST include the user's current operation password, a compliant new operation password, and a valid authenticated session. Successful change MUST invalidate any outstanding confirmations tied to the previous credential and MUST NOT change the login password or session state.
- **FR-035**: A successful operation-password change MUST be auditable without recording the passwords themselves.
- **FR-036**: Authorized administrators may initiate a reset for another user's operation credential, but they MUST NOT be able to view, recover, or assign the password.
- **FR-037**: Resetting a user's operation credential MUST invalidate the previous credential and MUST require the user to establish a new compliant credential through a validated re-provisioning flow before it becomes active.
- **FR-038**: A reset request MUST NOT by itself authenticate the user, authorize a critical operation, or create or extend a session.
- **FR-039**: The login-password re-verification required for provisioning and reset is a knowledge-based control and does not constitute independent multi-factor authentication.
- **FR-040**: Repeated incorrect operation-password attempts MUST trigger a temporary lockout according to configurable thresholds, and the lockout MUST NOT automatically invalidate the login session or deactivate the user account.
- **FR-041**: When a lockout period expires, the operation credential MUST automatically return to ACTIVE only when the account remains active, the credential has not been reset or disabled, and no other security restriction prevents its use. RESET_REQUIRED and DISABLED states MUST NOT be automatically cleared by lockout expiration.
- **FR-042**: Locked and reset-required credentials MUST NOT authorize critical operations.
- **FR-043**: Lockout, unlock, reset, and recovery events MUST be auditable without exposing secret values.
- **FR-044**: Deactivating a user account MUST immediately prevent use of the operation password.
- **FR-045**: Reactivating a user account MUST NOT automatically restore operation authorization. The user MUST authenticate again and complete the approved operation-credential reactivation verification before the credential is usable again.
- **FR-046**: If the operation credential was previously reset or disabled, the user MUST complete the corresponding credential lifecycle process before operation authorization becomes available again.
- **FR-047**: Account state and operation-credential state MUST remain distinct concepts, even when the account is reactivated or the credential is reset.
- **FR-048**: The system MUST audit security-relevant events including initial credential activation, password changes, reset requests, credential resets, failed operation confirmations, successful operation confirmations, lockouts, unlocks, credential disablement, and critical operation execution outcomes.
- **FR-049**: Audit records MUST identify the acting user, target resource, requested action, timestamp, outcome, and relevant security context without logging login passwords, operation passwords, session tokens, credential hashes, or recovery secrets.
- **FR-050**: The system MUST distinguish between successful credential verification, successful operation authorization, and successful business operation execution; these events are not equivalent for auditing or incident response.
- **FR-051**: The Next.js BFF MAY collect the operation password through a secure browser interaction, but the Go backend MUST perform authoritative verification.
- **FR-052**: The BFF MUST NOT determine operation authorization by itself and MUST NOT persist the operation password in browser storage.
- **FR-053**: The operation password MUST NOT be retained by the BFF after the verification request and MUST NOT appear in URLs, logs, analytics, or error messages.
- **FR-054**: Network transport MUST protect the operation password from interception while in transit.
- **FR-055**: The system MUST NOT treat possession of the operation password as sufficient authorization without a valid user session and current permissions.
- **FR-056**: Operation passwords are an additional knowledge-based control, not an independent MFA factor.
- **FR-057**: The system MUST preserve the approved separation between login authentication, session lifecycle, and operation-specific authorization, while ensuring each critical operation remains separately authorized and auditable.
- **FR-058**: The feature MUST remain compatible with the approved modular monolith architecture and MUST not bypass the Go backend as the authoritative verifier for security-sensitive decisions.

### Key Entities *(include if feature involves data)*

- **User Account**: The enterprise identity record for a person, including its current active status, role, and associated security credentials.
- **Operation Credential**: The independent, user-managed knowledge-based credential used to confirm critical operations; it has conceptual lifecycle states and is distinct from the login password.
- **Critical Operation**: A sensitive action that requires explicit confirmation, user authorization, current permissions, and resource-level validation before execution.
- **Confirmation Record**: A single-use authorization artifact associated with a particular action and target resource, created only after successful operation-password verification.
- **Audit Event**: A security-relevant record documenting a credential lifecycle change, confirmation attempt, or operation result without recording the secret value itself.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of critical operations require an authorized user session, current permissions, and a valid operation-specific confirmation before execution.
- **SC-002**: 100% of invalid or stale operation confirmations are denied, and no confirmation is accepted for a different action, resource, or time window.
- **SC-003**: 100% of operation-password changes keep the login password and operation password independent, and no change to either credential automatically alters the other.
- **SC-004**: 100% of reset and lockout events require auditable handling without exposing secret values, and the feature records all relevant security events in the audit trail.
- **SC-005**: 95% or more of valid users complete the initial operation-provisioning flow successfully without requiring an insecure shortcut or administrator-held secret.
- **SC-006**: 100% of critical operations are denied when the operation credential is not active, is locked, is disabled, or requires reset before reactivation.
- **SC-007**: 100% of account deactivation and reactivation events respect the requirement that deactivation disables operation authorization immediately and reactivation does not automatically restore the previous authorization state.
- **SC-008**: 100% of operation-password failures are throttled or locked according to policy without invalidating the user’s login session or account state.
- **SC-009**: Every critical operation is executed only when the current permissions and resource state remain valid at execution time, even under concurrent modification.
- **SC-010**: The feature remains compatible with the approved modular monolith architecture and authoritative Go-based verification for all security-sensitive decisions.
- **SC-011**: 100% of temporary operation-password lockouts expire automatically when security eligibility conditions are met, and 0% of RESET_REQUIRED or DISABLED credentials are auto-restored by lockout expiration.

## Assumptions

- Users have an active account and must complete a valid authenticated login session before operation-password setup or reset is allowed.
- The approved identity-verification mechanism for provisioning and reset is re-entry of the current login password as a knowledge-based control; it is not a separate MFA factor and must not be represented as an administrator-held secret or automatic provisioning action.
- Critical operations are business-sensitive actions that must be validated against current permissions and resource state at execution time, even when the user is already authenticated.
- The feature will be introduced in a context where the existing user-management and session-management features already govern user state, identity, account activation, and last-active-administrator protections.
- The system will continue to forbid public self-registration and will not assume any external identity provider or MFA scheme for the initial release.

## Remaining Unresolved Decisions

- The feature specification resolves the product-level lifecycle and security rules for operation credentials, but policy tuning values such as exact operation-password complexity thresholds, temporary lockout duration, and audit retention windows remain configuration decisions for product and security review rather than architectural ambiguities.
- The business classification of "critical operations" remains domain-specific; this specification does not assume a universal list beyond the requirement that each operation must be explicitly confirmed, resource-bound, and revalidated before execution.

## Decision Reconciliation

- This feature preserves the approved separation between login-password lifecycle, session lifecycle, account management, and operation-password lifecycle established in the user-management and authentication/session specs.
- The operation password is not a session-creation mechanism and does not bypass RBAC or role evaluation.
- The feature does not create a reusable elevated-privilege window or any implicit authorization beyond the exact operation and resource being confirmed.
- The feature remains compatible with the modular monolith architecture and the authoritative Go backend for security-sensitive validation.
