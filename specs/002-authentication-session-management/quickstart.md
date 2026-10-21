# Quickstart: Authentication and Session Management Validation

## Prerequisites

- Active PostgreSQL database and project services running
- An active user account with known login credentials
- Browser or API client able to maintain HTTP-only cookie behavior and session forwarding

## Validation Scenarios

### Scenario 1: Successful login
1. Submit valid email and password credentials.
2. Confirm the response establishes a secure server-managed session.
3. Confirm the session is bound to the correct active user and not exposed as a raw secret in logs or payloads.

Expected result: authentication succeeds and a valid session is created.

### Scenario 2: Inactive or invalid user attempts
1. Attempt authentication with an inactive account.
2. Attempt authentication with invalid credentials for known and unknown email addresses.
3. Confirm failure responses are generic and do not leak email existence.

Expected result: both cases fail without revealing account existence.

### Scenario 3: Session renewal and revocation
1. Authenticate successfully and wait until the session is near sliding expiry.
2. Submit a protected request to trigger transparent renewal.
3. Logout the current session and then all sessions.
4. Confirm expired or revoked sessions are denied for protected operations.

Expected result: valid sessions renew; revoked and expired sessions do not authorize protected access.

### Scenario 4: Deactivation enforcement
1. Authenticate as a user, then deactivate that account as an administrator.
2. Attempt to use the existing session and attempt a fresh login.
3. Confirm immediate denial.

Expected result: deactivation blocks both existing and new authentication attempts.
