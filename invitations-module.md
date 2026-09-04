# Invitations Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/invitations` (tenant-scoped) and `/api/tenant-invitations` (public)

**Role Requirement:** `Permission.MembersManage` for tenant-scoped operations; anonymous access for public operations

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Invitations](#21-list-invitations)
   - 2.2 [Create Invitation](#22-create-invitation)
   - 2.3 [Revoke Invitation](#23-revoke-invitation)
   - 2.4 [Inspect Invitation (Public)](#24-inspect-invitation-public)
   - 2.5 [Accept Invitation (Public)](#25-accept-invitation-public)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Invitations module enables tenant administrators to invite new members to join their organization. It supports both existing users (who can accept invitations while authenticated) and new users (who must provide a password to create an account during acceptance).

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Tenant-Scoped Invitations** | Invitations belong to a specific tenant. Only users with `Permission.MembersManage` can create, list, and revoke invitations. |
| **Public Acceptance** | Invitation inspection and acceptance are public (anonymous) endpoints accessible via a secure token sent to the invitee's email. |
| **Token-Based Security** | Each invitation generates a cryptographically secure token that is hashed and stored in the database. The plain token is sent via email and never stored. |
| **Email Delivery** | Invitation emails are sent asynchronously via an outbox pattern. The delivery status is tracked separately from the invitation status. |
| **Role Assignment** | Invitations can optionally specify a role. If no role is specified, the new member may receive a default role or no role (tenant-specific behavior). |
| **Expiration** | Invitations expire after a configurable duration (default: 7 days). Expired invitations cannot be accepted. |
| **State Transitions** | Invitations progress through states: Pending → Accepted/Revoked/Expired. Once accepted, revoked, or expired, they cannot be reused. |
| **Duplicate Prevention** | A tenant cannot have multiple pending invitations for the same email address. Creating a new invitation for an email with a pending invitation expires the old one first. |
| **Member Conflict Detection** | Invitations cannot be created for email addresses that are already active members of the tenant. |

### Invitation Lifecycle

1. **Creation**: A tenant admin creates an invitation with an email and optional role. The system generates a secure token, stores the invitation, and queues an email containing the invitation link.
2. **Email Delivery**: The invitation email is sent asynchronously. Delivery status is tracked (Pending, ProviderAccepted, Delivered, Failed, etc.).
3. **Inspection (Optional)**: The invitee clicks the link and the frontend calls the inspect endpoint to preview invitation details (tenant name, role, expiration) without accepting it.
4. **Acceptance**: The invitee accepts the invitation:
   - **Existing user**: If authenticated, the system verifies the email matches and creates an active membership.
   - **New user**: If not authenticated, the system requires a password, creates a new user account, and creates an active membership.
5. **Revocation (Optional)**: The tenant admin can revoke a pending invitation before it is accepted. Revoked invitations cannot be accepted.
6. **Expiration**: After the expiration time (default 7 days), the invitation automatically becomes unusable and returns an "Expired" error on acceptance attempts.

### Important Frontend Implications

- **Two separate API paths**: Tenant admins use `/api/tenants/{tenantId}/invitations` (requires authentication and `Permission.MembersManage`). Invitees use `/api/tenant-invitations` (anonymous, token-based).
- **Antiforgery required for mutations**: Create, revoke, and accept operations require the `X-CSRF-TOKEN` header.
- **Rate limiting**: Create, inspect, and accept endpoints are rate-limited to prevent abuse.
- **Email delivery is asynchronous**: The create endpoint returns immediately with a delivery status of `Pending` or `ProviderAccepted`. Actual delivery happens out-of-band.
- **Token security**: The invitation token is sensitive. It should be transmitted only via secure channels (HTTPS, email) and should not be logged or exposed in URLs (use POST with body, not GET with query params).
- **Authenticated vs. anonymous acceptance**: The frontend must detect whether the user is authenticated when accepting an invitation and conditionally show a password field for new users.

---

## 2. API Endpoints Reference

### 2.1 List Invitations

#### `GET /api/tenants/{tenantId}/invitations`

Retrieves all invitations for the specified tenant, including pending, accepted, revoked, and expired invitations.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.MembersManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
[
  {
    "invitationId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
    "email": "newmember@example.com",
    "roleId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "roleName": "Teacher",
    "createdAtUtc": "2026-08-28T10:30:00Z",
    "expiresAtUtc": "2026-09-04T10:30:00Z",
    "acceptedAtUtc": null,
    "revokedAtUtc": null,
    "status": "Pending",
    "deliveryStatus": "Delivered"
  },
  {
    "invitationId": "b2c3d4e5-6789-01bc-def1-234567890abc",
    "email": "oldmember@example.com",
    "roleId": "8d0f7780-8536-51ef-a55c-3d074g77bf8",
    "roleName": "Admin",
    "createdAtUtc": "2026-08-20T14:00:00Z",
    "expiresAtUtc": "2026-08-27T14:00:00Z",
    "acceptedAtUtc": "2026-08-21T09:15:00Z",
    "revokedAtUtc": null,
    "status": "Accepted",
    "deliveryStatus": "Delivered"
  }
]
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.MembersManage` or tenant context mismatch.

**Important Notes:**
- The response includes invitations in all states (Pending, Accepted, Revoked, Expired).
- `acceptedAtUtc` and `revokedAtUtc` are `null` unless the invitation has been accepted or revoked.
- `roleId` and `roleName` are `null` if no role was specified during creation.
- Results are not explicitly sorted by the backend—the frontend should sort as needed (e.g., by `createdAtUtc` descending or by `status`).

---

### 2.2 Create Invitation

#### `POST /api/tenants/{tenantId}/invitations`

Creates a new tenant invitation and sends an invitation email to the specified address.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.MembersManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Rate Limiting:** Applied

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body:**

```json
{
  "email": "newmember@example.com",
  "roleId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | `string` | Yes | Valid email address (max 256 characters). Annotated with `[EmailAddress]` validation. |
| `roleId` | `Guid?` | No | Optional role identifier. Must be a valid role within the tenant. |

**Response:** `201 Created`

```json
{
  "invitationId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "expiresAtUtc": "2026-09-04T10:30:00Z",
  "deliveryStatus": "ProviderAccepted"
}
```

**Location Header:** `/api/tenants/{tenantId}/invitations`

**Possible Errors:**
- `400 Bad Request` — Invalid email format or validation failure.
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.MembersManage` or tenant context mismatch.
- `409 Conflict` — One of:
  - `Invitations.MemberAlreadyExists`: The email is already an active member of the tenant.
  - `Invitations.PendingInvitationExists`: A pending invitation already exists for this email (rare—usually the old one is expired first).
  - `Invitations.InvalidRole`: The specified `roleId` does not exist or is not valid for this tenant.
- `429 Too Many Requests` — Rate limit exceeded.

**Important Notes:**
- **Automatic expiration of old pending invitations**: If a pending invitation exists for the same email, it is automatically expired before creating the new one. However, if a race condition occurs, a `PendingInvitationExists` error may be returned.
- **Email delivery is asynchronous**: The `deliveryStatus` in the response is an immediate snapshot. Possible values at creation time are typically `Pending` or `ProviderAccepted`. The actual delivery (or failure) happens later and is not reflected in this response.
- **Default expiration**: Invitations expire 7 days after creation (configurable on the backend).
- **Token is NOT returned**: The secure invitation token is sent via email only. The API does not return the token to the creator.

---

### 2.3 Revoke Invitation

#### `POST /api/tenants/{tenantId}/invitations/{invitationId}/revoke`

Revokes a pending invitation, preventing it from being accepted. Revoked invitations cannot be un-revoked.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.MembersManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `invitationId` | `Guid` | The invitation identifier. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body:** None

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Antiforgery validation failed.
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.MembersManage` or tenant context mismatch.
- `404 Not Found` — The invitation does not exist or does not belong to the specified tenant.
- `409 Conflict` — The invitation has already been accepted, revoked, or expired (cannot revoke a non-pending invitation).

**Important Notes:**
- Revoking an invitation does not delete it—it sets the status to `Revoked` and records the `revokedAtUtc` timestamp.
- Attempting to accept a revoked invitation returns `Invitations.Revoked` (409 Conflict).

---

### 2.4 Inspect Invitation (Public)

#### `POST /api/tenant-invitations/inspect`

Retrieves invitation details (tenant name, email, role, expiration, status) without accepting it. This endpoint is anonymous and uses the invitation token for authentication.

**Authentication:** None (anonymous)

**Authorization:** None (token-based)

**Rate Limiting:** Applied

**Request Body:**

```json
{
  "token": "CfDJ8Gz...protectedTokenString..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | `string` | Yes | The secure invitation token sent via email. |

**Response:** `200 OK`

```json
{
  "tenantName": "My School",
  "email": "newmember@example.com",
  "roleName": "Teacher",
  "expiresAtUtc": "2026-09-04T10:30:00Z",
  "status": "Pending"
}
```

**Possible Errors:**
- `400 Bad Request` — Token is missing or malformed.
- `404 Not Found` — The invitation does not exist (invalid token or already deleted).
- `429 Too Many Requests` — Rate limit exceeded.

**Important Notes:**
- `roleName` is `null` if no role was specified during invitation creation.
- The `status` field reflects the current state: `Pending`, `Accepted`, `Revoked`, or `Expired`.
- This endpoint does NOT mutate the invitation state—it is safe to call multiple times.
- Use this endpoint to preview the invitation before showing the acceptance UI.

---

### 2.5 Accept Invitation (Public)

#### `POST /api/tenant-invitations/accept`

Accepts a tenant invitation. If the current user is authenticated, it adds them as a member. If not authenticated, it creates a new user account (requires a password) and adds them as a member.

**Authentication:** Optional (supports both authenticated and anonymous users)

**Authorization:** None (token-based)

**Rate Limiting:** Applied

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token

**Request Body (existing user, authenticated):**

```json
{
  "token": "CfDJ8Gz...protectedTokenString..."
}
```

**Request Body (new user, not authenticated):**

```json
{
  "token": "CfDJ8Gz...protectedTokenString...",
  "password": "SecurePassword123!"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | `string` | Yes | The secure invitation token sent via email. |
| `password` | `string` | Conditional | Required if the user is NOT authenticated. Used to create a new user account. |

**Response:** `200 OK`

```json
{
  "tenantId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "newmember@example.com",
  "isNewUser": true
}
```

**Possible Errors:**
- `400 Bad Request` — One of:
  - Token is missing.
  - Password is required but not provided (for unauthenticated users).
  - Password validation failed (too weak, etc.).
- `404 Not Found` — The invitation does not exist (invalid token).
- `409 Conflict` — One of:
  - `Invitations.Expired`: The invitation has expired.
  - `Invitations.Revoked`: The invitation has been revoked.
  - `Invitations.AlreadyAccepted`: The invitation has already been accepted.
  - `Invitations.EmailMismatch`: The authenticated user's email does not match the invitation email.
  - `Invitations.MemberAlreadyExists`: The user is already an active member of the tenant.
  - `Invitations.TenantInactive`: The tenant is not active (suspended, expired, or closed).
- `429 Too Many Requests` — Rate limit exceeded.

**Important Notes:**
- **Authenticated path**: If the user is logged in, the backend verifies that the authenticated user's email matches the invitation email. If it matches, a membership is created and `isNewUser` is `false`. The `password` field is ignored.
- **Anonymous path**: If the user is not authenticated, the `password` field is required. The backend creates a new user account with the invitation email, then creates the membership. `isNewUser` is `true`.
- **Atomic operation**: User creation (if needed) and membership creation happen in a single transaction. If any part fails, the entire operation is rolled back.
- **Invitation is marked accepted**: The invitation's `status` becomes `Accepted` and `acceptedAtUtc` is recorded.
- **Role assignment**: If the invitation specified a `roleId`, the new membership is assigned that role. An audit event is created for the role assignment.
- **After acceptance**: The user should log in (if newly created) or refresh their session to see the new tenant in their memberships list.

---

## 3. Request & Response DTOs

### CreateTenantInvitationRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `email` | `string` | Yes | Valid email address. Max 256 characters. Annotated with `[Required]`, `[EmailAddress]`, `[StringLength(256)]`. |
| `roleId` | `Guid?` | No | Optional. Must be a valid role within the tenant if provided. |

### CreateTenantInvitationResponse

| Field | Type | Description |
|-------|------|-------------|
| `invitationId` | `Guid` | The unique identifier of the created invitation. |
| `expiresAtUtc` | `DateTimeOffset` | The UTC timestamp when the invitation expires (default: 7 days from creation). |
| `deliveryStatus` | `string` | The email delivery status at the time of creation (e.g., `"Pending"`, `"ProviderAccepted"`). See [EmailDeliveryStatus](#emaildeliverystatus). |

### TenantInvitationResponse

| Field | Type | Description |
|-------|------|-------------|
| `invitationId` | `Guid` | The unique identifier of the invitation. |
| `email` | `string` | The invitee's email address. |
| `roleId` | `Guid?` | The role identifier, or `null` if no role was specified. |
| `roleName` | `string?` | The role display name, or `null` if no role was specified. |
| `createdAtUtc` | `DateTimeOffset` | The UTC timestamp when the invitation was created. |
| `expiresAtUtc` | `DateTimeOffset` | The UTC timestamp when the invitation expires. |
| `acceptedAtUtc` | `DateTimeOffset?` | The UTC timestamp when the invitation was accepted, or `null` if not yet accepted. |
| `revokedAtUtc` | `DateTimeOffset?` | The UTC timestamp when the invitation was revoked, or `null` if not revoked. |
| `status` | `string` | The invitation status (see [TenantInvitationStatus](#tenantinvitationstatus)). |
| `deliveryStatus` | `string` | The email delivery status (see [EmailDeliveryStatus](#emaildeliverystatus)). |

### InspectTenantInvitationRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `token` | `string` | Yes | Non-empty string. The secure invitation token. |

### TenantInvitationInspectionResponse

| Field | Type | Description |
|-------|------|-------------|
| `tenantName` | `string` | The name of the tenant organization. |
| `email` | `string` | The invitee's email address. |
| `roleName` | `string?` | The role display name, or `null` if no role was specified. |
| `expiresAtUtc` | `DateTimeOffset` | The UTC timestamp when the invitation expires. |
| `status` | `string` | The invitation status (see [TenantInvitationStatus](#tenantinvitationstatus)). |

### AcceptTenantInvitationRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `token` | `string` | Yes | Non-empty string. The secure invitation token. |
| `password` | `string?` | Conditional | Required if the user is not authenticated. Used to create a new user account. ASP.NET Core Identity enforces password complexity rules. |

### AcceptTenantInvitationResponse

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `userId` | `Guid` | The user identifier (newly created if `isNewUser` is `true`, otherwise the authenticated user's ID). |
| `email` | `string` | The user's email address. |
| `isNewUser` | `bool` | `true` if a new user account was created; `false` if an existing authenticated user accepted the invitation. |

---

## 4. Enum Reference

### TenantInvitationStatus

Enum values are serialized as **strings** (e.g., `"Pending"`, `"Accepted"`).

| Value | Description |
|-------|-------------|
| `Pending` | Invitation has been created and is awaiting acceptance (value: 1). |
| `Accepted` | Invitation has been accepted and the membership has been created (value: 2). |
| `Revoked` | Invitation has been revoked by a tenant admin (value: 3). |
| `Expired` | Invitation has passed its expiration time (value: 4). |

### EmailDeliveryStatus

Enum values are serialized as **strings** (e.g., `"Pending"`, `"Delivered"`).

| Value | Description |
|-------|-------------|
| `Pending` | Email is queued and has not yet been sent (value: 1). |
| `Processing` | Email is currently being processed for delivery (value: 2). |
| `ProviderAccepted` | Email has been accepted by the email provider (e.g., Brevo) for delivery (value: 3). |
| `Failed` | Email delivery failed permanently (value: 4). |
| `Delivered` | Email was successfully delivered to the recipient's mailbox (value: 5). |
| `Deferred` | Email delivery was temporarily deferred by the recipient's mail server (value: 6). |
| `SoftBounce` | Email delivery failed temporarily (e.g., mailbox full) (value: 7). |
| `HardBounce` | Email delivery failed permanently (e.g., invalid email address) (value: 8). |
| `Blocked` | Email was blocked by the recipient's mail server or spam filter (value: 9). |

---

## 5. Validation & Business Rules

### Email Validation

**Backend-enforced:**
- Must be a valid email format.
- Maximum length: 256 characters.
- Trimmed automatically.
- Case-insensitive comparison for duplicate detection (normalized to uppercase).

**Error returned:** `Invitations.InvalidEmail` (400 Bad Request)

### Role Validation

**Backend-enforced:**
- If `roleId` is provided, it must be a valid role within the specified tenant.
- If `roleId` is `null`, the backend may assign a default role or no role (tenant-specific behavior not explicitly defined in the API layer).

**Error returned:** `Invitations.InvalidRole` (400 Bad Request)

### Duplicate Prevention

**Backend-enforced:**
- A tenant cannot create an invitation for an email that is already an active member.
- If a pending invitation exists for the same email, it is automatically expired when a new invitation is created. However, race conditions may cause a `PendingInvitationExists` error.

**Errors returned:**
- `Invitations.MemberAlreadyExists` (409 Conflict)
- `Invitations.PendingInvitationExists` (409 Conflict, rare)

### Invitation Lifecycle Rules

**Backend-enforced:**
- **Pending invitations** can be revoked or accepted. They automatically transition to `Expired` after the expiration time.
- **Accepted invitations** cannot be revoked or re-accepted.
- **Revoked invitations** cannot be accepted or un-revoked.
- **Expired invitations** cannot be accepted.

**Errors returned:**
- `Invitations.Expired` (409 Conflict)
- `Invitations.Revoked` (409 Conflict)
- `Invitations.AlreadyAccepted` (409 Conflict)

### Acceptance Rules

**Backend-enforced:**
- **Authenticated users**: The authenticated user's email must match the invitation email (case-insensitive). If they are already a member of the tenant, acceptance fails.
- **Unauthenticated users**: A password is required. The system creates a new user account with the invitation email, then creates the membership.
- **Email mismatch**: If an authenticated user's email does not match the invitation email, acceptance fails with `Invitations.EmailMismatch`.
- **Tenant inactivity**: Invitations to inactive tenants (suspended, expired, closed) cannot be accepted (though this check is not explicitly shown in the provided code—assume it is enforced).

**Errors returned:**
- `Invitations.PasswordRequired` (400 Bad Request)
- `Invitations.EmailMismatch` (409 Conflict)
- `Invitations.MemberAlreadyExists` (409 Conflict)
- `Invitations.TenantInactive` (409 Conflict, if enforced)

### Token Security

**Backend-enforced:**
- Invitation tokens are cryptographically secure random strings.
- Tokens are hashed (using a secure hash function) before being stored in the database.
- The plain token is sent via email only and is never returned by any API endpoint.
- Token validation is performed by hashing the provided token and comparing it to the stored hash.

**Frontend responsibility:**
- Treat invitation tokens as sensitive credentials.
- Transmit tokens only over HTTPS.
- Do not log or expose tokens in client-side analytics.
- Use POST with body (not GET with query params) to send tokens to avoid accidental logging in server access logs.

### Email Delivery

**Backend behavior:**
- Invitation emails are sent asynchronously via an outbox pattern.
- The `deliveryStatus` returned during creation is an immediate snapshot (usually `Pending` or `ProviderAccepted`).
- Actual delivery (or failure) happens later and is tracked separately.
- Email delivery failures (hard bounce, soft bounce, blocked) do not prevent invitation acceptance—the invitation remains valid as long as the token is correct and the invitation has not expired or been revoked.

**Frontend implication:**
- The `deliveryStatus` is informational only. The frontend should not block invitation workflows based on delivery status.
- If the invitee reports not receiving the email, the admin can create a new invitation (which expires the old one).

---

## 6. Error Handling Patterns

### Error Response Format

All errors follow the RFC 7807 Problem Details standard:

```json
{
  "type": "about:blank",
  "title": "The request conflicts with the current state.",
  "status": 409,
  "detail": "A pending invitation already exists for this email address.",
  "code": "Invitations.PendingInvitationExists"
}
```

### Invitation Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `Invitations.InvalidEmail` | 400 | Email format is invalid or exceeds 256 characters. |
| `Invitations.InvalidRole` | 400 | The specified `roleId` does not exist or is not valid for this tenant. |
| `Invitations.PasswordRequired` | 400 | Password is required for unauthenticated invitation acceptance. |
| `Invitations.NotFound` | 404 | The invitation does not exist or the token is invalid. |
| `Invitations.MemberAlreadyExists` | 409 | The email is already an active member of the tenant. |
| `Invitations.PendingInvitationExists` | 409 | A pending invitation already exists for this email (rare—usually auto-expired). |
| `Invitations.Expired` | 409 | The invitation has expired. |
| `Invitations.Revoked` | 409 | The invitation has been revoked. |
| `Invitations.AlreadyAccepted` | 409 | The invitation has already been accepted. |
| `Invitations.EmailMismatch` | 409 | The authenticated user's email does not match the invitation email. |
| `Invitations.TenantInactive` | 409 | The tenant is not active (suspended, expired, or closed). |
| `Authentication.Unauthorized` | 401 | User is not authenticated (for tenant-scoped operations). |
| `Tenancy.AccessDenied` | 403 | User does not have `Permission.MembersManage` or tenant context mismatch. |

### Status Code Summary

| Status | When It Occurs |
|--------|----------------|
| `200 OK` | Successful list, inspect, or accept operation. |
| `201 Created` | Successful invitation creation. |
| `204 No Content` | Successful invitation revocation. |
| `400 Bad Request` | Validation error (invalid email, missing password, antiforgery failure). |
| `401 Unauthorized` | User is not authenticated (for tenant-scoped operations). |
| `403 Forbidden` | User does not have the required permission or tenant context mismatch. |
| `404 Not Found` | Invitation not found (invalid token or revoked invitation ID). |
| `409 Conflict` | State conflict (expired, revoked, already accepted, duplicate email, etc.). |
| `429 Too Many Requests` | Rate limit exceeded. |

---

## 7. Frontend Integration Best Practices

### Tenant Admin Flow (Creating & Managing Invitations)

1. **List existing invitations:**
   - Call `GET /api/tenants/{tenantId}/invitations` to display pending, accepted, and revoked invitations.
   - Filter or sort the list in the UI (e.g., show only pending invitations, sort by creation date).

2. **Create a new invitation:**
   - Validate the email format on the frontend.
   - Optionally, allow the admin to select a role from a dropdown (fetch roles via the Roles API).
   - Include the `X-CSRF-TOKEN` and `X-Tenant-Id` headers.
   - Handle errors:
     - `409 MemberAlreadyExists`: Show a message like "This user is already a member. Use the member management page to update their role."
     - `409 InvalidRole`: Show "The selected role is invalid. Please refresh the page and try again."
     - `429 Too Many Requests`: Show rate limit message and disable the form temporarily.

3. **Display delivery status:**
   - Show the `deliveryStatus` from the list endpoint as informational (e.g., "Delivered", "Pending", "Failed").
   - If the status is `HardBounce` or `Failed`, inform the admin and suggest verifying the email address.

4. **Revoke an invitation:**
   - Show a "Revoke" button for pending invitations only (hide it for accepted/revoked/expired invitations).
   - Confirm the action with a modal (e.g., "Are you sure you want to revoke this invitation?").
   - Include the `X-CSRF-TOKEN` and `X-Tenant-Id` headers.
   - Handle errors:
     - `404 Not Found`: The invitation no longer exists—refresh the list.
     - `409 Conflict`: The invitation has already been accepted or revoked—refresh the list.

### Invitee Flow (Inspecting & Accepting Invitations)

1. **Extract the token:**
   - The invitation email contains a link like `https://app.example.com/accept-invitation?token=CfDJ8Gz...`.
   - Extract the `token` query parameter on the acceptance page.

2. **Inspect the invitation:**
   - Call `POST /api/tenant-invitations/inspect` with the token to preview the invitation details.
   - Display the tenant name, role, and expiration time.
   - If the invitation is expired, revoked, or already accepted, show an appropriate message and disable the acceptance form.

3. **Check authentication status:**
   - Call `GET /api/auth/me` to determine if the user is authenticated.
   - If authenticated, display the user's email and a "Join Tenant" button (no password field).
   - If not authenticated, show a "Create Account & Join" form with an email (pre-filled, read-only) and password field.

4. **Accept the invitation:**
   - Include the `X-CSRF-TOKEN` header.
   - If authenticated, send only the `token` in the request body.
   - If not authenticated, send both `token` and `password`.
   - Handle errors:
     - `409 EmailMismatch`: The authenticated user's email does not match the invitation. Show "You are logged in as a different user. Please log out and try again, or contact the tenant admin."
     - `409 MemberAlreadyExists`: The user is already a member. Show "You are already a member of this organization."
     - `409 Expired`: The invitation has expired. Show "This invitation has expired. Please contact the tenant admin for a new invitation."
     - `409 Revoked`: The invitation has been revoked. Show "This invitation has been revoked."

5. **After acceptance:**
   - If `isNewUser` is `true`, redirect the user to the login page with a success message (e.g., "Account created! Please log in.").
   - If `isNewUser` is `false`, refresh the session (call `/api/auth/me`) to update the tenant memberships list, then redirect to the tenant dashboard.

### Security Considerations

- **Token handling:**
  - Do not log or expose invitation tokens in analytics or error tracking.
  - Use POST with body (not GET with query params) to send tokens to the backend.
  - Clear the token from the URL after extracting it (replace the URL with a clean path like `/accept-invitation`).

- **Password security:**
  - Enforce password complexity on the frontend (e.g., minimum 8 characters, at least one uppercase, one digit).
  - Never log or expose passwords.

- **Rate limiting:**
  - Respect rate limits for create, inspect, and accept operations.
  - Show user-friendly messages and disable forms temporarily when limits are hit.

### Error Handling

- **Display user-friendly messages:**
  - Map error codes to localized messages.
  - For `Invitations.MemberAlreadyExists`, suggest contacting the admin or checking the member list.
  - For `Invitations.Expired`, suggest requesting a new invitation.

- **Handle concurrent state changes:**
  - The invitation state may change between inspect and accept (e.g., revoked by admin).
  - Show appropriate error messages and refresh the invitation details if needed.

### Performance Optimization

- **Cache antiforgery tokens:**
  - Fetch the token once on page load and reuse it for create, revoke, and accept operations.

- **Debounce invite creation:**
  - Prevent accidental double-submissions by disabling the submit button after the first click.

- **Lazy-load invitation list:**
  - Only fetch the invitation list when the admin navigates to the invitations page (not on every page load).

---

## Summary

The Invitations module supports two distinct workflows:

1. **Tenant admins** (authenticated, `Permission.MembersManage`) use `/api/tenants/{tenantId}/invitations` to create, list, and revoke invitations.
2. **Invitees** (anonymous or authenticated) use `/api/tenant-invitations` to inspect and accept invitations via a secure token.

Key integration points:

- **Antiforgery protection** is required for create, revoke, and accept operations.
- **Rate limiting** applies to create, inspect, and accept endpoints.
- **Email delivery is asynchronous**—the `deliveryStatus` is informational only.
- **Token security** is critical—treat tokens as sensitive credentials.
- **Authenticated vs. anonymous acceptance** requires conditional UI (password field for new users).
- **State transitions** are strictly enforced—expired, revoked, and accepted invitations cannot be reused.

Frontend developers should implement distinct UIs for tenant admins (invitation management) and invitees (acceptance flow), handle all error states gracefully, and ensure token security throughout the acceptance process.
