# Students Module — Frontend Integration Guide

**Base URLs:** 
- `/api/tenants/{tenantId}/students`
- `/api/tenants/{tenantId}/branches`
- `/api/tenants/{tenantId}/grade-levels`
- `/api/tenants/{tenantId}/guardians`

**Role Requirement:** Tenant context required (specific permissions vary by endpoint)

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Students module manages student records, organizational structures (branches and grade levels), and guardian relationships within a tenant's educational system.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Student** | Core entity representing an enrolled student with identification, academic placement, and contact information. |
| **Branch** | Physical location or campus where students are enrolled (e.g., "Main Campus", "North Branch"). |
| **Grade Level** | Academic level classification (e.g., "Grade 1", "Year 9", "Kindergarten"). Has a sort order for proper sequencing. |
| **Guardian** | Parent or legal guardian associated with students. One guardian can be linked to multiple students. |
| **Student-Guardian Link** | Relationship connecting a student to a guardian with relationship type (Father, Mother, Guardian, Other) and primary contact designation. |
| **Student Status** | Lifecycle state: Active, SuspendedAdministrative, SuspendedNonPayment, Graduated. |

### Important Frontend Implications

- **Setup sequence**: Create branches and grade levels before creating students (students require valid branch and grade level assignments).
- **Guardian management**: Guardians are created independently and then linked to students. One guardian can be linked to multiple students (e.g., siblings).
- **Status transitions**: Students can be suspended administratively, suspended for non-payment, reactivated, or graduated. These are separate endpoints with specific business rules.
- **Student codes**: Each student has a unique `studentCode` within the tenant (e.g., enrollment number, student ID).
- **National ID**: Students have a `nationalId` field for government identification (required in many educational systems).

---

## 2. API Endpoints Reference

### 2.1 Branches

#### `GET /api/tenants/{tenantId}/branches`
Lists all branches for the tenant.

**Response:** `200 OK` - Array of `BranchResponse`

#### `POST /api/tenants/{tenantId}/branches`
Creates a new branch.

**Request Body:** `{ "name": "Main Campus" }`

**Response:** `201 Created` - `BranchResponse`

**Errors:** `409 Conflict` if name already exists.

#### `GET /api/tenants/{tenantId}/branches/{branchId}`
Retrieves a specific branch.

**Response:** `200 OK` - `BranchResponse`

#### `PATCH /api/tenants/{tenantId}/branches/{branchId}`
Updates a branch name.

**Request Body:** `{ "name": "Updated Name" }`

**Response:** `200 OK` - `BranchResponse`

---

### 2.2 Grade Levels

#### `GET /api/tenants/{tenantId}/grade-levels`
Lists all grade levels for the tenant, ordered by `sortOrder`.

**Response:** `200 OK` - Array of `GradeLevelResponse`

#### `POST /api/tenants/{tenantId}/grade-levels`
Creates a new grade level.

**Request Body:** `{ "name": "Grade 1", "sortOrder": 1 }`

**Response:** `201 Created` - `GradeLevelResponse`

**Errors:** `409 Conflict` if name already exists.

#### `GET /api/tenants/{tenantId}/grade-levels/{gradeLevelId}`
Retrieves a specific grade level.

**Response:** `200 OK` - `GradeLevelResponse`

#### `PATCH /api/tenants/{tenantId}/grade-levels/{gradeLevelId}`
Updates a grade level.

**Request Body:** `{ "name": "Updated Name", "sortOrder": 2 }`

**Response:** `200 OK` - `GradeLevelResponse`

---

### 2.3 Students

#### `GET /api/tenants/{tenantId}/students`
Lists all students for the tenant.

**Response:** `200 OK` - Array of `StudentListResponse` (includes branch and grade level names)

#### `POST /api/tenants/{tenantId}/students`
Creates a new student.

**Request Body:**
```json
{
  "studentCode": "2024001",
  "fullName": "John Doe",
  "nationalId": "1234567890",
  "branchId": "uuid",
  "gradeLevelId": "uuid",
  "enrollmentDate": "2024-09-01",
  "phoneNumber": "+1234567890",
  "photoUrl": "https://..."
}
```

**Response:** `201 Created` - `StudentResponse`

**Errors:** 
- `404 Not Found` if branch or grade level doesn't exist
- `409 Conflict` if student code already exists

#### `GET /api/tenants/{tenantId}/students/{studentId}`
Retrieves a specific student.

**Response:** `200 OK` - `StudentResponse`

#### `PATCH /api/tenants/{tenantId}/students/{studentId}`
Updates student information (excludes status changes).

**Request Body:** Same as create, minus `studentCode` (codes are immutable)

**Response:** `200 OK` - `StudentResponse`

#### `PUT /api/tenants/{tenantId}/students/{studentId}/branch`
Assigns a student to a different branch.

**Request Body:** `{ "branchId": "uuid" }`

**Response:** `200 OK` - `StudentResponse`

#### `PUT /api/tenants/{tenantId}/students/{studentId}/grade-level`
Promotes/assigns a student to a different grade level.

**Request Body:** `{ "gradeLevelId": "uuid" }`

**Response:** `200 OK` - `StudentResponse`

---

### 2.4 Student Status Management

#### `POST /api/tenants/{tenantId}/students/{studentId}/suspensions/administrative`
Suspends a student for administrative reasons (behavior, disciplinary).

**Response:** `200 OK` - `StudentResponse` with `status: "SuspendedAdministrative"`

**Errors:** `409 Conflict` if already in that status.

#### `POST /api/tenants/{tenantId}/students/{studentId}/suspensions/non-payment`
Suspends a student for non-payment of fees.

**Response:** `200 OK` - `StudentResponse` with `status: "SuspendedNonPayment"`

#### `POST /api/tenants/{tenantId}/students/{studentId}/reactivation`
Reactivates a suspended student.

**Response:** `200 OK` - `StudentResponse` with `status: "Active"`

#### `POST /api/tenants/{tenantId}/students/{studentId}/graduation`
Marks a student as graduated.

**Response:** `200 OK` - `StudentResponse` with `status: "Graduated"`

**Note:** Graduation is typically irreversible.

---

### 2.5 Guardians

#### `GET /api/tenants/{tenantId}/guardians`
Lists all guardians for the tenant.

**Response:** `200 OK` - Array of `GuardianResponse`

#### `POST /api/tenants/{tenantId}/guardians`
Creates a new guardian.

**Request Body:** `{ "fullName": "Jane Doe", "phoneNumber": "+1234567890" }`

**Response:** `201 Created` - `GuardianResponse`

#### `GET /api/tenants/{tenantId}/guardians/{guardianId}`
Retrieves a specific guardian.

**Response:** `200 OK` - `GuardianResponse`

#### `PATCH /api/tenants/{tenantId}/guardians/{guardianId}`
Updates guardian information.

**Request Body:** `{ "fullName": "Updated Name", "phoneNumber": "+9876543210" }`

**Response:** `200 OK` - `GuardianResponse`

---

### 2.6 Student-Guardian Links

#### `GET /api/tenants/{tenantId}/students/{studentId}/guardians`
Lists all guardians linked to a specific student.

**Response:** `200 OK` - Array of `StudentGuardianResponse`

#### `POST /api/tenants/{tenantId}/students/{studentId}/guardians`
Links an existing guardian to a student.

**Request Body:**
```json
{
  "guardianId": "uuid",
  "relationshipType": "Father",
  "isPrimaryContact": true
}
```

**Response:** `201 Created` - `StudentGuardianResponse`

**Errors:** `409 Conflict` if guardian already linked to student.

#### `PATCH /api/tenants/{tenantId}/students/{studentId}/guardians/{guardianId}`
Updates the relationship details.

**Request Body:**
```json
{
  "relationshipType": "Mother",
  "isPrimaryContact": false
}
```

**Response:** `200 OK` - `StudentGuardianResponse`

#### `POST /api/tenants/{tenantId}/students/{studentId}/guardians/{guardianId}/unlink`
Removes the guardian link from the student.

**Response:** `204 No Content`

---

## 3. Request & Response DTOs

### BranchWriteRequest
- `name` (string, required): Branch name

### BranchResponse
- `id` (Guid)
- `name` (string)

### GradeLevelWriteRequest
- `name` (string, required): Grade level name
- `sortOrder` (int, required): Sort order for display

### GradeLevelResponse
- `id` (Guid)
- `name` (string)
- `sortOrder` (int)

### StudentCreateRequest
- `studentCode` (string, required): Unique student identifier
- `fullName` (string, required)
- `nationalId` (string, required): Government ID
- `branchId` (Guid, required)
- `gradeLevelId` (Guid, required)
- `enrollmentDate` (DateOnly, required)
- `phoneNumber` (string, optional)
- `photoUrl` (string, optional)

### StudentUpdateRequest
Same as create, minus `studentCode`

### StudentResponse
- `id` (Guid)
- `studentCode` (string)
- `fullName` (string)
- `nationalId` (string)
- `branchId` (Guid)
- `gradeLevelId` (Guid)
- `status` (StudentStatus enum)
- `enrollmentDate` (DateOnly)
- `phoneNumber` (string, nullable)
- `photoUrl` (string, nullable)

### StudentListResponse
Same as `StudentResponse` plus:
- `branchName` (string)
- `gradeLevelName` (string)

### GuardianWriteRequest
- `fullName` (string, required)
- `phoneNumber` (string, required)

### GuardianResponse
- `id` (Guid)
- `fullName` (string)
- `phoneNumber` (string)

### StudentGuardianCreateRequest
- `guardianId` (Guid, required)
- `relationshipType` (GuardianRelationshipType, required)
- `isPrimaryContact` (bool, required)

### StudentGuardianResponse
- `guardianId` (Guid)
- `relationshipType` (GuardianRelationshipType)
- `isPrimaryContact` (bool)

---

## 4. Enum Reference

### StudentStatus
| Value | Integer | Description |
|-------|---------|-------------|
| `Active` | 1 | Student is actively enrolled |
| `SuspendedAdministrative` | 2 | Suspended for administrative/disciplinary reasons |
| `SuspendedNonPayment` | 3 | Suspended due to non-payment of fees |
| `Graduated` | 4 | Student has graduated |

### GuardianRelationshipType
| Value | Integer | Description |
|-------|---------|-------------|
| `Father` | 1 | Father |
| `Mother` | 2 | Mother |
| `Guardian` | 3 | Legal guardian |
| `Other` | 4 | Other relationship |

---

## 5. Validation & Business Rules

### Student Code Uniqueness
- `studentCode` must be unique within the tenant.
- Once created, student codes cannot be changed.

### Branch and Grade Level Assignment
- Students must be assigned to valid branches and grade levels.
- Creating a student with invalid IDs returns 404 Not Found.

### Status Transitions
- Students start as `Active` upon creation.
- Status changes are done through dedicated endpoints (not via update).
- Suspended students can be reactivated.
- Graduated students typically cannot be reactivated (business rule).

### Guardian Linking
- A guardian must exist before linking to a student.
- A guardian cannot be linked to the same student twice.
- Only one guardian per student should be marked as `isPrimaryContact: true` (recommended frontend validation, not enforced by backend).

### National ID
- Required field for most educational systems.
- Validation format depends on tenant location (not enforced by this API).

---

## 6. Error Handling Patterns

### Common Error Codes
| Code | Status | Description |
|------|--------|-------------|
| `Students.NotFound` | 404 | Student, branch, grade level, or guardian not found |
| `Students.DuplicateCode` | 409 | Student code already exists |
| `Students.DuplicateName` | 409 | Branch or grade level name already exists |
| `Students.InvalidStatus` | 409 | Invalid status transition |
| `Students.GuardianAlreadyLinked` | 409 | Guardian already linked to student |

---

## 7. Frontend Integration Best Practices

### Setup Workflow
1. **Initial tenant setup:** Create branches first, then grade levels.
2. **Student enrollment:** Create guardian(s), create student, link guardian(s) to student.

### Student Management Page
- Display filters for branch, grade level, and status.
- Show primary guardian contact in student list for quick access.
- Provide bulk actions for common operations (e.g., promote all Grade 1 students to Grade 2).

### Status Management
- Show status badges with color coding (green=Active, yellow=Suspended, blue=Graduated).
- Provide action buttons based on current status:
  - Active: "Suspend (Admin)", "Suspend (Non-Payment)", "Graduate"
  - Suspended: "Reactivate"
  - Graduated: No actions (final state)

### Guardian Management
- Allow searching existing guardians before creating new ones (to avoid duplicates for siblings).
- Show all students linked to a guardian when viewing guardian details.
- Warn before unlinking the last guardian from a student.

---

## Summary

The Students module provides comprehensive student lifecycle management including:
- Organizational structures (branches, grade levels)
- Student CRUD with status transitions
- Guardian management and student-guardian relationships
- All mutations require antiforgery tokens and tenant context
