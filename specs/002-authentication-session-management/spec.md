# Feature Specification: Authentication and Session Management

**Feature Branch**: `002-authentication-session-management`

**Created**: 2026-10-19

**Status**: Draft

**Input**: User description: "Specify the Authentication and Session Management feature for an enterprise application. Project context: The backend is a modular monolith developed in Go. The application will use Next.js as a Backend for Frontend (BFF), while Go remains the authoritative backend for authentication, authorization, and business operations. The specification must follow the approved project constitution. Feature objective: Provide secure authentication and session management for enterprise users. Functional requirements: 1. Users authenticate using email and password. 2. Public user registration is not supported. 3. Only active accounts may authenticate. 4. Successful authentication creates a secure session. 5. Sessions must have a defined expiration policy. 6. Users must be able to terminate their current session. 7. Users must be able to terminate all their active sessions. 8. Administrators must be able to revoke sessions belonging to other users when authorized. 9. Deactivating a user account must immediately revoke all associated sessions. 10. Reactivating an account must not restore previously revoked sessions. 11. Invalid credentials must be rejected without revealing whether the email exists. 12. Authentication attempts must be protected against automated abuse and brute-force attacks. 13. Authentication and authorization must remain separate responsibilities. 14. Session validation must identify the authenticated user and their current authorization context. 15. Expired or revoked sessions must never authorize protected operations. 16. Authentication secrets must never appear in application logs or API responses. 17. The system must support secure authentication through a Next.js Backend for Frontend. 18. The Go backend must remain the authoritative source for session validity and user identity. Scope exclusions: - Public user registration - Password recovery - Multi-factor authentication - Multi-tenancy - External identity providers - Token issuance outside the application-controlled session model Requirements: - Define user stories. - Define functional requirements. - Define business rules. - Define measurable acceptance criteria. - Include relevant security edge cases. - Identify ambiguities requiring clarification. - Avoid implementation details and framework-specific decisions. - Do not generate application code. - Do not create the technical implementation plan. Focus on WHAT the system must do, not HOW it will be built."

## Clarifications

### Session 2026-10-19
- Q: What session model is required for the initial implementation? → A: The system MUST use stateful, server-managed sessions identified by cryptographically secure opaque bearer tokens and MUST NOT use JWT, JWE, access/refresh token pairs, or OAuth flows for the initial implementation.
- Q: What session lifetime and renewal rules are required? → A: Sessions MUST have a sliding expiration of 7 days and an absolute expiration of 30 days from successful authentication, both configurable. Renewal MUST be transparent, throttled, and restricted to active, unrevoked sessions that remain below the absolute expiration limit.
- Q: What are the authoritative components and browser/session transport decisions? → A: Go is the authoritative component for authentication and authorization. The Next.js BFF manages the browser-facing HttpOnly, Secure, host-only cookie and forwards the session credential to Go over the private application network; the BFF MUST NOT bypass Go authorization decisions.
- Q: What deployment and network isolation decisions apply to the initial release? → A: The initial single-host deployment uses Docker Compose with a reverse proxy, Next.js BFF, Go API, and PostgreSQL. Only the reverse proxy exposes public HTTP/HTTPS endpoints; the Go API and PostgreSQL do not publish public ports. Network isolation is required, but it is not a substitute for cryptographic service authentication.
- Q: What requirements must be preserved from user management? → A: The system MUST preserve the existing 001-user-management rules for admin-only user creation, activation/deactivation, immediate session invalidation on deactivation, reactivation requiring fresh authentication, current-role authorization, and last-active-administrator protection.

### Session 2026-10-20
- Q: How must login-password changes and operation-password authorization be separated from session management? → A: Login-password changes are a separate authentication lifecycle and MUST revoke all existing sessions for the affected user. Operation-password verification is a distinct, operation-specific authorization step that does not create or extend a login session, does not elevate roles or permissions, and does not modify the login password. The complete operation-password lifecycle belongs to feature 003, while this feature is limited to the login-session and session-management contract. Initial login-password provisioning is governed by secure activation invitations; password recovery remains a separate, out-of-scope capability.
- Q: How is initial login-password provisioning controlled for administrator-created accounts? → A: Initial login-password provisioning MUST use secure activation invitations that are single-use, time-limited, and redeemable exactly once by the intended account holder. Authorized administrators control invitation issuance, replacement, and revocation, and all invitation lifecycle events MUST be auditable. Account active/inactive state remains separate from login-credential provisioning state.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Secure sign-in for enterprise users (Priority: P1)
Enterprise users must be able to sign in using their email and password only when their account is active and permitted to authenticate. The platform must reject invalid credentials without disclosing account existence and must defend the sign-in flow against automated abuse.

**Why this priority**: Authentication is the entry point to enterprise access and the foundation for every protected operation.

**Independent Test**: A valid active user can sign in and receive a valid session, while inactive users and invalid credentials are rejected without leakage of account existence or privilege.

**Acceptance Scenarios**:

1. **Given** an active user with valid email and password credentials, **When** they authenticate, **Then** the system creates a valid, server-managed session for that user.
2. **Given** a user whose account is inactive, **When** they attempt to authenticate, **Then** the system rejects the attempt.
3. **Given** invalid credentials for a known or unknown email address, **When** the user attempts to authenticate, **Then** the system returns a generic failure and does not reveal whether the email exists.
4. **Given** repeated failed authentication attempts from the same user or source, **When** the configured abuse threshold is reached, **Then** the system denies additional attempts in a controlled manner.

---

### User Story 2 - Session lifecycle and renewal (Priority: P1)
Authenticated users must manage their sessions without unnecessary interruption while the system enforces strict expiration, renewal, and revocation rules. This supports long-running enterprise workflows without permitting stale or invalid sessions to survive.

**Why this priority**: Session lifecycle is the core operational mechanism that balances usability and security for sustained enterprise access.

**Independent Test**: A valid session can continue working through transparent renewal while expired or revoked sessions are denied and users can still log out from the current session or all sessions.

**Acceptance Scenarios**:

1. **Given** an authenticated user with an active session, **When** the session reaches the configured renewal threshold while still valid and below absolute expiration, **Then** the system MUST renew the session transparently without requiring a fresh login.
2. **Given** a user whose session has reached the absolute expiration limit, **When** they make a protected request, **Then** the system rejects the request and requires re-authentication.
3. **Given** a user with multiple active sessions, **When** they log out from all sessions, **Then** all associated sessions are invalidated immediately.
4. **Given** a revoked or expired session, **When** the user attempts to renew it, **Then** the system refuses renewal and requires a fresh authentication flow.

---

### User Story 3 - Authorized session administration (Priority: P1)
Administrators must be able to revoke sessions belonging to other users when policy allows it, while preserving the separate account-state safeguards defined in 001-user-management. Session revocation does not alter the user's account role or activation state.

**Why this priority**: Session control is a security and governance capability that protects enterprise operations while maintaining accountable administrative boundaries.

**Independent Test**: An authorized administrator can revoke a session for another user, including the final active administrator when needed, while unauthorized actors are denied.

**Acceptance Scenarios**:

1. **Given** an authenticated administrator with the required authority, **When** they revoke another user's session, **Then** the session is invalidated immediately.
2. **Given** a non-administrator or unauthorized actor, **When** they attempt to revoke another user's session, **Then** the request is denied.
3. **Given** a user account that is deactivated, **When** the deactivation is processed, **Then** all associated sessions are revoked immediately and remain invalid until the account is reactivated and the user authenticates again.
4. **Given** an authenticated administrator, **When** they attempt to revoke a session for a user whose account role or active status remains valid, **Then** the system revokes only the session and does not change the user's role or active status.
5. **Given** the final active administrator, **When** an administrator attempts to revoke that user's session, **Then** the system permits revocation because session revocation is separate from account-state safeguards and is not blocked by the last-active-administrator rule.

---

### User Story 4 - Critical operations require operation-specific authorization (Priority: P1)
Certain sensitive operations require a dedicated confirmation step distinct from normal login-session validation. The system must separate the login session from the authorization used to confirm a specific critical action without creating or extending an authenticated session.

**Why this priority**: Some enterprise actions require explicit confirmation without granting new session privileges or bypassing the normal authorization model.

**Independent Test**: A valid login session alone does not authorize a critical action unless the operation-specific confirmation step succeeds for that exact operation and resource, and that confirmation does not create new session validity or privileges.

**Acceptance Scenarios**:

1. **Given** a user with a valid active session, **When** they attempt a critical operation that requires operation-specific confirmation, **Then** the system requires the separate authorization step before the operation is accepted.
2. **Given** a valid operation-password confirmation for one operation, **When** the user attempts a different critical operation or a different resource, **Then** the system requires a fresh confirmation for that operation.
3. **Given** a successful operation-password verification, **When** the user continues to use the session normally, **Then** the system does not create or extend the authenticated session, does not grant new roles, and does not widen permissions.
4. **Given** an expired, revoked, or invalid login session, **When** the user attempts a critical operation, **Then** the operation remains denied unless the session is re-established and the operation-specific confirmation is valid.

---

### Security Acceptance Scenarios

1. **Given** multiple concurrent renewal attempts for the same valid session, **When** those requests are processed nearly simultaneously, **Then** the system preserves the session's correct validity window and does not shorten the effective lifetime or re-enable a revoked session.
2. **Given** a renewal in flight while the user logs out, **When** the logout request is processed, **Then** the system rejects the renewal and the session remains invalidated without allowing stale renewal state to restore access.
3. **Given** a renewal in flight while an authorized administrator revokes the same session, **When** both operations are processed concurrently, **Then** the system honors the revocation outcome and does not allow an already revoked session to become valid again.

---

### Edge Cases

- What happens when an attacker presents a valid-looking session token that has been revoked or flagged invalid by the authoritative server?
- What happens when a user attempts to re-use a session after their account is deactivated or after an administrator revokes the session?
- What happens when a session is renewed while another concurrent request is also using it?
- What happens when a renewal attempt arrives after the absolute expiration deadline but before the sliding expiration threshold is reached?
- What happens when a user attempts to authenticate with a correct password after repeated failed attempts have triggered lockout or throttling?
- What happens when the browser submits a cookie-bearing request without protection against request forgery on a state-changing action?
- What happens when a session credential is present in a URL, log line, or error payload due to client or server error handling?
- What happens when the authoritative session state cannot be validated for a protected request?
- What happens when an account is reactivated after sessions were previously revoked?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow users to authenticate using their email address and password.
- **FR-002**: The system MUST NOT support public user registration.
- **FR-003**: The system MUST allow only active accounts to authenticate successfully.
- **FR-004**: The system MUST create a server-managed authenticated session when a user authenticates successfully.
- **FR-005**: The system MUST identify each authenticated session by a cryptographically secure opaque bearer credential.
- **FR-006**: The system MUST use authoritative server-side session state to determine session validity.
- **FR-007**: The system MUST support a sliding expiration of 7 days and an absolute expiration of 30 days from successful authentication, with both values configurable.
- **FR-008**: The system MUST reject any session that has reached its absolute expiration limit, regardless of activity or renewal attempts.
- **FR-009**: The system MUST allow successful authorized activity to extend the sliding expiration subject to the absolute expiration limit.
- **FR-010**: The system MUST renew sessions transparently once a valid session reaches the configured renewal threshold, and MUST throttle renewal attempts to avoid unnecessary persistence writes.
- **FR-011**: The system MUST reject expired or revoked sessions from renewal.
- **FR-012**: The system MUST allow a user to terminate their current session and MUST allow them to terminate all of their active sessions.
- **FR-013**: The system MUST allow an authorized administrator to revoke sessions belonging to other users when policy allows it.
- **FR-014**: The system MUST revoke every active session associated with a user account immediately when that account is deactivated.
- **FR-015**: The system MUST NOT restore previously revoked sessions when a user account is reactivated.
- **FR-016**: The system MUST separate ordinary login-session validation from the operation-specific confirmation required for certain critical actions, and MUST NOT implement a reusable elevated-privilege window.
- **FR-017**: Certain critical operations MUST require an additional authorization step that uses an operation password independent from the login password.
- **FR-018**: Successful operation-password verification MUST NOT create, extend, or renew an authenticated session, MUST NOT elevate user roles, and MUST NOT grant additional permissions beyond the specific operation and resource being confirmed.
- **FR-019**: Every critical operation MUST require its own confirmation, and each confirmation MUST apply only to the specific operation and resource it is validating.
- **FR-020**: The complete operation-password lifecycle, including provisioning, validation, reset, and revocation, belongs to feature 003 and is outside the scope of the session-management specification.
- **FR-021**: The system MUST reject invalid credentials without revealing whether the submitted email address exists in the system.
- **FR-022**: The system MUST protect authentication attempts against automated abuse, brute-force attacks, and credential stuffing.
- **FR-023**: The system MUST keep authentication and authorization as distinct responsibilities and MUST require both to succeed for protected operations.
- **FR-024**: The system MUST validate the active session on every protected request and MUST identify the authenticated user and their current authorization context from authoritative server-side state.
- **FR-025**: The system MUST deny protected operations when a session is expired, revoked, missing, invalid, or otherwise not authoritatively valid.
- **FR-026**: The system MUST reject protected operations for inactive users even when the session itself remains technically present.
- **FR-027**: The system MUST never expose session credentials, user identity, roles, permissions, or other readable claims inside the session token itself.
- **FR-028**: The system MUST never store or log raw session credentials and MUST store only session credential hashes.
- **FR-029**: The system MUST reject requests that depend on client-supplied identity or authorization headers without authoritative server-side validation.
- **FR-030**: The system MUST ensure that authentication does not imply authorization and MUST fail closed when authoritative session state cannot be validated.
- **FR-031**: The system MUST protect the browser-facing session cookie with HttpOnly, Secure, host-only, and appropriate SameSite properties, and MUST prevent JavaScript access to the session credential.
- **FR-032**: The system MUST support a Next.js BFF that manages the browser-facing cookie while the Go backend remains the authoritative source for authentication and authorization.
- **FR-033**: The BFF MUST forward the opaque session credential to Go over the private application network and MUST NOT bypass Go authorization rules.
- **FR-034**: The system MUST protect state-changing browser requests against CSRF while not exposing session credentials in URLs, logs, or error responses.
- **FR-035**: The system MUST preserve the existing administration safeguards from 001-user-management, including admin-only user creation, self-protection against role or account deactivation, and preservation of the last active administrator.
- **FR-036**: The initial deployment MUST use a reverse proxy, Next.js BFF, Go API, and PostgreSQL in a single-host Docker Compose deployment, with only the reverse proxy exposed publicly.
- **FR-037**: The initial deployment MUST keep the Go API and PostgreSQL off public ports and MUST use network segmentation to restrict inter-container communication.
- **FR-038**: The system MUST document that network isolation is not a substitute for cryptographic service authentication and MUST retain the ability to introduce authenticated service-to-service communication in future deployments.
- **FR-039**: The system MUST use a secure password hashing algorithm for password verification and MUST ensure that authentication failure responses are generic and non-disclosing.
- **FR-040**: The system MUST protect against session fixation, secure logout, and safe handling of concurrent session operations without shortening session validity or reactivating revoked sessions.
- **FR-041**: The system MUST behave in a fail-closed manner when authoritative session state cannot be validated and MUST not elevate privileges based on BFF identity alone.
- **FR-042**: A successful login-password change MUST revoke all existing user sessions, require re-authentication with the new password, and keep operation-password changes as a separate workflow from login-password changes.
- **FR-043**: The system MUST not automatically alter the login password when an operation password changes, and the two password workflows MUST remain independent.
- **FR-044**: Authorized administrators may revoke sessions belonging to other users, including other administrators, without modifying the user's account state, role, or active status.
- **FR-045**: Authorized administrators MUST be allowed to revoke sessions for any other user, including users who are currently the last active administrator, because session revocation is independent from account-role and account-status safeguards.
- **FR-046**: The authentication model MUST require initial login-password provisioning through secure activation invitations that are single-use, expiring, and invalid after successful redemption, replacement, or revocation.
- **FR-047**: Authorized administrators MUST control activation-invitation issuance, replacement, and revocation according to policy, and these actions MUST NOT directly alter account active/inactive status or role assignments.
- **FR-048**: The system MUST treat administrative account status and login-credential provisioning status as separate states; successful authentication requires both an active account and completed credential provisioning.
- **FR-049**: Initial bootstrap administrator provisioning MUST occur through a secure, non-public initialization pathway and MUST be auditable.
- **FR-050**: Invitation issuance, replacement, revocation, expiration, and redemption outcomes MUST be captured in the security audit trail with actor, target account, timestamp, and result.

### Business Rules

- Public registration MUST be denied for enterprise access unless a separate, explicitly approved onboarding flow is introduced.
- Only active user accounts MUST be eligible to authenticate.
- Successful authentication MUST result in a stateful session tracked by authoritative server-side state.
- Session validity MUST be determined by the Go backend, not by the browser or BFF.
- Session expiration and session revocation MUST invalidate access without ambiguity.
- User-managed session termination MUST be honored immediately for the targeted session or all active sessions for that user.
- Administrator-initiated session revocation MUST be restricted to authorized roles and policy boundaries.
- Deactivation MUST immediately revoke every active session associated with the user, and reactivation MUST require fresh authentication.
- Authentication failures MUST not reveal whether an email exists in the system.
- Brute-force and automated abuse controls MUST reduce credential-guessing risk and protect the sign-in flow.
- The BFF MUST not override or weaken Go's authentication or authorization decisions.
- Authorization decisions MUST rely on the current user identity and role context derived from authoritative server-side validation.
- The system MUST preserve the 001-user-management rule set for administrator self-protection and final-administrator safety.
- Session renewal MUST remain transparent to the user but MUST not create privilege persistence beyond the configured absolute expiration limit.
- Renewal MUST be throttled to avoid unnecessary persistence writes, must not shorten session validity under concurrent requests, and must not resurrect a revoked session.
- Login-password changes MUST revoke all sessions belonging to the affected user, and operation-password changes MUST remain independent from the login-password workflow.
- Critical operations require an additional operation-specific authorization step that uses an independent password and does not create or extend the authenticated session.
- The initial deployment MUST rely on network isolation for service access control while recognizing that this does not provide cryptographic service authentication.

### Technical Constraints

- The initial session model is stateful and server-managed; JWT, JWE, access/refresh token pairs, and OAuth flows are explicitly excluded from the initial implementation.
- Each session has one opaque bearer credential generated using a cryptographically secure random number generator.
- Session credential hashes are stored; raw session credentials are never persisted or logged.
- The session credential MUST NOT carry identity, role, permission, or other readable claims.
- Sliding expiration is 7 days and absolute expiration is 30 days from successful authentication; both durations MUST be configurable.
- Successful authorized activity may extend the sliding expiration only while the session remains valid and below the absolute expiration limit.
- Renewal is transparent to the user, MUST be throttled to avoid unnecessary database writes, and MUST NOT reactivate expired or revoked sessions.
- The renewal threshold MUST be configurable, and once a valid session reaches that threshold the system MUST perform transparent renewal; concurrent requests MUST NOT shorten session validity, allow stale sessions to remain valid, or re-activate a revoked session.
- Login-credential provisioning for administrator-created accounts MUST rely on secure, single-use, expiring activation invitations that authorized administrators can issue, replace, and revoke with auditability.
- Account administrative state and credential-provisioning state are separate: active status alone is insufficient for authentication until initial credential provisioning is completed.
- Concurrent logout or administrative revocation MUST take precedence over any in-flight renewal and MUST never allow a revoked session to become valid again.
- No automatic 30-minute inactivity logout is permitted in the initial design.
- The Next.js BFF manages the browser cookie with HttpOnly, Secure, and host-only characteristics and uses an appropriate SameSite policy; browser JavaScript MUST not access the credential.
- The reverse proxy is the only public ingress point; the Go API and PostgreSQL are not publicly exposed.
- The initial deployment does not require a BFF API key, mTLS, or OAuth client credentials, because the network boundary is the initial service-access control.
- The implementation MUST fail closed when the authoritative session state cannot be validated.

### Key Entities *(include if feature involves data)*

- **User Account**: A person’s enterprise identity record containing account state, credentials, and authorization profile needed for secure access.
- **Session**: A server-managed authenticated state associated with a user, subject to sliding and absolute expiration, revocation, and renewal rules.
- **Session Credential Hash**: The stored representation of the opaque bearer credential used to validate sessions without storing the raw credential.
- **Authentication Context**: The current user and authorization state derived from authoritative server-side validation for protected operations.
- **Administrator**: An authorized user role with the business permission to revoke sessions for other users within approved policy boundaries.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All valid authentication requests for active user accounts produce a valid server-managed session under normal operating conditions.
- **SC-002**: All invalid credentials are rejected with a generic failure response that does not reveal whether the submitted email address exists.
- **SC-003**: All expired, revoked, missing, or invalid sessions are denied access to protected operations, and no revoked or expired session can be renewed.
- **SC-004**: All user-initiated logout actions succeed for the current session or all active sessions without affecting unrelated user sessions.
- **SC-005**: All account deactivation operations revoke every associated session immediately, and reactivation requires fresh authentication without restoring any prior session validity.
- **SC-006**: All authorized administrator revocations are accepted and all unauthorized revocation attempts are rejected, while self-protection and last-active-administrator safeguards are preserved.
- **SC-007**: All concurrent renewal, logout, and administrative-revocation races resolve without re-activating revoked sessions, shortening valid session lifetime, or creating privilege persistence beyond the configured absolute expiration limit.
- **SC-008**: Authentication secrets never appear in logs, diagnostics, or API responses, while password verification remains secure and fail-closed when authoritative validation is unavailable.
- **SC-009**: The Go backend remains the single authoritative source for authentication, session validity, and authorization decisions, even when the Next.js BFF manages the browser-facing cookie.
- **SC-010**: Every protected operation is rejected unless the active session and the current user authorization context can be validated from authoritative server-side state.
- **SC-011**: The initial deployment keeps only the reverse proxy publicly exposed, with the Go API and PostgreSQL isolated from direct public access and the system retaining the ability to add authenticated service-to-service communication in future deployments.
- **SC-012**: 100% of valid sessions that reach the configured renewal threshold while still below absolute expiration are transparently renewed, and 0% of revoked or expired sessions are renewed.
- **SC-013**: 100% of initial login-password provisioning events require a valid single-use, unexpired activation invitation, and 100% of revoked, replaced, expired, or previously redeemed invitations are rejected.
- **SC-014**: Authorized administrators can revoke sessions for any other user account, including the current last active administrator, without mutating account role or active status.

## Assumptions

- Enterprise users authenticate using a standard email-and-password sign-in model.
- Public self-registration is intentionally excluded from the approved scope for this feature.
- Active account state is the single business gate for successful authentication.
- Session policy is governed by enterprise security settings, but the specification defines the required guarantees rather than exact operational tuning values.
- The Next.js BFF acts as the browser-facing boundary while Go remains the authoritative system for identity and authorization.
- Password recovery, external identity providers, and multi-factor verification are separate capabilities and not part of this feature scope.

## Open Questions / Ambiguities Requiring Clarification

- The exact renewal throttle, abuse rate-limit thresholds, and lockout settings remain configuration decisions that must be refined by product or security stakeholders before deployment.
- The exact set of sensitive operations requiring an operation-specific confirmation remains to be confirmed during planning, but the confirmation must remain a separate authorization step independent from session renewal.
- The detailed security logging and audit retention policy for authentication and session events remains a product or compliance decision.

## Decision Reconciliation

- This specification retains the 001-user-management safeguards for admin-only user creation, self-protection against role or account deactivation, and protection of the last active administrator.
- The session model explicitly excludes JWT, JWE, access/refresh token pairs, and OAuth flows from the initial implementation to align with the approved stateful, server-managed design.
- The specification separates business requirements from technical constraints so that implementation details remain recorded as planning context without weakening the user-facing acceptance criteria.
