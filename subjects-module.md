# Subjects Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/subjects`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Specific permissions: `subjects.view` for read operations, `subjects.manage` for write operations.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Subjects](#21-list-subjects)
   - 2.2 [Get Subject by ID](#22-get-subject-by-id)
   - 2.3 [Create Subject](#23-create-subject)
   - 2.4 [Update Subject](#24-update-subject)
   - 2.5 [Deactivate Subject](#25-deactivate-subject)
   - 2.6 [Reactivate Subject](#26-reactivate-subject)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Search, Sorting & Pagination](#6-search-sorting--pagination)
7. [Error Handling Patterns](#7-error-handling-patterns)
8. [Frontend Integration Best Practices](#8-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Subjects module manages the academic subject catalog (e.g., Mathematics, Physics, English) within a specific tenant organization. Subjects are foundational entities referenced by teachers, groups, courses, homework, exams, and the question bank.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Subject** | Academic discipline or course topic configured within a tenant (e.g., "Biology", "Algebra I"). |
| **Normalized Name** | Upper-cased and trimmed name used for case-insensitive uniqueness checking per tenant. |
| **Subject Status** | Lifecycle state indicating whether the subject is currently operational (`Active = 1`) or archived (`Inactive = 2`). |
| **Tenant Isolation** | All subject operations are strictly scoped to the tenant specified in the route and `X-Tenant-Id` header. |

### Lifecycle & State Transitions

- **Creation**: When created, a subject defaults to `Active (1)`.
- **Deactivation (`POST /deactivate`)**: Moves an active subject to `Inactive (2)`. Inactive subjects remain in the database for historical reporting and audit integrity but cannot be assigned to new active groups or curriculum items.
- **Reactivation (`POST /reactivate`)**: Restores an inactive subject to `Active (1)`.

```
           +------------------+
           |      Create      |
           +--------+---------+
                    |
                    v
             +--------------+
      +----->|  Active (1)  |------+
      |      +--------------+      |
  Reactivate                    Deactivate
      |      +--------------+      |
      +------| Inactive (2) |<-----+
             +--------------+
```

### Important Frontend Implications

- **Setup Dependency**: Subjects should be defined early in tenant onboarding as Teachers, Groups, and Learning modules link directly to `SubjectId`.
- **Unique Naming**: Subject names must be unique within a tenant (case-insensitive). Renaming checks for conflicts against other existing subjects.
- **CSRF Protection**: All mutating operations (`POST`, `PATCH`) require the `X-CSRF-TOKEN` header and cookie.
- **Tenant Context**: The `X-Tenant-Id` header must match the `{tenantId}` path parameter.

---

## 2. API Endpoints Reference

### 2.1 List Subjects

#### `GET /api/tenants/{tenantId}/subjects`

Retrieves a paginated list of subjects for the tenant with optional search and status filtering.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.view`

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.

**Query Parameters:**
- `search` (string, optional): Search term to filter subjects by name (case-insensitive substring match).
- `status` (SubjectStatus integer/string, optional): Filter by status (`1` = Active, `2` = Inactive).
- `page` (integer, optional, default `1`): 1-based page index (minimum: 1).
- `pageSize` (integer, optional, default `50`): Number of records per page (range: 1 to 100).

**Request Body:** None

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
      "name": "Mathematics",
      "status": 1,
      "createdAtUtc": "2026-09-01T10:00:00Z",
      "updatedAtUtc": "2026-09-01T10:00:00Z"
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 50
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.view` permission or tenant access denied.

---

### 2.2 Get Subject by ID

#### `GET /api/tenants/{tenantId}/subjects/{subjectId}`

Retrieves details of a specific subject by ID.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.view`

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.
- `subjectId` (Guid, required): The subject ID.

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
  "name": "Mathematics",
  "status": 1,
  "createdAtUtc": "2026-09-01T10:00:00Z",
  "updatedAtUtc": "2026-09-01T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.view` permission or tenant access denied.
- `404 Not Found` — Subject not found (`Subjects.NotFound`).

---

### 2.3 Create Subject

#### `POST /api/tenants/{tenantId}/subjects`

Creates a new academic subject.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token.
- `X-Tenant-Id` (required): Active tenant identifier.

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.

**Request Body:**

```json
{
  "name": "Physics"
}
```

**Response:** `201 Created`

**Location Header:** `/api/tenants/{tenantId}/subjects/{subjectId}`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "name": "Physics",
  "status": 1,
  "createdAtUtc": "2026-09-04T08:00:00Z",
  "updatedAtUtc": "2026-09-04T08:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation failed (`Subjects.InvalidInput` — empty name or exceeds 150 characters).
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.manage` permission or tenant mismatch.
- `409 Conflict` — Subject name already exists in tenant (`Subjects.NameExists`).

---

### 2.4 Update Subject

#### `PATCH /api/tenants/{tenantId}/subjects/{subjectId}`

Updates the name of an existing subject.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token.

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.
- `subjectId` (Guid, required): The subject ID.

**Request Body:**

```json
{
  "name": "Advanced Physics"
}
```

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "name": "Advanced Physics",
  "status": 1,
  "createdAtUtc": "2026-09-04T08:00:00Z",
  "updatedAtUtc": "2026-09-04T08:15:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Invalid name (`Subjects.InvalidInput`).
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.manage` permission.
- `404 Not Found` — Subject not found (`Subjects.NotFound`).
- `409 Conflict` — New name duplicates another existing subject (`Subjects.NameExists`).

---

### 2.5 Deactivate Subject

#### `POST /api/tenants/{tenantId}/subjects/{subjectId}/deactivate`

Marks a subject as `Inactive (2)`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token.

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.
- `subjectId` (Guid, required): The subject ID.

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "name": "Advanced Physics",
  "status": 2,
  "createdAtUtc": "2026-09-04T08:00:00Z",
  "updatedAtUtc": "2026-09-04T08:30:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.manage` permission.
- `404 Not Found` — Subject not found (`Subjects.NotFound`).

---

### 2.6 Reactivate Subject

#### `POST /api/tenants/{tenantId}/subjects/{subjectId}/reactivate`

Restores an inactive subject to `Active (1)`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `subjects.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token.

**Path Parameters:**
- `tenantId` (Guid, required): The target tenant ID.
- `subjectId` (Guid, required): The subject ID.

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "name": "Advanced Physics",
  "status": 1,
  "createdAtUtc": "2026-09-04T08:00:00Z",
  "updatedAtUtc": "2026-09-04T08:35:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `subjects.manage` permission.
- `404 Not Found` — Subject not found (`Subjects.NotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `SubjectWriteRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `name` | `string` | Yes | Non-empty, max 150 characters, trimmed of surrounding whitespace. |

### Response DTOs

#### `SubjectResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` (UUID) | Unique identifier of the subject. |
| `name` | `string` | Subject display name. |
| `status` | `integer` (`SubjectStatus`) | Status enum: `1` (Active), `2` (Inactive). |
| `createdAtUtc` | `string` (ISO 8601 UTC) | Timestamp when the subject was created. |
| `updatedAtUtc` | `string` (ISO 8601 UTC) | Timestamp when the subject was last modified. |

#### `PagedResponse<T>`

| Property | Type | Description |
|----------|------|-------------|
| `items` | `Array<T>` | Page of subject items (`SubjectResponse`). |
| `totalCount` | `integer` | Total number of items matching filter criteria. |
| `page` | `integer` | Current 1-based page number. |
| `pageSize` | `integer` | Current page size. |

---

## 4. Enum Reference

### `SubjectStatus`

| Numeric Value | Name | Description |
|---------------|------|-------------|
| `1` | `Active` | Subject is active and available for teaching, scheduling, and enrollment. |
| `2` | `Inactive` | Subject is archived/deactivated. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Name Constraints**:
   - `name` cannot be empty or pure whitespace.
   - `name` maximum length is **150 characters**.
   - Names are trimmed before saving.
2. **Name Uniqueness**:
   - Subject names are normalized (`ToUpperInvariant()`) and must be unique within the tenant.
   - Attempting to create or rename to an existing normalized name results in `409 Conflict` (`Subjects.NameExists`).
3. **Tenant Scoping**:
   - Cross-tenant access is rejected with `403 Forbidden` (`Tenancy.AccessDenied`).
4. **Pagination Bounds**:
   - Default `page` is `1`. Values `< 1` are coerced to `1`.
   - Default `pageSize` is `50`. Values `< 1` default to `10`. Values `> 100` are clamped to `100`.

### Frontend Validation Recommendations

- Pre-validate that `name` is not blank and is $\le 150$ characters before making API requests.
- Provide a debounced search input (e.g. 300ms delay) when filtering by subject name.
- When rendering dropdown selectors for scheduling or course creation, filter the subjects list by `status = 1` (`Active`).

---

## 6. Search, Sorting & Pagination

- **Search**: `?search=math` performs a case-insensitive `EF.Functions.ILike` or contains search on the subject name.
- **Status Filter**: `?status=1` returns active subjects; `?status=2` returns inactive subjects; omitting returns all.
- **Ordering**: Server-side subjects are ordered by `Name` ascending by default in query listings.
- **Pagination**: Supports `page` (default 1) and `pageSize` (default 50, max 100).

---

## 7. Error Handling Patterns

Error responses follow the RFC 7807 `ProblemDetails` standard format:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.8",
  "title": "Conflict",
  "status": 409,
  "detail": "A subject with the same name already exists in this tenant.",
  "extensions": {
    "code": "Subjects.NameExists"
  }
}
```

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Subjects.NotFound` | 404 | Subject does not exist or has been deleted. | Redirect to subject list or show item not found message. |
| `Subjects.NameExists` | 409 | A subject with this name already exists in tenant. | Ask user to choose a different subject name. |
| `Subjects.InvalidInput` | 400 | Subject name is empty or exceeds 150 characters. | Show inline field validation error. |
| `Subjects.Inactive` | 400 | Attempted operation on an inactive subject. | Prompt user that subject must be reactivated first. |
| `Tenancy.AccessDenied` | 403 | Tenant context does not match user access. | Verify `X-Tenant-Id` selection. |

---

## 8. Frontend Integration Best Practices

1. **Active Subject Selectors**: When populating dropdowns for Teachers, Groups, Courses, or Question Banks, pass `?status=1` so only active subjects are selectable.
2. **Catalog Management View**: Provide tabs or filters for "All", "Active", and "Inactive" subjects so administrators can reactivate archived subjects easily.
3. **Optimistic Updates**: When renaming or toggling status, update the UI optimistically and roll back on error.
4. **Antiforgery Header**: Ensure all `POST` and `PATCH` requests include the `X-CSRF-TOKEN` header and `X-Tenant-Id` header.
