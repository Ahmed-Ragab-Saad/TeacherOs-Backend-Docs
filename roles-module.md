# Roles Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/roles`

**Role Requirement:** `Permission.RolesView` (list/get), `Permission.RolesManage` (create/update/delete/assign)

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Roles](#21-list-roles)
   - 2.2 [Get Role](#22-get-role)
   - 2.3 [Create Role](#23-create-role)
   - 2.4 [Update Role](#24-update-role)
   - 2.5 [Delete Role](#25-delete-role)
   - 2.6 [Assign Membership Role](#26-assign-membership-role)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Permission Reference](#4-permission-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Roles module manages role-based access control (RBAC) within a tenant. Roles are collections of permissions that can be assigned to tenant memberships to control what actions users can perform.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Role** | A named collection of permission codes that define what actions a member can perform within a tenant. |
| **Permission** | A string-based capability identifier (e.g., `"attendance.view"`, `"members.manage"`) that gates access to specific endpoints and operations. |
| **Owner Role** | A special protected role named `"Owner"` that grants all permissions and cannot be deleted or have its permissions modified to prevent tenant lockout. |
| **Role Assignment** | Memberships are linked to a single role (or no role). Changing a member's role updates their effective permissions. |
| **Tenant Isolation** | Roles are tenant-scoped. Each tenant has its own set of roles independent of other tenants. |
| **Audit Logging** | Role creation, updates, deletions, and assignments are recorded as audit events for compliance and traceability. |

### Permission Model

The TeacherOS permission system uses **fine-grained, string-based permission codes** organized by feature area:

- **Attendance**: `attendance.view`, `attendance.record`, `attendance.correct`
- **Finance**: `payment.record`, `payment.adjust`
- **Sessions**: `session.close`, `shift.close`
- **Content**: `content.publish`
- **Members**: `members.manage`
- **Subjects**: `subjects.view`, `subjects.manage`
- **Teachers**: `teachers.view`, `teachers.manage`
- **Groups**: `groups.view`, `groups.manage`, `groups.enrollments.manage`
- **Schedules**: `schedules.view`, `schedules.manage`
- **Sessions**: `sessions.view`, `sessions.manage`
- **Audit**: `audit.view`
- **Roles**: `roles.view`, `roles.manage`
- **Courses**: `courses.view`, `courses.manage`, `courses.publish`
- **Learning Resources**: `learning.resources.view`, `learning.resources.manage`, `learning.resources.publish`
- **Questions**: `questions.view`, `questions.manage`
- **Homework**: `homework.view`, `homework.manage`, `homework.publish`, `homework.grade`
- **Exams**: `exams.view`, `exams.manage`, `exams.publish`, `exams.grade`
- **AI**: `ai.generate`

See [Permission Reference](#4-permission-reference) for the complete list.

### Role Lifecycle

1. **Creation**: An admin with `Permission.RolesManage` creates a role by providing a name and an array of permission codes.
2. **Assignment**: Roles are assigned to memberships via the membership role assignment endpoint.
3. **Updates**: Role name and permissions can be updated independently. Updating a role's permissions affects all members assigned that role.
4. **Deletion**: Roles can be deleted only if they are not currently assigned to any active memberships. The "Owner" role cannot be deleted.
5. **Owner Protection**: The "Owner" role (case-insensitive name match) is immutable—its permissions cannot be changed and it cannot be deleted.

### Important Frontend Implications

- **Tenant context required**: All requests require the `X-Tenant-Id` header matching the `{tenantId}` path parameter.
- **Permission-based visibility**: Users need `Permission.RolesView` to view roles and `Permission.RolesManage` to modify them.
- **Owner role protection**: Disable edit/delete UI controls for the "Owner" role to prevent user confusion.
- **Permission validation**: The backend validates all permission codes against the known list. Sending invalid codes results in a 400 Bad Request error.
- **Partial updates**: The `PATCH /roles/{roleId}` endpoint supports updating name and/or permissions independently—send only the fields you want to change.
- **Role deletion restrictions**: Roles assigned to active memberships cannot be deleted. The frontend should check membership counts or handle 409 errors gracefully.

---

## 2. API Endpoints Reference

### 2.1 List Roles

#### `GET /api/tenants/{tenantId}/roles`

Retrieves all roles defined within the specified tenant.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesView` within the specified tenant

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
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "name": "Owner",
    "permissionCodes": [
      "attendance.view",
      "attendance.record",
      "members.manage",
      "roles.view",
      "roles.manage"
      // ... all 43 permission codes
    ],
    "isOwnerRole": true
  },
  {
    "id": "8d0f7780-8536-51ef-a55c-3d074g77bf89",
    "name": "Teacher",
    "permissionCodes": [
      "attendance.view",
      "attendance.record",
      "homework.view",
      "homework.manage",
      "exams.view"
    ],
    "isOwnerRole": false
  }
]
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesView` or tenant context mismatch.

**Important Notes:**
- Returns all roles for the tenant (no pagination).
- The "Owner" role is always included and has `isOwnerRole: true`.
- Permission codes are returned as an array of strings.

---

### 2.2 Get Role

#### `GET /api/tenants/{tenantId}/roles/{roleId}`

Retrieves details of a specific role by ID.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesView` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `roleId` | `Guid` | The role identifier. |

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "8d0f7780-8536-51ef-a55c-3d074g77bf89",
  "name": "Teacher",
  "permissionCodes": [
    "attendance.view",
    "attendance.record",
    "homework.view",
    "homework.manage",
    "exams.view"
  ],
  "isOwnerRole": false
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesView` or tenant context mismatch.
- `404 Not Found` — Role not found in the specified tenant.

---

### 2.3 Create Role

#### `POST /api/tenants/{tenantId}/roles`

Creates a new role with the specified name and permissions.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

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
  "name": "Student Assistant",
  "permissionCodes": [
    "attendance.view",
    "homework.view",
    "exams.view"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | Yes | Role display name. Max 100 characters. Trimmed automatically. Must be unique within the tenant. |
| `permissionCodes` | `string[]` | Yes | Array of valid permission codes. At least one permission is required. Duplicates are automatically removed. |

**Response:** `201 Created`

```json
{
  "id": "9e1f8891-9647-62fg-b66d-4e185h88cg90",
  "name": "Student Assistant",
  "permissionCodes": [
    "attendance.view",
    "homework.view",
    "exams.view"
  ],
  "isOwnerRole": false
}
```

**Location Header:** `/api/tenants/{tenantId}/roles/{roleId}`

**Possible Errors:**
- `400 Bad Request` — One of:
  - Name is empty or exceeds 100 characters.
  - `permissionCodes` is empty or contains invalid permission codes.
  - Antiforgery validation failed.
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesManage` or tenant context mismatch.
- `409 Conflict` — A role with the same name already exists in this tenant (`Roles.NameExists`).

**Important Notes:**
- **Name uniqueness**: Role names are case-sensitive for uniqueness checks.
- **Permission validation**: All permission codes are validated against the known list. Unknown codes result in a 400 error with a message identifying the invalid code.
- **Duplicate removal**: Duplicate permission codes in the request are automatically deduplicated.

---

### 2.4 Update Role

#### `PATCH /api/tenants/{tenantId}/roles/{roleId}`

Updates a role's name and/or permissions. Supports partial updates—send only the fields you want to change.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `roleId` | `Guid` | The role identifier. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body (update name only):**

```json
{
  "name": "Senior Teacher"
}
```

**Request Body (update permissions only):**

```json
{
  "permissionCodes": [
    "attendance.view",
    "attendance.record",
    "homework.view",
    "homework.manage",
    "homework.grade",
    "exams.view"
  ]
}
```

**Request Body (update both):**

```json
{
  "name": "Senior Teacher",
  "permissionCodes": [
    "attendance.view",
    "attendance.record",
    "homework.view",
    "homework.manage",
    "homework.grade",
    "exams.view"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string?` | No | New role name. Max 100 characters. If omitted, the name is not changed. |
| `permissionCodes` | `string[]?` | No | New permission codes array. If omitted, permissions are not changed. Must contain at least one valid permission. |

**Response:** `200 OK`

```json
{
  "id": "8d0f7780-8536-51ef-a55c-3d074g77bf89",
  "name": "Senior Teacher",
  "permissionCodes": [
    "attendance.view",
    "attendance.record",
    "homework.view",
    "homework.manage",
    "homework.grade",
    "exams.view"
  ],
  "isOwnerRole": false
}
```

**Possible Errors:**
- `400 Bad Request` — One of:
  - Both `name` and `permissionCodes` are `null` (nothing to update).
  - Name is empty or exceeds 100 characters.
  - `permissionCodes` is empty or contains invalid codes.
  - Antiforgery validation failed.
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesManage` or tenant context mismatch.
- `404 Not Found` — Role not found in the specified tenant.
- `409 Conflict` — One of:
  - `Roles.NameExists`: The new name conflicts with an existing role.
  - `Roles.OwnerProtected`: Attempting to update the "Owner" role's permissions (renaming is allowed if the name remains "Owner" or changes to another name, but permission changes are blocked).

**Important Notes:**
- **Partial updates**: Send only `name` to rename, only `permissionCodes` to change permissions, or both to update everything.
- **Owner role protection**: The "Owner" role's permissions cannot be changed. The backend checks `IsOwnerRole` (name equals "Owner", case-insensitive) and rejects permission updates with a 409 error. Renaming away from "Owner" is allowed, which removes the protection.
- **Empty request rejection**: If both fields are `null`, the backend returns `Roles.InvalidInput` (400 Bad Request).

---

### 2.5 Delete Role

#### `DELETE /api/tenants/{tenantId}/roles/{roleId}`

Deletes a role. Roles assigned to active memberships cannot be deleted.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `roleId` | `Guid` | The role identifier. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body:** None

**Response:** `204 No Content`

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesManage` or tenant context mismatch.
- `404 Not Found` — Role not found in the specified tenant.
- `409 Conflict` — One of:
  - `Roles.OwnerProtected`: Attempting to delete the "Owner" role.
  - `Roles.RoleInUse`: The role is currently assigned to one or more memberships.

**Important Notes:**
- **Owner role protection**: The "Owner" role cannot be deleted.
- **In-use protection**: Roles assigned to any membership (active or suspended) cannot be deleted. The frontend should unassign members from the role before deletion or handle the 409 error gracefully.

---

### 2.6 Assign Membership Role

#### `PATCH /api/tenants/{tenantId}/members/{membershipId}/role`

Assigns a role to a membership or clears the membership's role assignment.

**Authentication:** Required (cookie-based)

**Authorization:** `Permission.RolesManage` within the specified tenant

**Tenant Context:** Required (`X-Tenant-Id` header must match `{tenantId}`)

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | `Guid` | The tenant identifier. |
| `membershipId` | `Guid` | The membership identifier. |

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token
- `X-Tenant-Id` (required): Must match `{tenantId}`

**Request Body (assign a role):**

```json
{
  "roleId": "8d0f7780-8536-51ef-a55c-3d074g77bf89"
}
```

**Request Body (clear role assignment):**

```json
{
  "roleId": null
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `roleId` | `Guid?` | Yes | The role identifier to assign, or `null` to clear the role assignment. |

**Response:** `204 No Content`

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — User does not have `Permission.RolesManage` or tenant context mismatch.
- `404 Not Found` — Membership or role not found in the specified tenant.
- `409 Conflict` — The membership is already assigned the requested role (`Roles.AlreadyAssigned`).

**Important Notes:**
- **Idempotency check**: Assigning the same role the membership already has results in a 409 Conflict (`Roles.AlreadyAssigned`).
- **Null assignment**: Sending `{ "roleId": null }` clears the role assignment. If the role is already `null`, this also triggers `Roles.AlreadyAssigned`.
- **Audit logging**: Role assignments and changes are recorded as audit events (`TenantMembership.RoleChanged`).

---

## 3. Request & Response DTOs

### CreateRoleRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `name` | `string` | Yes | Non-empty, max 100 characters. Trimmed automatically. Must be unique within the tenant. |
| `permissionCodes` | `string[]` | Yes | Array of valid permission codes. At least one required. Duplicates are deduplicated automatically. |

### UpdateRoleRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `name` | `string?` | No | Non-empty, max 100 characters if provided. Trimmed automatically. Must be unique within the tenant. |
| `permissionCodes` | `string[]?` | No | Array of valid permission codes if provided. At least one required. Duplicates are deduplicated automatically. At least one of `name` or `permissionCodes` must be provided. |

### AssignMembershipRoleRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `roleId` | `Guid?` | Yes | Role identifier to assign, or `null` to clear the role assignment. Must exist in the tenant if not `null`. |

### RoleResponse

| Field | Type | Description |
|-------|------|-------------|
| `id` | `Guid` | The unique identifier of the role. |
| `name` | `string` | The role display name. |
| `permissionCodes` | `string[]` | Array of permission codes granted by this role. |
| `isOwnerRole` | `bool` | `true` if this is the protected "Owner" role; `false` otherwise. |

---

## 4. Permission Reference

### Complete Permission List

| Permission Code | Description |
|-----------------|-------------|
| `attendance.view` | View attendance records |
| `attendance.record` | Record attendance |
| `attendance.correct` | Correct/edit attendance records |
| `payment.record` | Record payments |
| `payment.adjust` | Adjust/modify payments |
| `session.close` | Close sessions |
| `shift.close` | Close shifts |
| `content.publish` | Publish content |
| `members.manage` | Manage tenant members (invite, suspend, etc.) |
| `subjects.view` | View subjects |
| `subjects.manage` | Create, update, delete subjects |
| `teachers.view` | View teachers |
| `teachers.manage` | Manage teacher records |
| `groups.view` | View groups/classes |
| `groups.manage` | Create, update, delete groups |
| `groups.enrollments.manage` | Manage student enrollments in groups |
| `schedules.view` | View schedules |
| `schedules.manage` | Create, update, delete schedules |
| `sessions.view` | View sessions |
| `sessions.manage` | Create, update, delete sessions |
| `audit.view` | View audit logs |
| `roles.view` | View roles |
| `roles.manage` | Create, update, delete, assign roles |
| `courses.view` | View courses |
| `courses.manage` | Create, update, delete courses |
| `courses.publish` | Publish courses |
| `learning.resources.view` | View learning resources/assets |
| `learning.resources.manage` | Manage learning resources |
| `learning.resources.publish` | Publish learning resources |
| `questions.view` | View question bank |
| `questions.manage` | Manage questions |
| `homework.view` | View homework assignments |
| `homework.manage` | Create, update, delete homework |
| `homework.publish` | Publish homework |
| `homework.grade` | Grade homework submissions |
| `exams.view` | View exams |
| `exams.manage` | Create, update, delete exams |
| `exams.publish` | Publish exams |
| `exams.grade` | Grade exam submissions |
| `ai.generate` | Use AI generation features |

**Total:** 43 permissions

**Owner Role:** The "Owner" role is granted all 43 permissions and is protected from modification.

---

## 5. Validation & Business Rules

### Name Validation

**Backend-enforced:**
- Must not be empty or whitespace.
- Maximum length: 100 characters.
- Trimmed automatically.
- Must be unique within the tenant (case-sensitive).

**Errors returned:**
- `Roles.InvalidInput` (400 Bad Request) if empty or too long.
- `Roles.NameExists` (409 Conflict) if a role with the same name exists.

### Permission Code Validation

**Backend-enforced:**
- At least one permission code is required.
- All codes must exist in the `Permission.All` list.
- Duplicate codes are automatically removed.
- Unknown permission codes trigger a descriptive error message identifying the invalid code.

**Error returned:** `Roles.InvalidInput` (400 Bad Request) with a message like `"'invalid.permission' is not a recognized permission code."`

### Owner Role Protection

**Backend-enforced:**
- The role named "Owner" (case-insensitive match) is protected:
  - Its permissions cannot be changed (update permissions endpoint returns `Roles.OwnerProtected`).
  - It cannot be deleted (delete endpoint returns `Roles.OwnerProtected`).
- Renaming the "Owner" role to another name removes the protection.

**Errors returned:**
- `Roles.OwnerProtected` (409 Conflict) when attempting to update permissions or delete.

### Role Deletion Rules

**Backend-enforced:**
- Roles assigned to any membership (active or suspended) cannot be deleted.
- The "Owner" role cannot be deleted regardless of assignment status.

**Errors returned:**
- `Roles.RoleInUse` (409 Conflict) if assigned to memberships.
- `Roles.OwnerProtected` (409 Conflict) if attempting to delete "Owner".

### Role Assignment Rules

**Backend-enforced:**
- Assigning a role that is already assigned to the membership returns `Roles.AlreadyAssigned`.
- Assigning `null` when the role is already `null` also returns `Roles.AlreadyAssigned`.
- The role must exist in the same tenant as the membership.

**Errors returned:**
- `Roles.AlreadyAssigned` (409 Conflict) for redundant assignments.
- `Roles.NotFound` (404 Not Found) if the role does not exist.

### Partial Update Rules

**Backend-enforced:**
- The `PATCH /roles/{roleId}` endpoint requires at least one of `name` or `permissionCodes` to be provided.
- If both are `null`, the request is rejected with `Roles.InvalidInput`.

**Error returned:** `Roles.InvalidInput` (400 Bad Request)

---

## 6. Error Handling Patterns

### Error Response Format

All errors follow RFC 7807 Problem Details:

```json
{
  "type": "about:blank",
  "title": "The request conflicts with the current state.",
  "status": 409,
  "detail": "A role with the same name already exists in this tenant.",
  "code": "Roles.NameExists"
}
```

### Role Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `Roles.InvalidInput` | 400 | Invalid role data (empty name, empty permissions, unknown permission code, or both fields null in update). |
| `Roles.NotFound` | 404 | Role not found in the specified tenant. |
| `Roles.NameExists` | 409 | A role with the same name already exists in this tenant. |
| `Roles.OwnerProtected` | 409 | Attempting to delete the "Owner" role or modify its permissions. |
| `Roles.RoleInUse` | 409 | The role is assigned to one or more memberships and cannot be deleted. |
| `Roles.AlreadyAssigned` | 409 | The membership already has the requested role assignment. |
| `Authentication.Unauthorized` | 401 | User is not authenticated. |
| `Tenancy.AccessDenied` | 403 | User lacks the required permission (`RolesView` or `RolesManage`) or tenant context mismatch. |
| `Antiforgery.ValidationFailed` | 400 | Antiforgery token missing or invalid. |

---

## 7. Frontend Integration Best Practices

### Roles List Page

1. **Fetch roles on page load:**
   - Call `GET /api/tenants/{tenantId}/roles`.
   - Display a table with columns: Name, Permissions Count, Actions.
   - Badge or icon for `isOwnerRole: true` (e.g., "OWNER" badge).

2. **Disable actions for Owner role:**
   - Disable "Edit Permissions" and "Delete" buttons for the Owner role.
   - Show a tooltip: *"The Owner role is protected and cannot be deleted or have its permissions changed."*

3. **Show permission count:**
   - Display the number of permissions (e.g., "12 permissions") rather than listing all codes.
   - Provide a "View Details" or expand action to show the full list.

### Role Creation

1. **UI Components:**
   - Text input for role name.
   - Multi-select or checkbox list for permissions (grouped by feature area).
   - Validate that at least one permission is selected before enabling the "Create" button.

2. **Submission:**
   - Include `X-CSRF-TOKEN` and `X-Tenant-Id` headers.
   - Handle errors:
     - `409 Roles.NameExists`: Display "A role with this name already exists. Choose a different name."
     - `400 Roles.InvalidInput`: Display specific validation messages (e.g., "Permission code 'xyz' is not valid.").

3. **Success:**
   - Add the new role to the local state or refresh the roles list.
   - Show a success message: "Role 'Teacher' created successfully."

### Role Editing

1. **Fetch role details:**
   - Call `GET /api/tenants/{tenantId}/roles/{roleId}` to populate the edit form.

2. **Partial updates:**
   - Allow editing name and permissions independently.
   - Send only the changed fields in the `PATCH` request.

3. **Owner role restrictions:**
   - Disable the permissions editor for `isOwnerRole: true`.
   - Allow renaming the Owner role (which removes the protection).

4. **Handle errors:**
   - `409 Roles.OwnerProtected`: Display "The Owner role's permissions cannot be changed."
   - `409 Roles.NameExists`: Display "A role with this name already exists."

5. **Success:**
   - Update the role in local state or refresh the list.
   - Show a success message: "Role updated successfully."

### Role Deletion

1. **Pre-deletion check:**
   - Optionally, fetch the count of memberships using this role (not directly exposed by this API, but may be available via a separate endpoint or in the role details).
   - Warn the user: "This role is assigned to 3 members. Please reassign them to another role before deleting."

2. **Confirmation modal:**
   - "Are you sure you want to delete this role? This action cannot be undone."

3. **Handle errors:**
   - `409 Roles.RoleInUse`: Display "This role is currently assigned to members. Please reassign them first."
   - `409 Roles.OwnerProtected`: Display "The Owner role cannot be deleted."

4. **Success:**
   - Remove the role from local state or refresh the list.
   - Show a success message: "Role deleted successfully."

### Role Assignment

1. **Member management page:**
   - For each member, show their current role in a dropdown or select input.
   - Fetch the roles list to populate the dropdown options.
   - Include a "No role" option for `{ "roleId": null }`.

2. **Assignment logic:**
   - Call `PATCH /api/tenants/{tenantId}/members/{membershipId}/role` with the selected `roleId`.
   - Include `X-CSRF-TOKEN` and `X-Tenant-Id` headers.

3. **Handle errors:**
   - `409 Roles.AlreadyAssigned`: Silently ignore or update UI (another admin may have changed it).
   - `404 Roles.NotFound`: Refresh the roles list (the role may have been deleted).

4. **Success:**
   - Update the member's role in local state.
   - Show a success message: "Role assigned successfully."

### Permission UI Organization

**Group permissions by feature area for better UX:**

- **Attendance**: attendance.view, attendance.record, attendance.correct
- **Finance**: payment.record, payment.adjust
- **Members & Roles**: members.manage, roles.view, roles.manage
- **Subjects**: subjects.view, subjects.manage
- **Teachers**: teachers.view, teachers.manage
- **Groups**: groups.view, groups.manage, groups.enrollments.manage
- **Schedules & Sessions**: schedules.view, schedules.manage, sessions.view, sessions.manage, session.close, shift.close
- **Courses**: courses.view, courses.manage, courses.publish
- **Learning Resources**: learning.resources.view, learning.resources.manage, learning.resources.publish
- **Question Bank**: questions.view, questions.manage
- **Homework**: homework.view, homework.manage, homework.publish, homework.grade
- **Exams**: exams.view, exams.manage, exams.publish, exams.grade
- **Content & AI**: content.publish, ai.generate
- **Audit**: audit.view

### Recommended Role Templates

Offer pre-configured templates for common roles:

- **Administrator**: All permissions except Owner-specific protections.
- **Teacher**: attendance.view, attendance.record, homework.*, exams.*, questions.view, courses.view, learning.resources.view.
- **Content Manager**: courses.*, learning.resources.*, homework.manage, homework.publish, exams.manage, exams.publish.
- **Viewer**: *.view permissions only (read-only access).

---

## Summary

The Roles module provides fine-grained, tenant-scoped RBAC. Key integration takeaways:

- **List roles**: `GET /api/tenants/{tenantId}/roles`
- **Create role**: `POST /api/tenants/{tenantId}/roles` with `name` and `permissionCodes`
- **Update role**: `PATCH /api/tenants/{tenantId}/roles/{roleId}` with optional `name` and/or `permissionCodes`
- **Delete role**: `DELETE /api/tenants/{tenantId}/roles/{roleId}` (protected for Owner, blocked if in use)
- **Assign role**: `PATCH /api/tenants/{tenantId}/members/{membershipId}/role` with `roleId` or `null`
- **Owner role is protected**: Cannot be deleted or have permissions changed.
- **All mutations require antiforgery tokens and tenant context headers.**
- **Permission codes are validated**: Invalid codes result in descriptive 400 errors.
