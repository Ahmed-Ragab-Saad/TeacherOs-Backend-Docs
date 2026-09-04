# Exams Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/learning/exams`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permissions: `exams.view` for reading, `exams.manage` for create/update, `exams.publish` for publishing, `exams.grade` for grading and integrity review.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Exams](#21-list-exams)
   - 2.2 [Create Exam](#22-create-exam)
   - 2.3 [Get Exam Details](#23-get-exam-details)
   - 2.4 [Edit Exam Draft](#24-edit-exam-draft)
   - 2.5 [Publish Exam](#25-publish-exam)
   - 2.6 [Archive Exam](#26-archive-exam)
   - 2.7 [Start Exam Attempt](#27-start-exam-attempt)
   - 2.8 [Save Exam Answers](#28-save-exam-answers)
   - 2.9 [Submit Exam Attempt](#29-submit-exam-attempt)
   - 2.10 [Review Exam Attempt](#210-review-exam-attempt)
   - 2.11 [Record Integrity Event](#211-record-integrity-event)
   - 2.12 [Get Exam Result](#212-get-exam-result)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Exams module manages formal assessments with stronger integrity controls than homework. Exams support timed sessions, multiple attempts, integrity event tracking (proctoring signals), and result release policies. Unlike homework, exams have stricter controls around availability windows and attempt management.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Exam** | Formal assessment with questions, attachments, availability window, and integrity settings. |
| **ExamStatus** | Draft → Published → Archived. No "Closed" status like homework — exams end when `availableUntilUtc` passes. |
| **ExamAttempt** | A student's attempt at an exam. Has its own status lifecycle and timer. |
| **ExamDeliveryMode** | OnlineQuestions, FileBased, or Hybrid. |
| **IntegrityEvent** | Proctoring signals captured during an attempt (focus lost, fullscreen exit, etc.). |
| **ResultReleasePolicy** | When students can see results: Immediate, AfterExamCloses, Manual. |

### Integrity Event Flow

```
Student Starts Exam
        |
        v
Integrity Monitoring (frontend)
        |
        +-- Browser focus lost -----> RecordIntegrityEvent(FocusLost)
        +-- App backgrounded -------> RecordIntegrityEvent(AppBackgrounded)
        +-- Fullscreen exit --------> RecordIntegrityEvent(FullscreenExit)
        +-- Reconnection -----------> RecordIntegrityEvent(ReconnectAttempt)
        |
        v
Teacher Reviews Attempt
        +-- Reviews Integrity Events
        +-- Determines if flags are acceptable
        +-- Assigns Final Score
```

### Data Flow

```
Create Exam (Draft)
        |
        v
Add Questions from Question Bank
        |
        v
Configure Integrity Settings
        |
        v
Set Availability Window
        |
        v
Publish → Available to Students
        |
        v
Students Start (timer begins) → Answer → Submit
        |
        v
Exam Ends (availability window closes)
        |
        v
Results Released (per policy)
        |
        v
Teacher Reviews (if needed)
```

### Important Frontend Implications

- **Timed Exams**: When a student starts an attempt, the exam timer begins. The frontend must track the remaining time accurately.
- **Integrity Events**: The frontend should monitor browser events (blur, visibilitychange, fullscreenchange) and report them to the API.
- **Result Visibility**: Results may be hidden until the exam window closes or released manually — respect the release policy.
- **File-Based Exams**: For FileBased mode, students upload their answers rather than answering inline questions.
- **Auto-Save**: Similar to homework, exam answers can be saved without submitting.

---

## 2. API Endpoints Reference

### 2.1 List Exams

#### `GET /api/tenants/{tenantId}/learning/exams`

Retrieves a paginated list of exams.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.view`

**Query Parameters:**
- `courseId` (Guid, optional): Filter by course.
- `sectionId` (Guid, optional): Filter by section.
- `lessonId` (Guid, optional): Filter by lesson.
- `status` (ExamStatus, optional): Filter by status.
- `page` (int, optional, default `1`): Page number.
- `pageSize` (int, optional, default `20`): Items per page (max 100).

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
      "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "lessonId": null,
      "title": "Midterm Examination",
      "deliveryMode": "OnlineQuestions",
      "availableFromUtc": "2026-09-15T08:00:00Z",
      "availableUntilUtc": "2026-09-15T12:00:00Z",
      "durationMinutes": 120,
      "maxAttempts": 1,
      "resultReleasePolicy": "AfterExamCloses",
      "status": "Published",
      "questionCount": 50,
      "attachmentCount": 1,
      "attemptCount": 0,
      "publishedAtUtc": "2026-09-01T10:00:00Z",
      "createdAtUtc": "2026-08-28T14:00:00Z"
    }
  ],
  "totalCount": 3,
  "page": 1,
  "pageSize": 20
}
```

**Possible Errors:**
- `400 Bad Request` — Invalid query parameters.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.view` permission or tenant access denied.

---

### 2.2 Create Exam

#### `POST /api/tenants/{tenantId}/learning/exams`

Creates a new exam in Draft status.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Midterm Examination",
  "instructions": "Answer all questions. No external resources allowed.",
  "deliveryMode": 1,
  "availableFromUtc": "2026-09-15T08:00:00Z",
  "availableUntilUtc": "2026-09-15T12:00:00Z",
  "durationMinutes": 120,
  "maxAttempts": 1,
  "resultReleasePolicy": 2,
  "shuffleQuestions": false,
  "shuffleOptions": true,
  "questionIds": [
    "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901"
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `courseId` | `Guid` | Yes | Course context. |
| `sectionId` | `Guid?` | No | Section context. |
| `lessonId` | `Guid?` | No | Lesson context. |
| `title` | `string` | Yes | Exam title. Max 200 characters. |
| `instructions` | `string?` | No | Exam instructions. Max 5000 characters. |
| `deliveryMode` | `ExamDeliveryMode` | Yes | `1`=OnlineQuestions, `2`=FileBased, `3`=Hybrid. |
| `availableFromUtc` | `DateTime?` | No | When exam becomes available. |
| `availableUntilUtc` | `DateTime?` | No | When exam window closes. |
| `durationMinutes` | `int?` | No | Time limit for each attempt. If null, no time limit. |
| `maxAttempts` | `int` | No | Maximum attempts per student. Default: 1. |
| `resultReleasePolicy` | `ExamResultReleasePolicy` | No | When results are shown. Default: `AfterExamCloses`. |
| `shuffleQuestions` | `bool` | No | Randomize question order. Default: false. |
| `shuffleOptions` | `bool` | No | Randomize option order within questions. Default: false. |
| `questionIds` | `Array<Guid>?` | Conditional | Required if `deliveryMode` is OnlineQuestions or Hybrid. |

**Response:** `201 Created`

```json
{
  "id": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Midterm Examination",
  "deliveryMode": "OnlineQuestions",
  "availableFromUtc": "2026-09-15T08:00:00Z",
  "availableUntilUtc": "2026-09-15T12:00:00Z",
  "durationMinutes": 120,
  "maxAttempts": 1,
  "resultReleasePolicy": "AfterExamCloses",
  "status": "Draft",
  "questionCount": 2,
  "attachmentCount": 0,
  "attemptCount": 0,
  "publishedAtUtc": null,
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.manage` permission or tenant access denied.
- `404 Not Found` — Course or question not found.

---

### 2.3 Get Exam Details

#### `GET /api/tenants/{tenantId}/learning/exams/{examId}`

Retrieves full exam details including questions and attachments.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.view`

**Response:** `200 OK`

```json
{
  "id": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Midterm Examination",
  "instructions": "Answer all questions. No external resources allowed.",
  "deliveryMode": "OnlineQuestions",
  "availableFromUtc": "2026-09-15T08:00:00Z",
  "availableUntilUtc": "2026-09-15T12:00:00Z",
  "durationMinutes": 120,
  "maxAttempts": 1,
  "resultReleasePolicy": "AfterExamCloses",
  "shuffleQuestions": false,
  "shuffleOptions": true,
  "status": "Published",
  "publishedAtUtc": "2026-09-01T10:00:00Z",
  "questions": [
    {
      "questionId": "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "questionBankItemId": "qb1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "questionText": "What is the result of 2 + 2?",
      "type": "MultipleChoice",
      "points": 5,
      "options": [
        { "id": "o1", "text": "3", "isCorrect": null },
        { "id": "o2", "text": "4", "isCorrect": null },
        { "id": "o3", "text": "5", "isCorrect": null },
        { "id": "o4", "text": "6", "isCorrect": null }
      ]
    }
  ],
  "attachments": [
    {
      "id": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
      "title": "Formula Sheet",
      "role": "ReferenceMaterial",
      "fileUrl": "https://cdn.teacheros.com/exams/formulas.pdf"
    }
  ],
  "createdAtUtc": "2026-08-28T14:00:00Z",
  "updatedAtUtc": "2026-09-01T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.view` permission or tenant access denied.
- `404 Not Found` — Exam not found (`Exam.NotFound`).

---

### 2.4 Edit Exam Draft

#### `PUT /api/tenants/{tenantId}/learning/exams/{examId}`

Updates a draft exam's content. Only allowed for Draft status.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Updated Exam Title",
  "instructions": "Updated instructions.",
  "durationMinutes": 90,
  "maxAttempts": 2,
  "shuffleQuestions": true,
  "shuffleOptions": true,
  "questionIds": [
    "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
    "q3c4d5e6-f7a8-9012-c3d4-e5f607890123"
  ]
}
```

All fields are optional.

**Response:** `200 OK`

Same structure as Create response.

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.manage` permission or tenant access denied.
- `404 Not Found` — Exam not found (`Exam.NotFound`).
- `409 Conflict` — Exam is not in Draft status (`Exam.InvalidStatus`).

---

### 2.5 Publish Exam

#### `POST /api/tenants/{tenantId}/learning/exams/{examId}/publish`

Publishes a draft exam, making it available to students.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.publish`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Midterm Examination",
  "status": "Published",
  "publishedAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.publish` permission or tenant access denied.
- `404 Not Found` — Exam not found (`Exam.NotFound`).
- `409 Conflict` — Exam is not in Draft status (`Exam.InvalidStatus`).

---

### 2.6 Archive Exam

#### `POST /api/tenants/{tenantId}/learning/exams/{examId}/archive`

Archives a published exam, hiding it from students.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Midterm Examination",
  "status": "Archived",
  "archivedAtUtc": "2026-09-16T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.manage` permission or tenant access denied.
- `404 Not Found` — Exam not found (`Exam.NotFound`).
- `409 Conflict` — Exam is not in Published status (`Exam.InvalidStatus`).

---

### 2.7 Start Exam Attempt

#### `POST /api/tenants/{tenantId}/learning/exams/{examId}/attempts/start`

Starts a new exam attempt for a student. Begins the exam timer if `durationMinutes` is set.

**Authentication:** Required (Cookie-based)

**Authorization:** Student context

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `201 Created`

```json
{
  "attemptId": "at1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "examId": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "studentId": "st1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "attemptNumber": 1,
  "status": "InProgress",
  "startedAtUtc": "2026-09-15T08:05:00Z",
  "expiresAtUtc": "2026-09-15T10:05:00Z",
  "durationMinutes": 120
}
```

| Property | Type | Description |
|----------|------|-------------|
| `expiresAtUtc` | `DateTime` | When the timed attempt expires. Client should auto-submit at this time. |

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not an eligible student.
- `404 Not Found` — Exam not found (`Exam.NotFound`).
- `409 Conflict` — Active attempt exists (`Exam.ActiveAttemptExists`), max attempts reached (`Exam.MaxAttemptsReached`), exam not available (`Exam.NotAvailable`).

---

### 2.8 Save Exam Answers

#### `PUT /api/tenants/{tenantId}/learning/exams/attempts/{attemptId}/answers`

Saves the student's answers without submitting. Supports periodic auto-save.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who started the attempt

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "answers": [
    {
      "questionId": "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "selectedOptionIds": ["o2"],
      "textAnswer": null
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `answers` | `Array<AnswerInput>` | Yes | Student's answers. |
| `answers[].questionId` | `Guid` | Yes | The question being answered. |
| `answers[].selectedOptionIds` | `Array<Guid>?` | Conditional | Selected option IDs for MultipleChoice/TrueFalse. |
| `answers[].textAnswer` | `string?` | Conditional | Text answer for ShortAnswer/Essay. |

**Response:** `200 OK`

```json
{
  "attemptId": "at1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "savedAtUtc": "2026-09-15T08:30:00Z",
  "answeredCount": 5,
  "totalQuestions": 50,
  "remainingSeconds": 5400
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not the attempt owner.
- `404 Not Found` — Attempt not found (`Exam.AttemptNotFound`).
- `409 Conflict` — Attempt already submitted or expired (`Exam.InvalidStatus`, `Exam.AttemptExpired`).

---

### 2.9 Submit Exam Attempt

#### `POST /api/tenants/{tenantId}/learning/exams/attempts/{attemptId}/submit`

Submits the exam. Once submitted, no further changes are allowed. Triggers auto-scoring for auto-gradable questions.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who started the attempt

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "attemptId": "at1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "status": "Submitted",
  "submittedAtUtc": "2026-09-15T09:45:00Z",
  "autoScore": 40,
  "maxScore": 50,
  "needsReview": true
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not the attempt owner.
- `404 Not Found` — Attempt not found (`Exam.AttemptNotFound`).
- `409 Conflict` — Attempt already submitted or expired (`Exam.InvalidStatus`, `Exam.AttemptExpired`).

---

### 2.10 Review Exam Attempt

#### `POST /api/tenants/{tenantId}/learning/exams/attempts/{attemptId}/review`

Teacher reviews a submitted attempt and assigns final scores.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `exams.grade`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "marks": [
    {
      "questionId": "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "score": 5,
      "feedback": "Correct!"
    },
    {
      "questionId": "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "score": 4,
      "feedback": "Good explanation."
    }
  ],
  "finalScore": 44,
  "releaseNow": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `marks` | `Array<MarkInput>` | Yes | Per-question scores. |
| `finalScore` | `decimal?` | No | Override total score. If omitted, sum of marks. |
| `releaseNow` | `bool?` | No | If true, release results immediately (ignores policy). |

**Response:** `200 OK`

```json
{
  "attemptId": "at1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "status": "Graded",
  "finalScore": 44,
  "maxScore": 50,
  "reviewedByMembershipId": "m1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "reviewedAtUtc": "2026-09-16T14:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `exams.grade` permission.
- `404 Not Found` — Attempt not found (`Exam.AttemptNotFound`).
- `409 Conflict` — Attempt not in reviewable state (`Exam.InvalidStatus`).

---

### 2.11 Record Integrity Event

#### `POST /api/tenants/{tenantId}/learning/exams/attempts/{attemptId}/integrity-events`

Records a proctoring signal during an exam attempt.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who started the attempt

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "eventType": 3,
  "details": "User exited fullscreen mode",
  "timestampUtc": "2026-09-15T08:30:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `eventType` | `ExamIntegrityEventType` | Yes | Type of integrity event. |
| `details` | `string?` | No | Additional context. Max 500 characters. |
| `timestampUtc` | `DateTime?` | No | When the event occurred. Defaults to now. |

**Response:** `204 No Content`

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not the attempt owner.
- `404 Not Found` — Attempt not found (`Exam.AttemptNotFound`).
- `409 Conflict` — Attempt not in progress (`Exam.InvalidStatus`).

---

### 2.12 Get Exam Result

#### `GET /api/tenants/{tenantId}/learning/exams/attempts/{attemptId}/result`

Retrieves the result for an exam attempt. Result visibility is controlled by the exam's `resultReleasePolicy`.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who made the attempt, or teacher with `exams.grade`

**Response:** `200 OK`

```json
{
  "attemptId": "at1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "examId": "e1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "studentId": "st1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "attemptNumber": 1,
  "status": "Graded",
  "finalScore": 44,
  "maxScore": 50,
  "isReleased": true,
  "submittedAtUtc": "2026-09-15T09:45:00Z"
}
```

**Response (Not Released):** `403 Forbidden`

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Forbidden",
  "status": 403,
  "detail": "Exam results have not been released yet according to policy.",
  "extensions": {
    "code": "Exam.ResultsNotReleased"
  }
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Results not released (`Exam.ResultsNotReleased`) or not the attempt owner.
- `404 Not Found` — Attempt not found (`Exam.AttemptNotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `CreateExamRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `CourseId` | `Guid` | Yes | Valid course reference. |
| `SectionId` | `Guid?` | No | Valid section reference. |
| `LessonId` | `Guid?` | No | Valid lesson reference. |
| `Title` | `string` | Yes | 1–200 characters. |
| `Instructions` | `string?` | No | Max 5000 characters. |
| `DeliveryMode` | `ExamDeliveryMode` | Yes | Enum value. |
| `AvailableFromUtc` | `DateTime?` | No | UTC timestamp. |
| `AvailableUntilUtc` | `DateTime?` | No | UTC timestamp. |
| `DurationMinutes` | `int?` | No | Positive integer. |
| `MaxAttempts` | `int` | No | Positive integer, default 1. |
| `ResultReleasePolicy` | `ExamResultReleasePolicy` | No | Enum value, default `AfterExamCloses`. |
| `ShuffleQuestions` | `bool` | No | Default false. |
| `ShuffleOptions` | `bool` | No | Default false. |
| `QuestionIds` | `List<Guid>?` | Conditional | Required for OnlineQuestions/Hybrid. |

#### `EditExamRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 characters. |
| `Instructions` | `string?` | No | Max 5000 characters. |
| `DeliveryMode` | `ExamDeliveryMode?` | No | Enum value. |
| `AvailableFromUtc` | `DateTime?` | No | UTC timestamp. |
| `AvailableUntilUtc` | `DateTime?` | No | UTC timestamp. |
| `DurationMinutes` | `int?` | No | Positive integer. |
| `MaxAttempts` | `int?` | No | Positive integer. |
| `ResultReleasePolicy` | `ExamResultReleasePolicy?` | No | Enum value. |
| `ShuffleQuestions` | `bool?` | No | Boolean. |
| `ShuffleOptions` | `bool?` | No | Boolean. |
| `QuestionIds` | `List<Guid>?` | No | Replace all questions if provided. |

#### `AnswerInput`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `QuestionId` | `Guid` | Yes | Valid question reference. |
| `SelectedOptionIds` | `List<Guid>?` | Conditional | Required for MultipleChoice/TrueFalse. |
| `TextAnswer` | `string?` | Conditional | Required for ShortAnswer/Essay. |

#### `MarkInput`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `QuestionId` | `Guid` | Yes | Valid question reference. |
| `Score` | `decimal` | Yes | Non-negative, <= question points. |
| `Feedback` | `string?` | No | Max 500 characters. |

#### `RecordIntegrityEventRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `EventType` | `ExamIntegrityEventType` | Yes | Enum value. |
| `Details` | `string?` | No | Max 500 characters. |
| `TimestampUtc` | `DateTime?` | No | UTC timestamp, defaults to now. |

---

### Response DTOs

#### `ExamSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Exam identifier. |
| `CourseId` | `Guid` | Course reference. |
| `SectionId` | `Guid?` | Section reference. |
| `LessonId` | `Guid?` | Lesson reference. |
| `Title` | `string` | Exam title. |
| `DeliveryMode` | `string` | Mode name. |
| `AvailableFromUtc` | `DateTime?` | Availability start. |
| `AvailableUntilUtc` | `DateTime?` | Window close time. |
| `DurationMinutes` | `int?` | Time limit per attempt. |
| `MaxAttempts` | `int` | Maximum attempts. |
| `ResultReleasePolicy` | `string` | Release policy name. |
| `Status` | `string` | Current status. |
| `QuestionCount` | `int` | Number of questions. |
| `AttachmentCount` | `int` | Number of attachments. |
| `AttemptCount` | `int` | Number of attempts made. |
| `PublishedAtUtc` | `DateTime?` | Publication timestamp. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |

#### `ExamDetails`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Exam identifier. |
| `CourseId` | `Guid` | Course reference. |
| `SectionId` | `Guid?` | Section reference. |
| `LessonId` | `Guid?` | Lesson reference. |
| `Title` | `string` | Exam title. |
| `Instructions` | `string?` | Instructions text. |
| `DeliveryMode` | `string` | Mode name. |
| `AvailableFromUtc` | `DateTime?` | Availability start. |
| `AvailableUntilUtc` | `DateTime?` | Window close time. |
| `DurationMinutes` | `int?` | Time limit. |
| `MaxAttempts` | `int` | Maximum attempts. |
| `ResultReleasePolicy` | `string` | Release policy name. |
| `ShuffleQuestions` | `bool` | Questions randomized. |
| `ShuffleOptions` | `bool` | Options randomized. |
| `Status` | `string` | Current status. |
| `PublishedAtUtc` | `DateTime?` | Publication timestamp. |
| `Questions` | `List<ExamQuestionDto>` | Questions with options (isCorrect hidden). |
| `Attachments` | `List<ExamAttachmentDto>` | Supporting files. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |
| `UpdatedAtUtc` | `DateTime` | Last update. |

#### `ExamAttemptSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Attempt identifier. |
| `ExamId` | `Guid` | Exam reference. |
| `StudentId` | `Guid` | Student reference. |
| `AttemptNumber` | `int` | Which attempt this is. |
| `Status` | `string` | Attempt status. |
| `StartedAtUtc` | `DateTime` | When attempt started. |
| `ExpiresAtUtc` | `DateTime?` | When timed attempt expires. |
| `SubmittedAtUtc` | `DateTime?` | Submission time. |
| `AutoScore` | `decimal?` | Auto-graded score. |
| `FinalScore` | `decimal?` | Final score after review. |
| `ReviewedByMembershipId` | `Guid?` | Reviewer ID. |
| `ReviewedAtUtc` | `DateTime?` | Review time. |
| `CreatedAtUtc` | `DateTime` | Creation time. |
| `UpdatedAtUtc` | `DateTime` | Last update. |

#### `ExamResultView`

| Property | Type | Description |
|----------|------|-------------|
| `AttemptId` | `Guid` | Attempt identifier. |
| `ExamId` | `Guid` | Exam reference. |
| `StudentId` | `Guid` | Student reference. |
| `AttemptNumber` | `int` | Attempt number. |
| `Status` | `string` | Attempt status. |
| `FinalScore` | `decimal?` | Final score. |
| `IsReleased` | `bool` | Whether results are visible. |
| `SubmittedAtUtc` | `DateTime?` | Submission time. |

---

## 4. Enum Reference

### `ExamStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Being prepared. Editable. Not visible to students. |
| `2` | `Published` | Live. Visible to students within availability window. |
| `3` | `Archived` | Complete. Hidden from students. Preserved for records. |

### `ExamDeliveryMode`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `OnlineQuestions` | Students answer questions online. |
| `2` | `FileBased` | Students upload their answers as files. |
| `3` | `Hybrid` | Both online questions and file upload. |

### `ExamAttemptStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `InProgress` | Student is taking the exam. Timer running. |
| `2` | `Submitted` | Submitted. Awaiting auto-score or review. |
| `3` | `NeedsReview` | Flagged for teacher review. |
| `4` | `Graded` | Fully graded. Results available per policy. |

### `ExamResultReleasePolicy`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Immediate` | Results shown immediately after submission. |
| `2` | `AfterExamCloses` | Results shown after the exam window closes. |
| `3` | `Manual` | Results released manually by teacher. |

### `ExamIntegrityEventType`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `FocusLost` | Browser/app lost focus (switched away). |
| `2` | `AppBackgrounded` | App moved to background (mobile). |
| `3` | `FullscreenExit` | User exited fullscreen mode. |
| `4` | `ReconnectAttempt` | Student reconnected after connection loss. |

### `ExamAttachmentRole`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `ExamDocument` | The exam paper/questions document. |
| `2` | `ReferenceMaterial` | Reference files allowed during exam. |
| `3` | `AnswerModel` | Teacher answer key (hidden from students). |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Draft-Only Edits**: Exam metadata can only be updated when status is `Draft`.
2. **Publish Prerequisite**: Exam must be `Draft` to be published.
3. **Archive Prerequisite**: Exam must be `Published` to be archived.
4. **Availability Window**: Attempts can only be started within `availableFromUtc` and `availableUntilUtc`.
5. **Duration Timer**: When `durationMinutes` is set, the attempt auto-expires at `startedAt + durationMinutes`.
6. **Max Attempts**: Students cannot start a new attempt if they have reached `maxAttempts`.
7. **Active Attempt**: Students can only have one `InProgress` attempt at a time.
8. **Submitted Immutability**: Once submitted, answers cannot be changed.
9. **Results Visibility**: Results are only visible when `isReleased` is true, which is controlled by the `resultReleasePolicy`.
10. **Integrity Events**: Events can only be recorded for `InProgress` attempts.
11. **Questions Required**: OnlineQuestions and Hybrid modes require at least one question.

### Frontend Validation Recommendations

- Validate that `availableUntilUtc` is after `availableFromUtc`.
- Warn if `durationMinutes` is not set for an online exam.
- For timed exams, implement a countdown timer in the UI.
- When timer reaches 5 minutes, show a warning.
- When timer expires, auto-submit the attempt.
- Monitor `visibilitychange`, `blur`, and `fullscreenchange` events and report integrity events.
- For FileBased mode, provide file upload UI.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Exam.NotFound` | 404 | Exam does not exist or belongs to another tenant. | Refresh exam list. |
| `Exam.AttemptNotFound` | 404 | Attempt does not exist. | Refresh attempt list. |
| `Exam.InvalidStatus` | 409 | Operation not allowed for current exam/attempt status. | Check current status. |
| `Exam.MaxAttemptsReached` | 409 | Student has used all allowed attempts. | No action available. |
| `Exam.NotAvailable` | 409 | Exam is not currently available (outside window). | Check availability window. |
| `Exam.AttemptExpired` | 409 | Timed attempt has expired. | Attempt auto-submitted. |
| `Exam.ActiveAttemptExists` | 409 | Student already has an in-progress attempt. | Resume existing attempt. |
| `Exam.ResultsNotReleased` | 403 | Results not visible per release policy. | Wait for release. |
| `Tenancy.AccessDenied` | 403 | Tenant context mismatch. | Verify tenant selection. |

---

## 7. Frontend Integration Best Practices

1. **Exam Builder**: Create a multi-step exam builder: (1) Basic info & settings, (2) Questions, (3) Review & Publish.
2. **Timer Implementation**: For timed exams, implement a precise countdown timer. Sync with server `expiresAtUtc` to handle clock drift.
3. **Auto-Submit on Expiry**: When the timer expires, automatically submit the exam without requiring user action.
4. **Integrity Monitoring**: Subscribe to browser events and record integrity events:
   - `window.addEventListener('blur', ...)` → `FocusLost`
   - `document.addEventListener('visibilitychange', ...)` → `AppBackgrounded`
   - `document.addEventListener('fullscreenchange', ...)` → `FullscreenExit`
5. **Warning Banners**: Show a persistent banner if integrity events have been recorded (e.g., "3 focus losses recorded").
6. **Status Badges**: Color-code exam status (gray=Draft, green=Published, muted=Archived) and attempt status.
7. **Availability Countdown**: Show "Exam opens in X days" or "Exam closes in X hours" messaging.
8. **Result Policy Display**: Clearly communicate when results will be available based on the release policy.
9. **Integrity Dashboard**: For teachers, build an integrity review panel showing all events for each attempt.
10. **Grading Interface**: Build a grading interface that shows student answers, integrity events, and allows per-question scoring.
11. **Attempts History**: Show students their attempt history with scores (if released) and attempt numbers.
12. **Shuffle Display**: When shuffling is enabled, display questions and options in randomized order per student.
