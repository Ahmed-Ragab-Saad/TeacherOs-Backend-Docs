# Courses Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/courses`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permissions vary by operation: `courses.view` for read, `courses.manage` for create/update/delete, `courses.publish` for publishing.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Courses](#21-list-courses)
   - 2.2 [Create Course](#22-create-course)
   - 2.3 [Get Course Details](#23-get-course-details)
   - 2.4 [Update Course](#24-update-course)
   - 2.5 [Publish Course](#25-publish-course)
   - 2.6 [Archive Course](#26-archive-course)
   - 2.7 [Create Section](#27-create-section)
   - 2.8 [Update Section](#28-update-section)
   - 2.9 [Reorder Sections](#29-reorder-sections)
   - 2.10 [Create Lesson](#210-create-lesson)
   - 2.11 [Update Lesson](#211-update-lesson)
   - 2.12 [Reorder Lessons](#212-reorder-lessons)
   - 2.13 [Assign Groups to Course](#213-assign-groups-to-course)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Courses module provides the hierarchical structure for organizing learning content. Courses contain Sections, which contain Lessons. This structure supports multiple content types (video, text, quiz) and can be associated with specific student groups for targeted delivery.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Course** | Top-level container. Has a status lifecycle: Draft → Published → Archived. |
| **Section** | Groups related lessons within a course. Can be reordered independently. |
| **Lesson** | Leaf-level content unit within a section. Has a type (Video, Text, Quiz). |
| **Course Status** | Draft courses are editable; Published courses are visible to assigned groups; Archived courses are hidden. |
| **Group Assignment** | Courses can be assigned to one or more groups, controlling which students see the course content. |
| **Subject/GradeLevel** | Optional taxonomy tags for organizing courses by curriculum area. |

### Hierarchy

```
Course
├── Section 1
│   ├── Lesson 1.1
│   ├── Lesson 1.2
│   └── ...
├── Section 2
│   ├── Lesson 2.1
│   └── ...
└── ...
```

### Data Flow

```
CreateCourse → Draft
       |
       v
Add Sections (CreateSection)
       |
       v
Add Lessons to Sections (CreateLesson)
       |
       v
(Optional) Assign Groups (AssignGroups)
       |
       v
PublishCourse → Published
       |
       v
Students see course in their enrolled courses list
```

### Important Frontend Implications

- **Draft Until Published**: New courses are created in Draft status. Content changes are allowed until published.
- **Group-Based Visibility**: Students only see courses assigned to their groups. The course must be Published AND assigned to a group.
- **Section/Lesson Reordering**: The `ReorderSections` and `ReorderLessons` endpoints accept a full list of IDs in the desired order — not individual moves.
- **No Delete Endpoint**: Courses use Archive (soft delete) rather than hard delete. Archived courses cannot be recovered.

---

## 2. API Endpoints Reference

### 2.1 List Courses

#### `GET /api/tenants/{tenantId}/courses`

Retrieves all courses for the tenant with summary information.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.view`

**Path Parameters:**
- `tenantId` (Guid, required)

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
[
  {
    "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
    "title": "Grade 11 Mathematics",
    "description": "Complete mathematics curriculum for Grade 11",
    "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
    "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg",
    "status": "Published",
    "sectionCount": 8,
    "publishedAtUtc": "2026-08-15T10:00:00Z",
    "updatedAtUtc": "2026-09-01T14:30:00Z"
  },
  {
    "id": "d67b5291-7bb1-43f3-b156-6ae32de06449",
    "title": "Physics Fundamentals",
    "description": null,
    "subjectId": "c3d4e5f6-a7b8-9012-c3d4-e5f607890123",
    "gradeLevelId": null,
    "coverImageUrl": null,
    "status": "Draft",
    "sectionCount": 3,
    "publishedAtUtc": null,
    "updatedAtUtc": "2026-09-02T09:15:00Z"
  }
]
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.view` permission or tenant access denied (`Tenancy.AccessDenied`).

---

### 2.2 Create Course

#### `POST /api/tenants/{tenantId}/courses`

Creates a new course in Draft status. Use `PublishCourse` to make it visible to students.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Grade 11 Mathematics",
  "description": "Complete mathematics curriculum for Grade 11",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `string` | Yes | Course title. Max 200 characters. |
| `description` | `string?` | No | Detailed description. Max 2000 characters. |
| `subjectId` | `Guid?` | No | Reference to a subject for categorization. |
| `gradeLevelId` | `Guid?` | No | Reference to a grade level for filtering. |
| `coverImageUrl` | `string?` | No | URL to course cover image. Max 500 characters. |

**Response:** `201 Created`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "title": "Grade 11 Mathematics",
  "description": "Complete mathematics curriculum for Grade 11",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg",
  "status": "Draft",
  "sectionCount": 0,
  "publishedAtUtc": null,
  "updatedAtUtc": "2026-09-04T10:00:00Z"
}
```

**Location Header:** `/api/tenants/{tenantId}/courses/c56a4180-65aa-42ec-a945-5fd21dec0538`

**Possible Errors:**
- `400 Bad Request` — Validation error (`Course.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `409 Conflict` — Duplicate course name within tenant (`Course.DuplicateName`).

---

### 2.3 Get Course Details

#### `GET /api/tenants/{tenantId}/courses/{courseId}`

Retrieves full course details including all sections and lessons.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.view`

**Path Parameters:**
- `tenantId` (Guid, required)
- `courseId` (Guid, required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "title": "Grade 11 Mathematics",
  "description": "Complete mathematics curriculum for Grade 11",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg",
  "status": "Published",
  "publishedAtUtc": "2026-08-15T10:00:00Z",
  "sections": [
    {
      "id": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "title": "Algebra Basics",
      "description": "Introduction to algebraic expressions",
      "displayOrder": 1,
      "status": "Published",
      "lessons": [
        {
          "id": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
          "title": "Variables and Expressions",
          "lessonType": "Video",
          "durationMinutes": 15,
          "displayOrder": 1,
          "status": "Published"
        },
        {
          "id": "l2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
          "title": "Solving Simple Equations",
          "lessonType": "Text",
          "durationMinutes": 10,
          "displayOrder": 2,
          "status": "Published"
        }
      ]
    },
    {
      "id": "s2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "title": "Functions",
      "description": null,
      "displayOrder": 2,
      "status": "Draft",
      "lessons": []
    }
  ],
  "createdAtUtc": "2026-08-01T08:00:00Z",
  "updatedAtUtc": "2026-09-01T14:30:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.view` permission or tenant access denied.
- `404 Not Found` — Course not found (`Course.NotFound`).

---

### 2.4 Update Course

#### `PUT /api/tenants/{tenantId}/courses/{courseId}`

Updates course metadata. Only allowed for Draft courses.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Grade 11 Advanced Mathematics",
  "description": "Updated description for the course",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11-v2.jpg"
}
```

All fields are optional. Only provided fields are updated.

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "title": "Grade 11 Advanced Mathematics",
  "description": "Updated description for the course",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11-v2.jpg",
  "status": "Draft",
  "sectionCount": 8,
  "publishedAtUtc": null,
  "updatedAtUtc": "2026-09-04T11:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error (`Course.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course not found (`Course.NotFound`).
- `409 Conflict` — Duplicate course name (`Course.DuplicateName`) or course is not in Draft status (`Course.InvalidStatusTransition`).

---

### 2.5 Publish Course

#### `POST /api/tenants/{tenantId}/courses/{courseId}/publish`

Publishes a Draft course, making it visible to assigned groups. Cannot be undone — use Archive instead.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.publish`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "title": "Grade 11 Mathematics",
  "description": "Complete mathematics curriculum for Grade 11",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg",
  "status": "Published",
  "sectionCount": 8,
  "publishedAtUtc": "2026-09-04T11:00:00Z",
  "updatedAtUtc": "2026-09-04T11:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.publish` permission or tenant access denied.
- `404 Not Found` — Course not found (`Course.NotFound`).
- `409 Conflict` — Course is not in Draft status (`Course.InvalidStatusTransition`).

---

### 2.6 Archive Course

#### `POST /api/tenants/{tenantId}/courses/{courseId}/archive`

Archives a Published course, hiding it from students. Archived courses are preserved but cannot be edited or republished.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "title": "Grade 11 Mathematics",
  "description": "Complete mathematics curriculum for Grade 11",
  "subjectId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "gradeLevelId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
  "coverImageUrl": "https://cdn.teacheros.com/courses/math-11.jpg",
  "status": "Archived",
  "sectionCount": 8,
  "publishedAtUtc": "2026-08-15T10:00:00Z",
  "updatedAtUtc": "2026-09-04T12:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course not found (`Course.NotFound`).
- `409 Conflict` — Course is Archived already (`Course.CannotDeletePublishedCourse`).

---

### 2.7 Create Section

#### `POST /api/tenants/{tenantId}/courses/{courseId}/sections`

Adds a new section to a course.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Algebra Basics",
  "description": "Introduction to algebraic expressions",
  "displayOrder": 1
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `string` | Yes | Section title. Max 200 characters. |
| `description` | `string?` | No | Section description. Max 500 characters. |
| `displayOrder` | `int?` | No | Position in the course. If omitted, appends to end. |

**Response:** `201 Created`

```json
{
  "id": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Algebra Basics",
  "description": "Introduction to algebraic expressions",
  "displayOrder": 1,
  "status": "Draft",
  "lessonCount": 0
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course not found (`Course.NotFound`).
- `409 Conflict` — Course is not in Draft status (`Course.InvalidStatusTransition`).

---

### 2.8 Update Section

#### `PUT /api/tenants/{tenantId}/courses/{courseId}/sections/{sectionId}`

Updates a section's title or description.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Updated Section Title",
  "description": "Updated description"
}
```

All fields are optional.

**Response:** `200 OK`

```json
{
  "id": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Updated Section Title",
  "description": "Updated description",
  "displayOrder": 1,
  "status": "Draft",
  "lessonCount": 3
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course or Section not found (`Course.NotFound`, `Course.SectionNotFound`).

---

### 2.9 Reorder Sections

#### `POST /api/tenants/{tenantId}/courses/{courseId}/sections/reorder`

Reorders all sections within a course. The request body must include all section IDs in the desired order.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "ids": [
    "s2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
    "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "s3c4d5e6-f7a8-9012-b3c4-d5e6f7089012"
  ]
}
```

**Important:** The `ids` array must contain all section IDs for the course. Sections not included will be removed.

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Empty or invalid IDs list.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course or one or more sections not found (`Course.NotFound`, `Course.SectionNotFound`).

---

### 2.10 Create Lesson

#### `POST /api/tenants/{tenantId}/courses/{courseId}/sections/{sectionId}/lessons`

Adds a new lesson to a section.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Variables and Expressions",
  "lessonType": "Video",
  "durationMinutes": 15,
  "content": "{\"videoUrl\": \"https://cdn.teacheros.com/videos/lesson1.mp4\"}"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `string` | Yes | Lesson title. Max 200 characters. |
| `lessonType` | `LessonType` | Yes | `1`=Video, `2`=Text, `3`=Quiz |
| `durationMinutes` | `int?` | No | Estimated completion time in minutes. |
| `content` | `string?` | No | JSON content (video URL, text body, quiz config). Max 50000 chars. |

**Response:** `201 Created`

```json
{
  "id": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Variables and Expressions",
  "lessonType": "Video",
  "durationMinutes": 15,
  "displayOrder": 1,
  "status": "Draft"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course or Section not found (`Course.NotFound`, `Course.SectionNotFound`).

---

### 2.11 Update Lesson

#### `PUT /api/tenants/{tenantId}/courses/{courseId}/sections/{sectionId}/lessons/{lessonId}`

Updates a lesson's content.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Updated Lesson Title",
  "lessonType": "Video",
  "durationMinutes": 20,
  "content": "{\"videoUrl\": \"https://cdn.teacheros.com/videos/lesson1-v2.mp4\"}"
}
```

All fields are optional. `lessonType` can only be changed if the lesson has no content.

**Response:** `200 OK`

```json
{
  "id": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Updated Lesson Title",
  "lessonType": "Video",
  "durationMinutes": 20,
  "displayOrder": 1,
  "status": "Draft"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course, Section, or Lesson not found (`Course.NotFound`, `Course.SectionNotFound`, `Course.LessonNotFound`).

---

### 2.12 Reorder Lessons

#### `POST /api/tenants/{tenantId}/courses/{courseId}/sections/{sectionId}/lessons/reorder`

Reorders all lessons within a section. Similar to section reordering, requires all lesson IDs.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "ids": [
    "l2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
    "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"
  ]
}
```

**Important:** The `ids` array must contain all lesson IDs for the section.

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Empty or invalid IDs list.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course, Section, or one or more lessons not found.

---

### 2.13 Assign Groups to Course

#### `POST /api/tenants/{tenantId}/courses/{courseId}/assign-groups`

Assigns one or more groups to the course. Students in these groups will see the course (if Published).

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `courses.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "groupIds": [
    "g1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "g2b3c4d5-e6f7-8901-b2c3-d4e5f6078901"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `groupIds` | `Array<Guid>` | Yes | List of group IDs to assign. Empty array removes all assignments. |

**Response:** `200 OK`

```json
{
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "assignedGroupIds": [
    "g1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "g2b3c4d5-e6f7-8901-b2c3-d4e5f6078901"
  ]
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error (`Course.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `courses.manage` permission or tenant access denied.
- `404 Not Found` — Course or one or more groups not found (`Course.NotFound`).
- `409 Conflict` — One or more groups are incompatible (`Course.IncompatibleGroup`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `CreateCourseRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route tenantId. |
| `Title` | `string` | Yes | 1–200 characters. |
| `Description` | `string?` | No | Max 2000 characters. |
| `SubjectId` | `Guid?` | No | Valid subject reference. |
| `GradeLevelId` | `Guid?` | No | Valid grade level reference. |
| `CoverImageUrl` | `string?` | No | Valid URL, max 500 chars. |

#### `UpdateCourseRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 characters if provided. |
| `Description` | `string?` | No | Max 2000 characters. |
| `SubjectId` | `Guid?` | No | Valid subject reference or null. |
| `GradeLevelId` | `Guid?` | No | Valid grade level reference or null. |
| `CoverImageUrl` | `string?` | No | Valid URL, max 500 chars, or null. |

#### `CreateSectionRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string` | Yes | 1–200 characters. |
| `Description` | `string?` | No | Max 500 characters. |
| `DisplayOrder` | `int?` | No | If omitted, appends to end. |

#### `UpdateSectionRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 characters. |
| `Description` | `string?` | No | Max 500 characters. |

#### `CreateLessonRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string` | Yes | 1–200 characters. |
| `LessonType` | `LessonType` | Yes | Enum value (1=Video, 2=Text, 3=Quiz). |
| `DurationMinutes` | `int?` | No | Positive integer. |
| `Content` | `string?` | No | JSON string, max 50000 chars. |

#### `UpdateLessonRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 characters. |
| `LessonType` | `LessonType?` | No | Enum value. Only changeable if no content. |
| `DurationMinutes` | `int?` | No | Positive integer or null. |
| `Content` | `string?` | No | JSON string, max 50000 chars. |

#### `ReorderItemsRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Ids` | `List<Guid>` | Yes | Must contain all item IDs for the parent. |

#### `AssignGroupsRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `GroupIds` | `List<Guid>` | Yes | Valid group references. Empty = remove all. |

---

### Response DTOs

#### `CourseSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Unique course identifier. |
| `Title` | `string` | Course title. |
| `Description` | `string?` | Course description. |
| `SubjectId` | `Guid?` | Subject reference if set. |
| `GradeLevelId` | `Guid?` | Grade level reference if set. |
| `CoverImageUrl` | `string?` | Cover image URL if set. |
| `Status` | `string` | Status name (Draft, Published, Archived). |
| `SectionCount` | `int` | Number of sections in the course. |
| `PublishedAtUtc` | `DateTime?` | When the course was published, if applicable. |
| `UpdatedAtUtc` | `DateTime` | Last modification timestamp. |

#### `CourseDetails`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Course identifier. |
| `Title` | `string` | Course title. |
| `Description` | `string?` | Course description. |
| `SubjectId` | `Guid?` | Subject reference. |
| `GradeLevelId` | `Guid?` | Grade level reference. |
| `CoverImageUrl` | `string?` | Cover image URL. |
| `Status` | `string` | Current status. |
| `PublishedAtUtc` | `DateTime?` | Publication timestamp. |
| `Sections` | `List<CourseSectionSummary>` | All sections with lessons. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |
| `UpdatedAtUtc` | `DateTime` | Last modification timestamp. |

#### `CourseSectionSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Section identifier. |
| `Title` | `string` | Section title. |
| `Description` | `string?` | Section description. |
| `DisplayOrder` | `int` | Position in course. |
| `Status` | `string` | Section status. |
| `Lessons` | `List<LessonSummary>` | All lessons in section. |

#### `LessonSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Lesson identifier. |
| `Title` | `string` | Lesson title. |
| `LessonType` | `string` | Type name (Video, Text, Quiz). |
| `DurationMinutes` | `int?` | Estimated duration. |
| `DisplayOrder` | `int` | Position in section. |
| `Status` | `string` | Lesson status. |

---

## 4. Enum Reference

### `CourseStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Course is being prepared. Editable. Not visible to students. |
| `2` | `Published` | Course is live. Visible to assigned groups. Not editable. |
| `3` | `Archived` | Course is hidden and locked. Preserved for records. |

### `CourseSectionStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Section is being prepared. |
| `2` | `Published` | Section is visible to students. |

### `LessonType`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Video` | Video-based lesson with streaming content. |
| `2` | `Text` | Text-based lesson with written content. |
| `3` | `Quiz` | Interactive quiz lesson. |

### `LessonStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Lesson is being prepared. |
| `2` | `Published` | Lesson is visible to students. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Title Uniqueness**: No two courses within a tenant can have the same title (case-insensitive).
2. **Draft-Only Edits**: Course metadata can only be updated when status is `Draft`. Published and Archived courses are immutable.
3. **Publish Prerequisite**: Courses must be in `Draft` status to be published.
4. **Archive Constraint**: Only `Published` courses can be archived. Already `Archived` courses cannot be re-archived.
5. **Section DisplayOrder**: When creating a section without `displayOrder`, it appends to the end.
6. **Lesson Type Immutability**: Once a lesson has content, its `LessonType` cannot be changed.
7. **Content Size**: Lesson content JSON must not exceed 50,000 characters.
8. **Group Compatibility**: Some groups may be incompatible with certain courses (e.g., different curriculum tracks).
9. **Reorder Completeness**: The reorder endpoints require all IDs for the parent entity — partial reorders are not supported.
10. **Assign Groups Replaces**: Calling `assign-groups` with a new list replaces all existing assignments (no merge).

### Frontend Validation Recommendations

- Validate title length client-side before submission (max 200 chars).
- Confirm before publishing — publishing is irreversible.
- Show a warning when archiving a course with active students.
- When reordering, collect all IDs from the current list and reorder in memory before submitting.
- For group assignment, show the current assignments and allow adding/removing from a picker.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Course.NotFound` | 404 | Course does not exist or belongs to another tenant. | Refresh course list. |
| `Course.SectionNotFound` | 404 | Section does not exist within the course. | Refresh course details. |
| `Course.LessonNotFound` | 404 | Lesson does not exist within the section. | Refresh section details. |
| `Course.InvalidInput` | 400 | Missing or invalid input fields. | Highlight invalid fields. |
| `Course.DuplicateName` | 409 | A course with this title already exists in the tenant. | Choose a different title. |
| `Course.InvalidStatusTransition` | 409 | Operation not allowed for current course status. | Check course status before action. |
| `Course.IncompatibleGroup` | 409 | One or more groups cannot be assigned to this course. | Contact administrator. |
| `Course.SubjectNotFound` | 404 | Referenced subject does not exist. | Select a valid subject. |
| `Course.GradeLevelNotFound` | 404 | Referenced grade level does not exist. | Select a valid grade level. |
| `Course.CannotDeletePublishedCourse` | 409 | Cannot delete a published course. Archive instead. | Use archive endpoint. |
| `Tenancy.AccessDenied` | 403 | Tenant context does not match user access. | Verify tenant selection. |

---

## 7. Frontend Integration Best Practices

1. **Course Builder UI**: Build a hierarchical course editor with drag-and-drop sections and lessons. Use the reorder endpoints for final save.
2. **Status Badges**: Color-code course status (gray=Draft, green=Published, orange=Archived).
3. **Draft Indicator**: Show clear "Draft" watermarks on unpublished course cards.
4. **Publish Confirmation**: Require explicit confirmation dialog before publishing — it cannot be undone.
5. **Group Picker**: When assigning groups, show a searchable multi-select with current assignments pre-checked.
6. **Curriculum Tags**: Display subject and grade level badges on course cards for filtering.
7. **Section/Lesson Counts**: Show lesson count per section in the section header (e.g., "Algebra Basics (5 lessons)").
8. **Reorder UX**: For reordering, use drag handles and preview the new order before saving via the API.
9. **Cover Image Fallback**: Provide a default cover image placeholder when `coverImageUrl` is null.
10. **Published Timestamp**: Display `publishedAtUtc` on published courses so teachers know when it went live.
