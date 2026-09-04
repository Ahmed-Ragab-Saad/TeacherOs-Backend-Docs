# Groups Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/groups`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Specific permissions: `groups.view` for read operations, `groups.manage` for group lifecycle operations, `groups.enrollments.manage` for enrollment operations.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Group Management](#21-group-management)
   - 2.2 [Group Assignments](#22-group-assignments)
   - 2.3 [Pricing](#23-pricing)
   - 2.4 [Enrollment Management](#24-enrollment-management)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Search, Sorting & Pagination](#6-search-sorting--pagination)
7. [Error Handling Patterns](#7-error-handling-patterns)
8. [Frontend Integration Best Practices](#8-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Groups module manages class groups (cohorts) within a tenant organization. A group represents a scheduled cohort of students assigned to a specific subject, grade level, and branch, optionally taught by a particular teacher. Groups are the primary vehicle for scheduling sessions and recording attendance.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Group** | A named cohort linked to a subject, grade level, and branch. Optionally assigned to a teacher and optionally capped with a capacity. |
| **Group Status** | Lifecycle state: `Active = 1` (available for enrollment/scheduling) or `Archived = 2` (read-only, no new enrollments). |
| **Enrollment** | A student's membership in a group, tracked with an `EnrollmentStatus` (`Active = 1`, `Left = 2`) and timestamps. |
| **Price Per Session** | Optional decimal price that defines the default tuition rate for each attended session. Changes to this value do not affect past `SessionCharge` records (they snapshot the price at attendance time). |
| **Capacity** | Optional maximum number of active students allowed in the group. Enforces enrollment limits. |

### Lifecycle & State Transitions

```
        +----------+
        |  Create  |
        +----+-----+
             |
             v
      +------------+
+---->|  Active(1) |<----+
|    +------------+     | Archive
|                      | (prevents new enrollments)
|    Reactivate        | or scheduling
+----|--------------+
     |
```

### Important Relationships

```
+------------------+     1:N          +-------------------------+
| Subject (Active)  |---------------->|         Group           |
+------------------+                 +-------+-------+----+----+
                                            |       |       |
+------------------+     1:N          +---v---+ +--v---+ +-v---+
| GradeLevel       |---------------->|       | |      | |     |
+------------------+                 | Grade | |Subj. | |Grade|
                                       LevelId  |      | |Level|
+------------------+     1:N          +--------+ +------+ +-----+
| Branch           |---------------->|  Branch| |Branch| |...
+------------------+                 |  Id    | |Id    | |
                                       +--------+ +------+ +-----+
+------------------+     1:0-1        +---------+-------+-------+
| TeacherProfile   |---------------->| Teacher | (optional link) |
+------------------+                 +---------+-----------------+
                                            | 1:N (enrollments)
                                            v
                                   +-------------------+
                                   | StudentGroup      |
                                   | Enrollment         |
                                   +-------------------+
```

### Important Frontend Implications

- **Grade Level Matching**: Only students whose `gradeLevelId` matches the group's `gradeLevelId` are eligible for enrollment.
- **Student Eligibility**: Only `Active` students can be enrolled. Archived/graduated students are excluded.
- **No Duplicate Active Enrollment**: A student cannot be enrolled in the same group if they already have an `Active` enrollment there. They can re-enroll after leaving.
- **Archived Groups Are Read-Only**: Archived groups cannot receive new enrollments or be used for scheduling.
- **Price Per Session Snapshot**: When attendance is recorded for a session, the `SessionCharge` records the group's current `pricePerSession` as a fixed value at that moment. Subsequent price changes do not affect past charges.

---

## 2. API Endpoints Reference

### 2.1 Group Management

#### `GET /api/tenants/{tenantId}/groups`

Lists groups with rich detail including `activeStudentCount`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `groups.view`

**Query Parameters:**
- `search` (string, optional): Substring filter on group name.
- `subjectId` (Guid, optional): Filter by subject.
- `gradeLevelId` (Guid, optional): Filter by grade level.
- `branchId` (Guid, optional): Filter by branch.
- `teacherId` (Guid, optional): Filter by assigned teacher.
- `status` (GroupStatus integer, optional): Filter by `1` (Active) or `2` (Archived).
- `page` (integer, optional, default `1`): Page number (min: 1).
- `pageSize` (integer, optional, default `50`): Items per page (range: 1–100).

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Grade 10 - Mathematics - Section A",
      "subjectId": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
      "gradeLevelId": "f0e1d2c3-b4a5-6789-0abc-def123456789",
      "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
      "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
      "capacity": 30,
      "activeStudentCount": 24,
      "status": 1,
      "createdAtUtc": "2026-09-01T08:00:00Z",
      "updatedAtUtc": "2026-09-01T08:00:00Z"
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 50
}
```

---

#### `POST /api/tenants/{tenantId}/groups`

Creates a new group. Fails with `409 Conflict` if an active group with the same name, subject, grade level, and branch already exists.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `groups.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)
- `X-Tenant-Id` (required)

**Request Body:**

```json
{
  "name": "Grade 10 - Mathematics - Section A",
  "subjectId": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
  "gradeLevelId": "f0e1d2c3-b4a5-6789-0abc-def123456789",
  "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
  "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "capacity": 30
}
```

**Response:** `201 Created`

**Location Header:** `/api/tenants/{tenantId}/groups/{groupId}`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Grade 10 - Mathematics - Section A",
  "subjectId": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
  "gradeLevelId": "f0e1d2c3-b4a5-6789-0abc-def123456789",
  "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
  "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "capacity": 30,
  "pricePerSession": null,
  "status": 1,
  "createdAtUtc": "2026-09-04T10:00:00Z",
  "updatedAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation failure (`Groups.InvalidInput`).
- `404 Not Found` — Subject, grade level, or branch does not exist.
- `409 Conflict` — Active group with same name/subject/grade/branch already exists (`Groups.DuplicateActiveGroup`).

---

#### `GET /api/tenants/{tenantId}/groups/{groupId}`

Retrieves a single group by ID.

**Response:** `200 OK` — `GroupResponse`

**Possible Errors:**
- `404 Not Found` — Group not found (`Groups.NotFound`).

---

#### `PATCH /api/tenants/{tenantId}/groups/{groupId}`

Updates the group's name and/or capacity.

**Request Body:**

```json
{
  "name": "Grade 10 - Mathematics - Section B",
  "capacity": 25
}
```

**Possible Errors:**
- `400 Bad Request` — `capacity` is less than the current `activeStudentCount` (`Groups.InvalidInput`).
- `409 Conflict` — Renaming would create a duplicate active group with same name/subject/grade/branch (`Groups.DuplicateActiveGroup`).

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/archive`

Archives the group. Archived groups cannot accept new enrollments or be scheduled.

**Response:** `200 OK` — `GroupResponse` with `status: 2`

**Possible Errors:**
- `404 Not Found` — Group not found (`Groups.NotFound`).

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/reactivate`

Restores an archived group to `Active`.

**Response:** `200 OK` — `GroupResponse` with `status: 1`

**Possible Errors:**
- `409 Conflict` — Reactivating would create a duplicate active group (`Groups.DuplicateActiveGroup`).

---

### 2.2 Group Assignments

#### `PUT /api/tenants/{tenantId}/groups/{groupId}/teacher`

Assigns (or unassigns) a teacher to the group.

**Request Body:**

```json
{
  "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300"
}
```

Set `teacherProfileId` to `null` to unassign.

**Response:** `200 OK` — Updated `GroupResponse`

**Possible Errors:**
- `400 Bad Request` — Teacher is inactive or not assigned to the group's subject/branch (`Groups.TeacherNotEligible`).

---

#### `PUT /api/tenants/{tenantId}/groups/{groupId}/branch`

Transfers the group to a different branch.

**Request Body:**

```json
{
  "branchId": "b2c3d4e5-f6a7-8901-bcde-f23456789012"
}
```

**Possible Errors:**
- `404 Not Found` — Branch not found (`Groups.BranchNotFound`).
- `409 Conflict` — Transfer would create a duplicate active group with same name/subject/grade/branch at new branch (`Groups.DuplicateActiveGroup`).

---

### 2.3 Pricing

#### `PATCH /api/tenants/{tenantId}/groups/{groupId}/price`

Updates the group's per-session price with an audited reason.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `groups.manage`

**Request Body:**

```json
{
  "pricePerSession": 50.00,
  "reason": "New academic year pricing starting September 2026"
}
```

**Notes:**
- `pricePerSession` must be `null` or strictly positive.
- The `reason` field is recorded in the audit trail (see Audit module).
- Setting `pricePerSession` to `null` makes the group unchargeable.

**Response:** `200 OK` — Updated `GroupResponse`

**Possible Errors:**
- `400 Bad Request` — Price is not positive (`Groups.InvalidInput`).

---

### 2.4 Enrollment Management

#### `GET /api/tenants/{tenantId}/groups/{groupId}/students`

Lists all students currently enrolled or previously enrolled (including `Left`) in the group.

**Authorization:** Permission `groups.view`

**Response:** `200 OK`

```json
[
  {
    "studentId": "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
    "studentCode": "STU-2024-001",
    "fullName": "Ahmed Hassan",
    "nationalId": "1234567890",
    "homeBranchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
    "homeBranchName": "Main Campus",
    "gradeLevelId": "f0e1d2c3-b4a5-6789-0abc-def123456789",
    "gradeLevelName": "Grade 10",
    "joinedAtUtc": "2026-09-05T10:00:00Z",
    "status": 1
  }
]
```

---

#### `GET /api/tenants/{tenantId}/groups/{groupId}/eligible-students`

Lists students eligible for enrollment in this group. Excludes: students already actively enrolled in this group, students with a different `gradeLevelId`, and inactive students.

**Authorization:** Permission `groups.view`

**Query Parameters:**
- `search` (string, optional): Filter by student name or code.
- `page` (integer, optional, default `1`): Page number.
- `pageSize` (integer, optional, default `50`): Items per page (max 100).

**Response:** `200 OK` — Paginated list of `EligibleStudentItem`

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/enrollments`

Enrolls one or more eligible students in the group.

**Authorization:** Permission `groups.enrollments.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "studentIds": [
    "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
    "t2u3v4w5-x6y7-8901-bcde-f23456789012"
  ]
}
```

**Response:** `201 Created`

```json
[
  {
    "id": "e1n2r3o4-l5m6-7890-a1b2-c3d4e5f60789",
    "studentId": "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
    "status": 1,
    "joinedAtUtc": "2026-09-05T10:30:00Z",
    "leftAtUtc": null
  }
]
```

**Possible Errors:**
- `400 Bad Request` — Empty `studentIds` array.
- `404 Not Found` — Group not found (`Groups.NotFound`).
- `409 Conflict` — Enrollment would exceed capacity (`Groups.CapacityExceeded`), or a student is already actively enrolled or ineligible (`Groups.IneligibleStudent`).

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/enrollments/{studentId}/leave`

Marks the student's enrollment as `Left`.

**Authorization:** Permission `groups.enrollments.manage`

**Response:** `204 No Content`

**Possible Errors:**
- `404 Not Found` — Group not found or no active enrollment found (`Groups.NotFound`, `Groups.EnrollmentNotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `GroupCreateRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `name` | `string` | Yes | Non-empty, max 150 characters, unique per subject/grade/branch among active groups. |
| `subjectId` | `Guid` | Yes | Must be an existing active subject. |
| `gradeLevelId` | `Guid` | Yes | Must be an existing grade level. |
| `branchId` | `Guid` | Yes | Must be an existing branch. |
| `teacherProfileId` | `Guid?` | No | If provided, teacher must be active and assigned to subject+branch. |
| `capacity` | `int?` | No | If provided, must be > 0. |

#### `GroupUpdateRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `name` | `string` | Yes | Non-empty, max 150 characters. |
| `capacity` | `int?` | No | If provided, must be >= current active student count. |

#### `GroupTeacherAssignmentRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `teacherProfileId` | `Guid?` | Yes | `null` to unassign; if set, teacher must be active and assigned to subject+branch. |

#### `GroupBranchAssignmentRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `branchId` | `Guid` | Yes | Must exist. Transferring may cause name conflicts. |

#### `GroupPriceUpdateRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `pricePerSession` | `decimal?` | Yes | `null` to clear pricing (makes group unchargeable); if non-null, must be > 0. |
| `reason` | `string` | Yes | Reason for the price change (used in audit logs). |

#### `GroupEnrollmentsRequest`

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `studentIds` | `Array<Guid>` | Yes | Array of student IDs to enroll. Must not be empty. |

### Response DTOs

#### `GroupResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Group identifier. |
| `name` | `string` | Display name of the group. |
| `subjectId` | `Guid` | Associated subject. |
| `gradeLevelId` | `Guid` | Associated grade level. |
| `branchId` | `Guid` | Associated branch. |
| `teacherProfileId` | `Guid?` | Assigned teacher (nullable). |
| `capacity` | `int?` | Maximum active enrollment count (nullable). |
| `pricePerSession` | `decimal?` | Default per-session charge amount (nullable). |
| `status` | `integer` (`GroupStatus`) | `1` = Active, `2` = Archived. |
| `createdAtUtc` | `string` | ISO 8601 creation timestamp. |
| `updatedAtUtc` | `string` | ISO 8601 last update timestamp. |

#### `GroupListItem`

Extends `GroupResponse` with:

| Property | Type | Description |
|----------|------|-------------|
| `activeStudentCount` | `integer` | Count of students with `EnrollmentStatus.Active` currently in the group. |

#### `GroupStudentItem`

| Property | Type | Description |
|----------|------|-------------|
| `studentId` | `Guid` | Student identifier. |
| `studentCode` | `string` | Student's enrollment number. |
| `fullName` | `string` | Full name of the student. |
| `nationalId` | `string` | National ID. |
| `homeBranchId` | `Guid` | Student's home branch. |
| `homeBranchName` | `string` | Home branch display name. |
| `gradeLevelId` | `Guid` | Student's current grade level. |
| `gradeLevelName` | `string` | Grade level display name. |
| `joinedAtUtc` | `string` | When the student enrolled. |
| `status` | `integer` (`EnrollmentStatus`) | `1` = Active, `2` = Left. |

#### `EligibleStudentItem`

| Property | Type | Description |
|----------|------|-------------|
| `studentId` | `Guid` | Student identifier. |
| `studentCode` | `string` | Student's enrollment number. |
| `fullName` | `string` | Full name. |
| `nationalId` | `string` | National ID. |
| `homeBranchId` | `Guid` | Home branch. |
| `homeBranchName` | `string` | Home branch name. |
| `gradeLevelId` | `Guid` | Grade level (matches group). |
| `gradeLevelName` | `string` | Grade level name. |

#### `GroupEnrollmentResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Enrollment record ID. |
| `studentId` | `Guid` | Enrolled student ID. |
| `status` | `integer` (`EnrollmentStatus`) | `1` = Active, `2` = Left. |
| `joinedAtUtc` | `string` | Enrollment timestamp. |
| `leftAtUtc` | `string?` | Null when active; set when student leaves. |

---

## 4. Enum Reference

### `GroupStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Active` | Group is open for enrollment and scheduling. |
| `2` | `Archived` | Group is closed. No new enrollments or scheduling. |

### `EnrollmentStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Active` | Student is currently enrolled. |
| `2` | `Left` | Student has left the group (after calling `POST /enrollments/{studentId}/leave`). |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Group Name Uniqueness**: No two active groups may share the same combination of name + subjectId + gradeLevelId + branchId. Archived groups do not block creation of a new active group with the same attributes.
2. **Capacity Enforcement**: When enrolling, the backend checks that `currentActiveCount + newStudentIds.Count <= capacity`. If the group has no capacity set, enrollment is unlimited.
3. **Grade Level Matching**: Only students with `gradeLevelId` matching the group's `gradeLevelId` can be enrolled. Students with different grade levels are ineligible.
4. **Student Status**: Only `StudentStatus.Active` students can be enrolled. Suspended or graduated students are excluded from eligibility.
5. **No Duplicate Active Enrollment**: A student cannot be enrolled in a group if they already have an `Active` enrollment in that group. They must first `leave` before re-enrolling.
6. **Archived Group Restrictions**: Archived groups reject new enrollment operations.
7. **Teacher Eligibility**: When assigning a teacher to a group, the teacher must be active and must have both the group's subject and branch in their assignment lists. Passing `null` unassigns the teacher without eligibility checks.
8. **Price Must Be Positive**: `pricePerSession` must be `null` or > 0.
9. **Capacity Must Be Positive**: If set, `capacity` must be > 0.
10. **Capacity Cannot Shrink Below Active Count**: `PATCH /groups/{groupId}` with a new `capacity` must provide a value >= the current active student count.

### Frontend Validation Recommendations

- Before calling `EnrollStudentsAsync`, fetch eligible students with `GET /eligible-students` so the UI only presents valid options.
- When rendering the group creation form, pre-filter subjects by `status = 1` and populate branch/grade-level dropdowns from their respective endpoints.
- Show capacity usage visually (e.g., "24/30 students") by combining `capacity` and `activeStudentCount` from `GroupListItem`.

---

## 6. Search, Sorting & Pagination

- **Search**: `?search=math` filters groups by name.
- **Filters**: `subjectId`, `gradeLevelId`, `branchId`, `teacherId`, `status`.
- **Pagination**: Default `page=1`, `pageSize=50` (max: 100).
- **Eligible Students**: `?search=john` filters by name or student code; paginated (max 100).

---

## 7. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Groups.NotFound` | 404 | Group does not exist. | Navigate back to group list. |
| `Groups.InvalidInput` | 400 | Missing required fields, name too long, or capacity invalid. | Correct highlighted fields in form. |
| `Groups.SubjectNotFound` | 404 | Subject does not exist or is inactive. | Refresh subjects list and reselect. |
| `Groups.GradeLevelNotFound` | 404 | Grade level does not exist. | Refresh grade levels and reselect. |
| `Groups.BranchNotFound` | 404 | Branch does not exist. | Refresh branches and reselect. |
| `Groups.TeacherNotEligible` | 400 | Teacher is inactive or not assigned to subject+branch. | Assign the teacher to the subject and branch in the Teachers module first. |
| `Groups.DuplicateActiveGroup` | 409 | Another active group has the same name/subject/grade/branch. | Rename or choose a different combination. |
| `Groups.InactiveGroup` | 400 | Operation on an archived group. | Reactivate the group first. |
| `Groups.CapacityExceeded` | 409 | Enrollment would exceed the group's capacity. | Reduce enrollment count or increase group capacity. |
| `Groups.IneligibleStudent` | 400/409 | Student is already enrolled, inactive, or wrong grade level. | Use the "eligible students" list for valid choices. |
| `Groups.EnrollmentNotFound` | 404 | No active enrollment found for the student. | Student may have already left the group. |

---

## 8. Frontend Integration Best Practices

1. **Group Creation Wizard**: Guide users through: (a) name → (b) subject selection → (c) grade level → (d) branch → (e) optional teacher → (f) optional capacity.
2. **Capacity Dashboard**: Display group cards with a capacity progress bar (active students / capacity) using `activeStudentCount` and `capacity`.
3. **Enrollment Dialog**: When enrolling, fetch `GET /eligible-students` and present them in a searchable, paginated table. Perform optimistic removal of selected students from the list on the client side.
4. **Price Update Audit**: Always prompt the user to enter a meaningful `reason` before calling `PATCH /price`. Display the reason in the group's history or audit log.
5. **Branch Transfer Warning**: Changing a group's branch may cause `Groups.DuplicateActiveGroup` conflicts if another active group with the same name exists at the destination. Warn users before the transfer.
6. **Teacher Assignment Flow**: Before assigning a teacher, the frontend should verify the teacher has both the group's subject and branch in their assignments. The backend will also enforce this, but better UX is to validate pre-emptively.
