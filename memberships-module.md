# Memberships Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/members`

**Role Requirement:** `Permission.MembersManage`

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Members](#21-list-members)
   - 2.2 [Update Membership Status](#22-update-membership-status)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Memberships module manages user memberships within a tenant organization. It allows tenant administrators to view all members, their roles, and their membership statuses, as well as activate or suspend memberships.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Tenant Membership** | A relationship connecting a user (`ApplicationUser`) to a tenant organization (`Tenant`) with a specific role (`Role`) and status (`TenantMembershipStatus`). |
| **Membership Status** | Memberships can be `Active` (can access the tenant) or `Suspended` (access revoked). |
| **Last Active Owner Protection** | A tenant must always have at least one active owner. Suspending the last active owner is prevented by the backend. |
| **Audit Logging** | Status changes are recorded as audit events with before/after state diffs for compliance and traceability. |
| **Role Assignment** | Each membership can be associated with a single role. Role assignments determine the permissions granted to the member within the tenant context. |

### Membership Lifecycle

1. **Creation**: Memberships are created through:
   - **Initial Registration**: The user who registers the tenant is automatically created as an active member with the "Owner" role.
   - **Invitation Acceptance**: An invited user accepts an invitation, creating an active membership with the role specified in the invitation.
2. **Status Changes**: Administrators with `Permission.MembersManage` can change a member's status between `Active` and `Suspended`.
3. **Access Enforcement**: The `TenantContextMiddleware` checks the user's membership status on every tenant-scoped request. Suspended members cannot access any tenant-scoped endpoints (returns 403 Forbidden).

### Important Frontend Implications

- **Tenant context required**: All requests require the `X-Tenant-Id` header matching the `{tenantId}` path parameter.
- **Permission required**: Users must have `Permission.MembersManage` to view or update memberships.
- **Antiforgery required for status updates**: The `PATCH /status` endpoint requires the `X-CSRF-TOKEN` header.
- **Suspended members cannot select the tenant**: If a user's membership is suspended, they will be blocked at the middleware level from performing any tenant-scoped operations.
- **Disable/Suspend button guards**: The frontend should disable the "Suspend" button for the last active owner to prevent errors before submission.

---

## 2. API Endpoints Reference

### 2.1 List Members

#### `GET /api/tenants/{tenantId}/members`

Retrieves all members of the specified tenant, including their user details, assigned roles, and membership statuses.

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
    "membershipId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "userId": "e1b2c3d4-5678-90ab-cdef-1234567890ab",
    "email": "owner@example.com",
    "roleId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "roleName": "Owner",
    "status": "Active"
  },
  {
    "membershipId": "4gb96g75-6828-5673-c4gd-3d074g77bfb7",
    "userId": "f2c3d4e5-6789-01bc-def1-234567890abc",
    "email": "teacher@example.com",
    "roleId": "8d0f7780-8536-51ef-a55c-3d074g77bf89",
    "roleName": "Teacher",
    "status": "Suspended"
  }
]
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.MembersManage` or tenant context mismatch.

**Important Notes:**
- Returns both `Active` and `Suspended` members.
- `roleId` and `roleName` may be `null` if the member has no role assigned.
- `status` is serialized as a string (`"Active"` or `"Suspended"`).
- The list is not server-side paginated; all members of the tenant are returned.

---

### 2.2 Update Membership Status

#### `PATCH /api/tenants/{tenantId}/members/{membershipId}/status`

Updates the status of a tenant membership (e.g., activating or suspending a member).

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.MembersManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `membershipId` | `Guid` | The membership identifier to update. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body:**

```json
{
  "status": "Suspended"
}
```

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `status` | `string` | Yes | Must be a valid `TenantMembershipStatus` value (`"Active"` or `"Suspended"`). Case-insensitive. |

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Invalid status value (not "Active" or "Suspended") or antiforgery validation failure.
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.MembersManage` or tenant context mismatch.
- `404 NotFound` — Membership not found in the specified tenant.
- `409 Conflict` — One of:
  - `Memberships.AlreadyInStatus`: The membership is already in the requested status.
  - `Memberships.CannotDisableLastOwner`: Attempting to suspend the last active owner of the tenant.

**Important Notes:**
- **Last active owner protection**: If the member is an active owner and is the only active owner in the tenant, the backend rejects suspension with a 409 Conflict error (`Memberships.CannotDisableLastOwner`).
- **Idempotency/Redundancy check**: Sending the same status the membership already has returns `Memberships.AlreadyInStatus` (409 Conflict).
- **Audit event created**: Every successful status change generates an audit log entry (`TenantMembership.StatusChanged`) recording the actor, target membership, and previous/new status.

---

## 3. Request & Response DTOs

### TenantMemberResponse

| Field | Type | Description |
|-------|------|-------------|
| `membershipId` | `Guid` | The unique identifier of the tenant membership record. |
| `userId` | `Guid` | The unique identifier of the underlying user account. |
| `email` | `string` | The user's email address. |
| `roleId` | `Guid?` | The unique identifier of the assigned role, or `null` if unassigned. |
| `roleName` | `string?` | The display name of the assigned role, or `null` if unassigned. |
| `status` | `string` | The membership status (see [TenantMembershipStatus](#tenantmembershipstatus)). |

### UpdateMembershipStatusRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `status` | `string` | Yes | Valid `TenantMembershipStatus` name (`"Active"` or `"Suspended"`). Case-insensitive. |

---

## 4. Enum Reference

### TenantMembershipStatus

Enum values are serialized/parsed as **strings** (case-insensitive on request parsing).

| Value | Integer Value | Description |
|-------|---------------|-------------|
| `Active` | 1 | The member is active and can access the tenant. |
| `Suspended` | 2 | The member is suspended and cannot access the tenant. |

---

## 5. Validation & Business Rules

### Status Value Validation

**Backend-enforced:**
- The `status` string in `UpdateMembershipStatusRequest` must parse to a valid `TenantMembershipStatus` enum value (`Active` or `Suspended`).
- Case-insensitive parsing is supported (e.g., `"active"`, `"ACTIVE"`, `"Active"` are all accepted).

**Error returned:** `Memberships.InvalidStatus` (400 Bad Request)

### State Transition Rules

**Backend-enforced:**
- **No-op transitions prohibited**: Updating a membership to its current status is treated as a conflict.
  - Active → Active: Rejected (`Memberships.AlreadyInStatus`)
  - Suspended → Suspended: Rejected (`Memberships.AlreadyInStatus`)
  - Active → Suspended: Allowed (subject to owner protection)
  - Suspended → Active: Allowed

**Error returned:** `Memberships.AlreadyInStatus` (409 Conflict)

### Last Active Owner Protection

**Backend-enforced:**
- When suspending a member (`status = "Suspended"`), the backend checks if:
  1. The target member currently holds an "Owner" role (a role with `Permission.All` or the designated owner role).
  2. The target member is currently `Active`.
  3. The count of active owners in the tenant is ≤ 1.
- If all three conditions are met, the operation is blocked.

**Error returned:** `Memberships.CannotDisableLastOwner` (409 Conflict)

### Tenant Isolation Rules

**Backend-enforced:**
- The `{tenantId}` in the URL must match the `X-Tenant-Id` header.
- The membership being accessed or modified must belong to the specified tenant.
- Cross-tenant membership lookups or updates fail closed with 404 NotFound.

---

## 6. Error Handling Patterns

### Error Response Format

All errors follow RFC 7807 Problem Details:

```json
{
  "type": "about:blank",
  "title": "The request conflicts with the current state.",
  "status": 409,
  "detail": "Cannot disable or suspend the last active owner of the tenant.",
  "code": "Memberships.CannotDisableLastOwner"
}
```

### Membership Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `Memberships.InvalidStatus` | 400 | The status string is not a valid enum value ("Active" or "Suspended"). |
| `Memberships.NotFound` | 404 | The membership ID does not exist in the specified tenant. |
| `Memberships.AlreadyInStatus` | 409 | The membership already has the requested status. |
| `Memberships.CannotDisableLastOwner` | 409 | Cannot suspend the last remaining active owner of the tenant. |
| `Authentication.Unauthorized` | 401 | User is not authenticated. |
| `Tenancy.AccessDenied` | 403 | User lacks `Permission.MembersManage` or is not an active member. |
| `Antiforgery.ValidationFailed` | 400 | Antiforgery token missing or invalid. |

---

## 7. Frontend Integration Best Practices

### Member List Page

1. **Data Fetching:**
   - Fetch the members list on page load using `GET /api/tenants/{tenantId}/members`.
   - Ensure `X-Tenant-Id` is present in request headers.

2. **UI Controls & Indicators:**
   - Display a status badge for each member: green for `Active`, red/gray for `Suspended`.
   - Provide an action toggle or button: "Suspend" for active members, "Activate" for suspended members.
   - Count the number of active owners in the list. If only 1 active owner exists, disable the "Suspend" action for that member and display a tooltip: *"Cannot suspend the only active owner."*

3. **Confirmation Dialogs:**
   - When suspending a member, show a confirmation modal: *"Suspending this member will immediately revoke their access to this organization. Continue?"*
   - When activating a member, show: *"This will restore the member's access to this organization. Continue?"*

### Status Update Execution

1. **Request Construction:**
   - Send `PATCH /api/tenants/{tenantId}/members/{membershipId}/status`.
   - Include `X-CSRF-TOKEN` and `X-Tenant-Id` headers.
   - Request body: `{ "status": "Suspended" }` or `{ "status": "Active" }`.

2. **Optimistic Updates vs. Re-fetching:**
   - On `204 NoContent`, update the local member's `status` field in client state.
   - Alternatively, re-fetch the members list to ensure synchronization.

3. **Error Handling:**
   - `409 Memberships.CannotDisableLastOwner`: Display an alert explaining that another member must be promoted to Owner before this one can be suspended.
   - `409 Memberships.AlreadyInStatus`: Update local state silently (another admin may have already changed it).
   - `404 Memberships.NotFound`: Show an error and refresh the members list.

### Edge Cases to Handle

- **Self-Suspension**: An admin can suspend their own membership if there is another active owner. If an admin suspends themselves, their subsequent requests will fail with `403 Tenancy.AccessDenied`. The frontend should detect self-suspension, display a notification, and redirect to the tenant selection / home screen.
- **Role Display**: When `roleName` is `null`, display a placeholder such as *"No role assigned"* or *"—"*.

---

## Summary

The Memberships module provides member roster visibility and access control (active vs. suspended) within a tenant. Key integration takeaways:

- **List members**: `GET /api/tenants/{tenantId}/members`
- **Update status**: `PATCH /api/tenants/{tenantId}/members/{membershipId}/status` with `{ "status": "Active" | "Suspended" }`
- **Guard last owner**: Prevent UI suspension of the sole active owner to avoid backend 409 errors.
- **All mutations require antiforgery tokens and tenant context headers.**
