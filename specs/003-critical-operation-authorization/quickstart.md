# Quickstart: Critical Operation Authorization Validation

## Prerequisites

- Active PostgreSQL database and project services running
- An active user account with a valid login session
- Access to a critical operation that requires operation-password confirmation

## Validation Scenarios

### Scenario 1: Provision an operation credential
1. Authenticate as a user with an active account.
2. Submit a valid operation-password provisioning request with the current login password and compliant new operation password.
3. Confirm the credential becomes active only after successful validation.

Expected result: operation credential is activated and usable for future confirmations.

### Scenario 2: Reject invalid provisioning
1. Submit operation provisioning with an incorrect current login password.
2. Attempt to set the operation password equal to the login password.
3. Confirm both requests are rejected.

Expected result: the credential remains unconfigured or unchanged.

### Scenario 3: Confirm a critical action
1. Ensure the user has a valid session, required permissions, and active operation credential.
2. Submit confirmation for a specific operation and resource.
3. Confirm the confirmation is bound to that exact operation and resource.

Expected result: the action is authorized only for the exact intended target.

### Scenario 4: Invalidate stale or reused confirmations
1. Create a validation confirmation for one resource.
2. Attempt to reuse it for another action or resource.
3. Confirm the system rejects all stale or duplicate confirmations.

Expected result: confirmation reuse and stale approvals are denied.
