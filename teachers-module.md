# Teachers Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/teachers`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Specific permissions: `teachers.view` for read operations, `teachers.manage` for write and assignment operations.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Teacher Profiles Management](#21-teacher-profiles-management)
     - [List Teachers](#get-apitenantstenantidteachers)
     - [Get Teacher by ID](#get-apitenantstenantidteachersteacherid)
     - [Create Teacher](#post-apitenantstenantidteachers)
     - [Update Teacher](#patch-apitenantstenantidteachersteacherid)
   - 2.2 [Status Management](#22-status-management)
     - [Deactivate Teacher](#post-apitenantstenantidteachersteacheriddeactivate)
     - [Reactivate Teacher](#post-apitenantstenantidteachersteacheridreactivate)
   - 2.3 [Subject & Branch Assignments](#23-subject--branch-assignments)
     - [Get Assigned Subjects](#get-apitenantstenantidteachersteacheridsubjects)
     - [Replace Assigned Subjects](#put-apitenantstenantidteachersteacheridsubjects)
     - [Get Assigned Branches](#get-apitenantstenantidteachersteacheridbranches)
     - [Replace Assigned Branches](#put-apitenantstenantidteachersteacheridbranches)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Search, Sorting & Pagination](#6-search-sorting--pagination)
7. [Error Handling Patterns](#7-error-handling-patterns)
8. [Frontend Integration Best Practices](#8-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Teachers module manages teacher profiles, their academic qualifications (subject specializations), campus assignments (branches), and user account linkages (`MembershipId`) within a tenant organization.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Teacher Profile** | Primary profile representing an instructor with contact details (`FullName`, `PhoneNumber`, `PhotoUrl`) and status. |
| **Membership Link** | Optional linkage (`membershipId`) connecting the teacher profile to a tenant membership user account. |
| **Subject Specializations** | Many-to-many relationship linking a teacher to active subjects they are qualified to teach. |
| **Branch Assignments** | Many-to-many relationship linking a teacher to physical campuses/branches where they operate. |
| **Teacher Status** | State indicator: `Active = 1`, `Inactive = 2`. |

### Important Relationships & Dependencies

```
 +-----------------+          1:N          +-------------------+
 |  Subject (Active)|<----------------------|   TeacherSubject  |
 +-----------------+                       +---------+---------+
                                                     |
                                                     | N:1
 +-----------------+          1:N          +---------v---------+
 |      Branch     |<----------------------|   TeacherBranch   |
 +-----------------+                       +---------+---------+
                                                     |
                                                     | N:1
 +-----------------+          1:1 (opt)    +---------v---------+
 | TenantMembership|<----------------------|   TeacherProfile  |
 +-----------------+                       +-------------------+
```

### Important Frontend Implications

- **Setup Flow**: Create the teacher profile first, then assign subjects (`PUT /subjects`) and branches (`PUT /branches`).
- **Active Subject Rule**: A teacher cannot be assigned to an `Inactive` subject. The backend validates that all subject IDs exist and are active.
- **Branch Existence**: All branch IDs provided during branch assignment must exist within the tenant.
- **Profile Deactivation**: Inactive teachers cannot be assigned to new groups or scheduling sessions.
- **CSRF Tokens**: All mutating operations (`POST`, `PATCH`, `PUT`) require `X-CSRF-TOKEN` and `X-Tenant-Id`.

---

## 2. API Endpoints Reference

### 2.1 Teacher Profiles Management

#### `GET /api/tenants/{tenantId}/teachers`

Retrieves a paginated list of teachers with their assigned subject IDs and branch IDs.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.view`

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.

**Query Parameters:**
- `search` (string, optional): Substring filter on `FullName` or phone number.
- `subjectId` (Guid, optional): Filter teachers assigned to a specific subject.
- `branchId` (Guid, optional): Filter teachers assigned to a specific branch.
- `status` (TeacherStatus integer/string, optional): Filter by `1` (Active) or `2` (Inactive).
- `page` (integer, optional, default `1`): Page number (min: 1).
- `pageSize` (integer, optional, default `50`): Items per page (range: 1–100).

**Request Body:** None

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
      "membershipId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "fullName": "Dr. Sarah Connor",
      "phoneNumber": "+1-555-0199",
      "photoUrl": "https://assets.teacheros.io/photos/sarah.jpg",
      "status": 1,
      "subjectIds": [
        "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4"
      ],
      "branchIds": [
        "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
      ],
      "createdAtUtc": "2026-09-01T12:00:00Z",
      "updatedAtUtc": "2026-09-01T12:00:00Z"
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 50
}
```

---

#### `GET /api/tenants/{tenantId}/teachers/{teacherId}`

Retrieves complete details for a single teacher, including their assigned subject and branch IDs.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.view`

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.
- `teacherId` (Guid, required): Target teacher ID.

**Response:** `200 OK`

```json
{
  "id": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "membershipId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "fullName": "Dr. Sarah Connor",
  "phoneNumber": "+1-555-0199",
  "photoUrl": "https://assets.teacheros.io/photos/sarah.jpg",
  "status": 1,
  "subjectIds": [
    "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4"
  ],
  "branchIds": [
    "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
  ],
  "createdAtUtc": "2026-09-01T12:00:00Z",
  "updatedAtUtc": "2026-09-01T12:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Unauthenticated.
- `403 Forbidden` — Missing `teachers.view` permission.
- `404 Not Found` — Teacher not found (`Teachers.NotFound`).

---

#### `POST /api/tenants/{tenantId}/teachers`

Creates a new teacher profile.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)
- `X-Tenant-Id` (required)

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.

**Request Body:**

```json
{
  "fullName": "Jane Doe",
  "membershipId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "phoneNumber": "+1-555-0144",
  "photoUrl": "https://assets.teacheros.io/photos/jane.jpg"
}
```

**Response:** `201 Created`

**Location Header:** `/api/tenants/{tenantId}/teachers/{teacherId}`

```json
{
  "id": "6a9e102f-b472-4cf3-bfb7-25e229c12df8",
  "membershipId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "fullName": "Jane Doe",
  "phoneNumber": "+1-555-0144",
  "photoUrl": "https://assets.teacheros.io/photos/jane.jpg",
  "status": 1,
  "createdAtUtc": "2026-09-04T09:00:00Z",
  "updatedAtUtc": "2026-09-04T09:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation failure (`Teachers.InvalidInput` — empty name or invalid length constraints).
- `401 Unauthorized` — Unauthenticated.
- `403 Forbidden` — Missing `teachers.manage` permission.

---

#### `PATCH /api/tenants/{tenantId}/teachers/{teacherId}`

Updates a teacher's contact and basic profile information.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.
- `teacherId` (Guid, required): Target teacher ID.

**Request Body:**

```json
{
  "fullName": "Jane Doe-Smith",
  "phoneNumber": "+1-555-0188",
  "photoUrl": "https://assets.teacheros.io/photos/jane-new.jpg"
}
```

**Response:** `200 OK` (returns updated `TeacherResponse`)

**Possible Errors:**
- `400 Bad Request` — Invalid input.
- `401 Unauthorized` — Unauthenticated.
- `403 Forbidden` — Missing permission.
- `404 Not Found` — Teacher not found (`Teachers.NotFound`).

---

### 2.2 Status Management

#### `POST /api/tenants/{tenantId}/teachers/{teacherId}/deactivate`

Sets a teacher's status to `Inactive (2)`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Response:** `200 OK` (returns updated `TeacherResponse` with `status: 2`)

---

#### `POST /api/tenants/{tenantId}/teachers/{teacherId}/reactivate`

Sets a teacher's status to `Active (1)`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Response:** `200 OK` (returns updated `TeacherResponse` with `status: 1`)

---

### 2.3 Subject & Branch Assignments

#### `GET /api/tenants/{tenantId}/teachers/{teacherId}/subjects`

Retrieves an array of subject GUIDs assigned to the teacher.

**Response:** `200 OK`

```json
[
  "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
  "c56a4180-65aa-42ec-a945-5fd21dec0538"
]
```

---

#### `PUT /api/tenants/{tenantId}/teachers/{teacherId}/subjects`

Replaces the full collection of subject assignments for this teacher.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "ids": [
    "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
    "c56a4180-65aa-42ec-a945-5fd21dec0538"
  ]
}
```

**Response:** `200 OK`

```json
[
  "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
  "c56a4180-65aa-42ec-a945-5fd21dec0538"
]
```

**Possible Errors:**
- `400 Bad Request` — One or more subject IDs do not exist or are inactive (`Teachers.SubjectNotFound`).
- `404 Not Found` — Teacher profile not found (`Teachers.NotFound`).

---

#### `GET /api/tenants/{tenantId}/teachers/{teacherId}/branches`

Retrieves an array of branch GUIDs assigned to the teacher.

**Response:** `200 OK`

```json
[
  "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
]
```

---

#### `PUT /api/tenants/{tenantId}/teachers/{teacherId}/branches`

Replaces the full collection of branch assignments for this teacher.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `teachers.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "ids": [
    "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
  ]
}
```

**Response:** `200 OK`

```json
[
  "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
]
```

**Possible Errors:**
- `400 Bad Request` — One or more branch IDs do not exist (`Teachers.BranchNotFound`).
- `404 Not Found` — Teacher profile not found (`Teachers.NotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `TeacherCreateRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `fullName` | `string` | Yes | Non-empty, max 200 characters. Trimmed. |
| `membershipId` | `Guid?` | No | If provided, cannot be `00000000-0000-0000-0000-000000000000`. |
| `phoneNumber` | `string?` | No | Max 30 characters. Trimmed. Set to `null` if empty. |
| `photoUrl` | `string?` | No | Max 2048 characters. Trimmed. Set to `null` if empty. |

#### `TeacherUpdateRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `fullName` | `string` | Yes | Non-empty, max 200 characters. |
| `phoneNumber` | `string?` | No | Max 30 characters. |
| `photoUrl` | `string?` | No | Max 2048 characters. |

#### `TeacherAssignmentsRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `ids` | `Array<Guid>` | Yes | List of GUIDs (Subjects or Branches). Duplicates automatically removed. |

### Response DTOs

#### `TeacherResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Teacher profile identifier. |
| `membershipId` | `Guid?` | Linked user membership ID if attached. |
| `fullName` | `string` | Full name of instructor. |
| `phoneNumber` | `string?` | Phone contact number. |
| `photoUrl` | `string?` | URL to profile avatar. |
| `status` | `integer` (`TeacherStatus`) | `1` = Active, `2` = Inactive. |
| `createdAtUtc` | `string` | ISO 8601 creation timestamp. |
| `updatedAtUtc` | `string` | ISO 8601 last update timestamp. |

#### `TeacherDetailResponse`

Includes all properties of `TeacherResponse` plus:

| Property | Type | Description |
|----------|------|-------------|
| `subjectIds` | `Array<Guid>` | List of assigned subject GUIDs. |
| `branchIds` | `Array<Guid>` | List of assigned branch GUIDs. |

---

## 4. Enum Reference

### `TeacherStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Active` | Teacher is active and available for teaching assignments, schedule creation, and group allocation. |
| `2` | `Inactive` | Teacher is deactivated/suspended. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Full Name Constraints**:
   - `fullName` is required and cannot be whitespace.
   - Max length is **200 characters**.
2. **Contact & URL Constraints**:
   - `phoneNumber` max length is **30 characters**.
   - `photoUrl` max length is **2048 characters**.
3. **Subject Assignment Validation**:
   - Every subject ID in `PUT /subjects` must exist in the tenant and have `status = 1` (`Active`).
   - If any subject is non-existent or `Inactive`, the entire request fails with `Teachers.SubjectNotFound`.
4. **Branch Assignment Validation**:
   - Every branch ID in `PUT /branches` must exist in the tenant.
   - If any branch is non-existent, the request fails with `Teachers.BranchNotFound`.
5. **Replacement Semantics**:
   - Both `PUT /subjects` and `PUT /branches` use full collection replacement semantics (not delta/append). Sending `[]` removes all assignments.

### Frontend Validation Recommendations

- Ensure `fullName` is non-empty before submitting creation/update forms.
- Filter subject selectors by `status = 1` before allowing the user to select them for teacher assignment.
- Validate telephone format and image URL formats on client-side before sending.

---

## 6. Search, Sorting & Pagination

- **Search Filter**: `?search=doe` filters teachers by name or phone.
- **Subject Filter**: `?subjectId={guid}` returns only teachers assigned to that subject.
- **Branch Filter**: `?branchId={guid}` returns only teachers assigned to that branch.
- **Status Filter**: `?status=1` filters active teachers.
- **Pagination**: Default `page=1`, `pageSize=50` (max: 100).

---

## 7. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Teachers.NotFound` | 404 | Teacher profile does not exist. | Navigate back to teacher roster. |
| `Teachers.InvalidInput` | 400 | Required fields missing or exceed length limits. | Correct inputs highlighted in form. |
| `Teachers.Inactive` | 400 | Action disallowed because teacher is inactive. | Reactivate teacher profile first. |
| `Teachers.SubjectNotFound` | 400 | Assigned subject is missing or inactive. | Refresh subjects list and reselect active subjects. |
| `Teachers.BranchNotFound` | 400 | Assigned branch is missing. | Refresh branches list and reselect valid branches. |

---

## 8. Frontend Integration Best Practices

1. **Two-Step Creation Flow**: When designing the Teacher Creation UI:
   - Step 1: Submit `POST /api/tenants/{tenantId}/teachers` with core profile data.
   - Step 2: Use the returned `id` to submit `PUT /subjects` and `PUT /branches` for assignments.
2. **Filtering for Schedules & Groups**: When selecting a teacher for a new group or session, filter teachers by `branchId` and `subjectId` query parameters to display only qualified teachers.
3. **Membership Linking**: Teachers who log into the portal to take attendance or grade homework must have a valid `membershipId` linked to their profile.
