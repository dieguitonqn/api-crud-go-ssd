# Quickstart: User Management Validation

## Prerequisites

- Active PostgreSQL database and application environment
- Authenticated administrator session
- At least one active user record for validation

## Validation Scenarios

### Scenario 1: Admin creates an account
1. Authenticate as an administrator.
2. Submit a user creation request with unique email, name, and role.
3. Confirm the account is created as active and visible to administrative listing.
4. Confirm the response excludes all password or credential material.

Expected result: creation succeeds, account is visible, and no secret values are returned.

### Scenario 2: Duplicate email is rejected
1. Create a user with a given email.
2. Attempt to create another user with the same email in different case.
3. Confirm the request fails with conflict or validation error.

Expected result: duplicate emails are rejected across all account records.

### Scenario 3: Deactivation and self-protection
1. Authenticate as administrator A.
2. Deactivate user B.
3. Confirm B cannot authenticate and B's prior sessions are revoked.
4. Attempt to deactivate or demote administrator A.
5. Confirm the system rejects the change.

Expected result: normal users can be deactivated, but admins cannot remove their own access or leave zero active admins.

### Scenario 4: Self-service visibility
1. Authenticate as a standard user.
2. Request the current-user account endpoint.
3. Confirm only that user's own account is returned.

Expected result: regular users cannot enumerate other accounts or access forbidden fields.
