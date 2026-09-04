# Homework Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/learning/homeworks`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permissions: `homework.view` for reading, `homework.manage` for create/update, `homework.publish` for publishing, `homework.grade` for grading.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Homeworks](#21-list-homeworks)
   - 2.2 [Create Homework](#22-create-homework)
   - 2.3 [Get Homework Details](#23-get-homework-details)
   - 2.4 [Edit Homework Draft](#24-edit-homework-draft)
   - 2.5 [Publish Homework](#25-publish-homework)
   - 2.6 [Close Homework](#26-close-homework)
   - 2.7 [Archive Homework](#27-archive-homework)
   - 2.8 [Start Submission](#28-start-submission)
   - 2.9 [Save Answers](#29-save-answers)
   - 2.10 [Submit Answers](#210-submit-answers)
   - 2.11 [Review Submission](#211-review-submission)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Homework module manages assignments distributed to students. It supports multiple submission modes (online questions, file upload, or hybrid), a draft/submit/review workflow, and attachment support. Homeworks are created within the context of a course/section/lesson.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Homework** | An assignment with title, instructions, questions, attachments, and availability window. |
| **Submission Mode** | How students submit: OnlineQuestions, FileUpload, or Hybrid. |
| **Homework Status** | Draft (editable) → Published → Closed → Archived. |
| **Submission Status** | Per-student state: InProgress, Submitted, NeedsReview, Reviewed. |
| **Recipients** | Groups or individual students assigned to the homework. |
| **Attachments** | Supporting files (instructions, worksheets, reference material, answer keys). |
| **Questions** | Questions from the question bank assigned to this homework. |

### Submission Workflow

```
Teacher Side:
Create Homework (Draft) → Edit → Publish → (Close at deadline) → Review Submissions → Archive

Student Side:
View Published Homework → Start Submission → Save Answers (auto-save) → Submit → Await Review
                                                                              ↓
                                                                    Teacher Reviews
                                                                              ↓
                                                                    Submission Reviewed
```

### Data Flow

```
Create Homework (Draft)
        |
        v
Add Questions from Question Bank
        |
        v
Attach Supporting Files
        |
        v
Select Recipients (Groups/Students)
        |
        v
Publish → Available to Students
        |
        v
Students Start → Answer → Submit
        |
        v
Teacher Reviews (after deadline)
        |
        v
Feedback Given → Submission Complete
```

### Important Frontend Implications

- **Draft Until Published**: Homeworks are editable until published. After publishing, only minor changes allowed.
- **Questions from Bank**: Homework questions are references to the question bank — editing a bank question updates all homeworks using it.
- **Auto-Save**: Students can save answers without submitting. The `SaveAnswers` endpoint supports periodic auto-save.
- **Late Submissions**: Whether late submissions are allowed depends on the homework settings.
- **File Upload Support**: For FileUpload and Hybrid modes, students upload files as part of their submission.
- **Max Attempts**: Students may be limited to a number of submission attempts.

---

## 2. API Endpoints Reference

### 2.1 List Homeworks

#### `GET /api/tenants/{tenantId}/learning/homeworks`

Retrieves a paginated list of homeworks.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.view`

**Query Parameters:**
- `courseId` (Guid, optional): Filter by course.
- `sectionId` (Guid, optional): Filter by section.
- `lessonId` (Guid, optional): Filter by lesson.
- `status` (HomeworkStatus, optional): Filter by status.
- `page` (int, optional, default `1`): Page number.
- `pageSize` (int, optional, default `20`): Items per page (max 100).

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
      "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "lessonId": null,
      "title": "Math Homework Chapter 5",
      "submissionMode": "OnlineQuestions",
      "availableFromUtc": "2026-09-01T00:00:00Z",
      "availableUntilUtc": "2026-09-08T23:59:00Z",
      "allowLateSubmissions": true,
      "maxAttempts": 2,
      "status": "Published",
      "questionCount": 10,
      "attachmentCount": 1,
      "submissionCount": 15,
      "publishedAtUtc": "2026-08-30T10:00:00Z",
      "createdAtUtc": "2026-08-28T14:00:00Z"
    }
  ],
  "totalCount": 5,
  "page": 1,
  "pageSize": 20
}
```

**Possible Errors:**
- `400 Bad Request` — Invalid query parameters.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.view` permission or tenant access denied.

---

### 2.2 Create Homework

#### `POST /api/tenants/{tenantId}/learning/homeworks`

Creates a new homework in Draft status.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Math Homework Chapter 5",
  "instructions": "Complete all questions and submit before the deadline.",
  "submissionMode": 2,
  "availableFromUtc": "2026-09-01T00:00:00Z",
  "availableUntilUtc": "2026-09-08T23:59:00Z",
  "allowLateSubmissions": true,
  "maxAttempts": 2,
  "questionIds": [
    "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901"
  ],
  "recipientType": 1,
  "recipientGroupIds": ["g1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `courseId` | `Guid` | Yes | Course context. |
| `sectionId` | `Guid?` | No | Section context. |
| `lessonId` | `Guid?` | No | Lesson context. |
| `title` | `string` | Yes | Homework title. Max 200 characters. |
| `instructions` | `string?` | No | Detailed instructions. Max 5000 characters. |
| `submissionMode` | `HomeworkSubmissionMode` | Yes | `1`=NoSubmission, `2`=OnlineQuestions, `3`=FileUpload, `4`=Hybrid. |
| `availableFromUtc` | `DateTime?` | No | When homework becomes available. |
| `availableUntilUtc` | `DateTime?` | No | Deadline for submissions. |
| `allowLateSubmissions` | `bool` | No | Whether late submissions are allowed. Default: false. |
| `maxAttempts` | `int` | No | Maximum submission attempts. Default: 1. |
| `questionIds` | `Array<Guid>?` | Conditional | Required if `submissionMode` is OnlineQuestions or Hybrid. |
| `recipientType` | `HomeworkRecipientType` | Yes | `1`=AllStudents, `2`=SpecificGroups, `3`=SpecificStudents. |
| `recipientGroupIds` | `Array<Guid>?` | Conditional | Required if `recipientType` is SpecificGroups. |
| `recipientStudentIds` | `Array<Guid>?` | Conditional | Required if `recipientType` is SpecificStudents. |

**Response:** `201 Created`

```json
{
  "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Math Homework Chapter 5",
  "instructions": "Complete all questions and submit before the deadline.",
  "submissionMode": "OnlineQuestions",
  "availableFromUtc": "2026-09-01T00:00:00Z",
  "availableUntilUtc": "2026-09-08T23:59:00Z",
  "allowLateSubmissions": true,
  "maxAttempts": 2,
  "status": "Draft",
  "questionCount": 2,
  "attachmentCount": 0,
  "submissionCount": 0,
  "publishedAtUtc": null,
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.manage` permission or tenant access denied.
- `404 Not Found` — Course, section, or question not found.

---

### 2.3 Get Homework Details

#### `GET /api/tenants/{tenantId}/learning/homeworks/{homeworkId}`

Retrieves full homework details including questions, attachments, and recipients.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.view`

**Response:** `200 OK`

```json
{
  "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": null,
  "title": "Math Homework Chapter 5",
  "instructions": "Complete all questions and submit before the deadline.",
  "submissionMode": "OnlineQuestions",
  "availableFromUtc": "2026-09-01T00:00:00Z",
  "availableUntilUtc": "2026-09-08T23:59:00Z",
  "allowLateSubmissions": true,
  "maxAttempts": 2,
  "status": "Published",
  "publishedAtUtc": "2026-08-30T10:00:00Z",
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
      "title": "Chapter 5 Notes",
      "role": "ReferenceMaterial",
      "fileUrl": "https://cdn.teacheros.com/homeworks/notes.pdf"
    }
  ],
  "recipientType": "AllStudents",
  "recipientGroupIds": null,
  "recipientStudentIds": null,
  "createdAtUtc": "2026-08-28T14:00:00Z",
  "updatedAtUtc": "2026-08-30T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.view` permission or tenant access denied.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).

---

### 2.4 Edit Homework Draft

#### `PUT /api/tenants/{tenantId}/learning/homeworks/{homeworkId}`

Updates a draft homework's content. Only allowed for Draft status.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Updated Homework Title",
  "instructions": "Updated instructions with more detail.",
  "submissionMode": 2,
  "availableFromUtc": "2026-09-01T00:00:00Z",
  "availableUntilUtc": "2026-09-10T23:59:00Z",
  "allowLateSubmissions": true,
  "maxAttempts": 3,
  "questionIds": [
    "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
    "q3c4d5e6-f7a8-9012-c3d4-e5f607890123"
  ],
  "recipientType": 2,
  "recipientGroupIds": ["g1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"]
}
```

All fields are optional except at least one must be provided.

**Response:** `200 OK`

Same structure as Create response.

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.manage` permission or tenant access denied.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).
- `409 Conflict` — Homework is not in Draft status (`Homework.InvalidStatus`).

---

### 2.5 Publish Homework

#### `POST /api/tenants/{tenantId}/learning/homeworks/{homeworkId}/publish`

Publishes a draft homework, making it available to recipients.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.publish`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Math Homework Chapter 5",
  "status": "Published",
  "publishedAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.publish` permission or tenant access denied.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).
- `409 Conflict` — Homework is not in Draft status (`Homework.InvalidStatus`).

---

### 2.6 Close Homework

#### `POST /api/tenants/{tenantId}/learning/homeworks/{homeworkId}/close`

Closes a published homework, preventing new submissions.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Math Homework Chapter 5",
  "status": "Closed",
  "closedAtUtc": "2026-09-08T23:59:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.manage` permission or tenant access denied.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).
- `409 Conflict` — Homework is not in Published status (`Homework.InvalidStatus`).

---

### 2.7 Archive Homework

#### `POST /api/tenants/{tenantId}/learning/homeworks/{homeworkId}/archive`

Archives a homework, hiding it from students and marking as complete.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Math Homework Chapter 5",
  "status": "Archived",
  "archivedAtUtc": "2026-09-15T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.manage` permission or tenant access denied.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).
- `409 Conflict` — Homework is not in Closed status (`Homework.InvalidStatus`).

---

### 2.8 Start Submission

#### `POST /api/tenants/{tenantId}/learning/homeworks/{homeworkId}/submissions/start`

Starts a new submission attempt for a student. Creates an in-progress submission record.

**Authentication:** Required (Cookie-based)

**Authorization:** Student context (authenticated user must be the student)

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `201 Created`

```json
{
  "submissionId": "sub1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "homeworkId": "h1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "studentId": "st1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "attemptNumber": 1,
  "status": "InProgress",
  "startedAtUtc": "2026-09-04T10:30:00Z",
  "expiresAtUtc": "2026-09-08T23:59:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not an eligible student.
- `404 Not Found` — Homework not found (`Homework.HomeworkNotFound`).
- `409 Conflict` — Active submission already exists (`Homework.ActiveSubmissionExists`), max attempts reached (`Homework.MaxAttemptsReached`), or homework not available (`Homework.InvalidStatus`).

---

### 2.9 Save Answers

#### `PUT /api/tenants/{tenantId}/learning/homeworks/submissions/{submissionId}/answers`

Saves the student's answers without submitting. Supports auto-save.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who started the submission

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
    },
    {
      "questionId": "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "selectedOptionIds": null,
      "textAnswer": "The cell membrane controls what enters and exits the cell."
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `answers` | `Array<AnswerInput>` | Yes | Student's answers to questions. |
| `answers[].questionId` | `Guid` | Yes | The question being answered. |
| `answers[].selectedOptionIds` | `Array<Guid>?` | Conditional | Selected option IDs for MultipleChoice/TrueFalse. |
| `answers[].textAnswer` | `string?` | Conditional | Text answer for ShortAnswer/Essay. |

**Response:** `200 OK`

```json
{
  "submissionId": "sub1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "savedAtUtc": "2026-09-04T10:45:00Z",
  "answeredCount": 2,
  "totalQuestions": 10
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not the submission owner.
- `404 Not Found` — Submission not found (`Homework.SubmissionNotFound`).
- `409 Conflict` — Submission already submitted (`Homework.InvalidStatus`).

---

### 2.10 Submit Answers

#### `POST /api/tenants/{tenantId}/learning/homeworks/submissions/{submissionId}/submit`

Submits the homework. Once submitted, no further changes are allowed.

**Authentication:** Required (Cookie-based)

**Authorization:** The student who started the submission

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "submissionId": "sub1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "status": "Submitted",
  "submittedAtUtc": "2026-09-04T11:00:00Z",
  "isLate": false,
  "autoScore": 45,
  "maxScore": 50
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not the submission owner.
- `404 Not Found` — Submission not found (`Homework.SubmissionNotFound`).
- `409 Conflict` — Late submission not allowed (`Homework.LateSubmissionNotAllowed`).

---

### 2.11 Review Submission

#### `POST /api/tenants/{tenantId}/learning/homeworks/submissions/{submissionId}/review`

Teacher reviews a submitted homework and provides feedback/grades.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `homework.grade`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "feedback": "Good work! Remember to show your work for calculation questions.",
  "marks": [
    {
      "questionId": "q1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "score": 5,
      "feedback": "Correct!"
    },
    {
      "questionId": "q2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "score": 4,
      "feedback": "Well explained, but minor detail missing."
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `feedback` | `string?` | No | Overall submission feedback. Max 2000 characters. |
| `marks` | `Array<MarkInput>` | Yes | Per-question scores and feedback. |
| `marks[].questionId` | `Guid` | Yes | The question being graded. |
| `marks[].score` | `decimal` | Yes | Score awarded for this question. |
| `marks[].feedback` | `string?` | No | Per-question feedback. Max 500 characters. |

**Response:** `200 OK`

```json
{
  "submissionId": "sub1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "status": "Reviewed",
  "finalScore": 45,
  "maxScore": 50,
  "reviewedAtUtc": "2026-09-10T14:00:00Z",
  "reviewedByMembershipId": "m1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `homework.grade` permission or tenant access denied.
- `404 Not Found` — Submission not found (`Homework.SubmissionNotFound`).
- `409 Conflict` — Submission not in a reviewable state (`Homework.InvalidStatus`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `CreateHomeworkRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `CourseId` | `Guid` | Yes | Valid course reference. |
| `SectionId` | `Guid?` | No | Valid section reference. |
| `LessonId` | `Guid?` | No | Valid lesson reference. |
| `Title` | `string` | Yes | 1–200 characters. |
| `Instructions` | `string?` | No | Max 5000 characters. |
| `SubmissionMode` | `HomeworkSubmissionMode` | Yes | Enum value. |
| `AvailableFromUtc` | `DateTime?` | No | UTC timestamp. |
| `AvailableUntilUtc` | `DateTime?` | No | UTC timestamp. |
| `AllowLateSubmissions` | `bool` | No | Default false. |
| `MaxAttempts` | `int` | No | Positive integer, default 1. |
| `QuestionIds` | `List<Guid>?` | Conditional | Required for OnlineQuestions/Hybrid. |
| `RecipientType` | `HomeworkRecipientType` | Yes | Enum value. |
| `RecipientGroupIds` | `List<Guid>?` | Conditional | Required for SpecificGroups. |
| `RecipientStudentIds` | `List<Guid>?` | Conditional | Required for SpecificStudents. |

#### `EditHomeworkRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 characters. |
| `Instructions` | `string?` | No | Max 5000 characters. |
| `SubmissionMode` | `HomeworkSubmissionMode?` | No | Enum value. |
| `AvailableFromUtc` | `DateTime?` | No | UTC timestamp. |
| `AvailableUntilUtc` | `DateTime?` | No | UTC timestamp. |
| `AllowLateSubmissions` | `bool?` | No | Boolean. |
| `MaxAttempts` | `int?` | No | Positive integer. |
| `QuestionIds` | `List<Guid>?` | No | Replace all questions if provided. |
| `RecipientType` | `HomeworkRecipientType?` | No | Enum value. |
| `RecipientGroupIds` | `List<Guid>?` | No | Replace groups if provided. |
| `RecipientStudentIds` | `List<Guid>?` | No | Replace students if provided. |

#### `AnswerInput`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `QuestionId` | `Guid` | Yes | Valid question reference. |
| `SelectedOptionIds` | `List<Guid>?` | Conditional | Required for MultipleChoice/TrueFalse. |
| `TextAnswer` | `string?` | Conditional | Required for ShortAnswer/Essay. Max 10000 characters. |

#### `MarkInput`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `QuestionId` | `Guid` | Yes | Valid question reference. |
| `Score` | `decimal` | Yes | Non-negative, <= question points. |
| `Feedback` | `string?` | No | Max 500 characters. |

---

### Response DTOs

#### `HomeworkSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Homework identifier. |
| `CourseId` | `Guid` | Course reference. |
| `SectionId` | `Guid?` | Section reference. |
| `LessonId` | `Guid?` | Lesson reference. |
| `Title` | `string` | Homework title. |
| `SubmissionMode` | `string` | Mode name. |
| `AvailableFromUtc` | `DateTime?` | Availability start. |
| `AvailableUntilUtc` | `DateTime?` | Deadline. |
| `AllowLateSubmissions` | `bool` | Late submission allowed. |
| `MaxAttempts` | `int` | Maximum attempts. |
| `Status` | `string` | Current status. |
| `QuestionCount` | `int` | Number of questions. |
| `AttachmentCount` | `int` | Number of attachments. |
| `SubmissionCount` | `int` | Number of submissions. |
| `PublishedAtUtc` | `DateTime?` | Publication timestamp. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |

#### `HomeworkDetails`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Homework identifier. |
| `CourseId` | `Guid` | Course reference. |
| `SectionId` | `Guid?` | Section reference. |
| `LessonId` | `Guid?` | Lesson reference. |
| `Title` | `string` | Homework title. |
| `Instructions` | `string?` | Instructions text. |
| `SubmissionMode` | `string` | Mode name. |
| `AvailableFromUtc` | `DateTime?` | Availability start. |
| `AvailableUntilUtc` | `DateTime?` | Deadline. |
| `AllowLateSubmissions` | `bool` | Late submission allowed. |
| `MaxAttempts` | `int` | Maximum attempts. |
| `Status` | `string` | Current status. |
| `PublishedAtUtc` | `DateTime?` | Publication timestamp. |
| `Questions` | `List<HomeworkQuestionDto>` | Questions with options (answers hidden for students). |
| `Attachments` | `List<HomeworkAttachmentDto>` | Supporting files. |
| `RecipientType` | `string` | Recipient type. |
| `RecipientGroupIds` | `List<Guid>?` | Assigned groups. |
| `RecipientStudentIds` | `List<Guid>?` | Assigned students. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |
| `UpdatedAtUtc` | `DateTime` | Last update. |

#### `HomeworkQuestionDto`

| Property | Type | Description |
|----------|------|-------------|
| `QuestionId` | `Guid` | Question ID. |
| `QuestionBankItemId` | `Guid` | Bank item reference. |
| `QuestionText` | `string` | Question text. |
| `Type` | `string` | Question type. |
| `Points` | `int` | Points value. |
| `Options` | `List<QuestionOptionDto>` | Options (isCorrect hidden for students). |

#### `HomeworkAttachmentDto`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Attachment ID. |
| `Title` | `string` | Attachment title. |
| `Role` | `string` | Role (Instructions, ReferenceMaterial, Worksheet, AnswerKey). |
| `FileUrl` | `string` | File URL. |

#### `HomeworkSubmissionSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Submission ID. |
| `HomeworkId` | `Guid` | Homework reference. |
| `StudentId` | `Guid` | Student reference. |
| `AttemptNumber` | `int` | Which attempt this is. |
| `Status` | `string` | Submission status. |
| `StartedAtUtc` | `DateTime` | When submission started. |
| `ExpiresAtUtc` | `DateTime?` | Deadline for this attempt. |
| `SubmittedAtUtc` | `DateTime?` | Submission time. |
| `AutoScore` | `decimal?` | Auto-graded score. |
| `FinalScore` | `decimal?` | Final score after review. |
| `ReviewedByMembershipId` | `Guid?` | Reviewer ID. |
| `ReviewedAtUtc` | `DateTime?` | Review time. |
| `CreatedAtUtc` | `DateTime` | Creation time. |
| `UpdatedAtUtc` | `DateTime` | Last update. |

---

## 4. Enum Reference

### `HomeworkStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Being prepared. Editable. Not visible to students. |
| `2` | `Published` | Live. Visible to recipients. Accepting submissions. |
| `3` | `Closed` | Deadline passed. No new submissions. |
| `4` | `Archived` | Complete. Hidden from students. Preserved for records. |

### `HomeworkSubmissionMode`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `NoSubmission` | Reference material only — no submission required. |
| `2` | `OnlineQuestions` | Students answer questions online. |
| `3` | `FileUpload` | Students upload a file (document, PDF, etc.). |
| `4` | `Hybrid` | Both online questions and file upload. |

### `HomeworkSubmissionStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `InProgress` | Student has started but not yet submitted. |
| `2` | `Submitted` | Submitted and awaiting review. |
| `3` | `NeedsReview` | Flagged for teacher review (e.g., essay question). |
| `4` | `Reviewed` | Graded and feedback provided. |

### `HomeworkRecipientType`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `AllStudents` | All students enrolled in the course. |
| `2` | `SpecificGroups` | Only students in specified groups. |
| `3` | `SpecificStudents` | Only explicitly listed students. |

### `HomeworkAttachmentRole`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Instructions` | Assignment instructions document. |
| `2` | `ReferenceMaterial` | Reference files for students. |
| `3` | `Worksheet` | Printable worksheet. |
| `4` | `AnswerKey` | Teacher answer key (hidden from students). |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Draft-Only Edits**: Homework metadata can only be updated when status is `Draft`.
2. **Publish Prerequisite**: Homework must be `Draft` to be published.
3. **Close Prerequisite**: Homework must be `Published` to be closed.
4. **Archive Prerequisite**: Homework must be `Closed` to be archived.
5. **Availability Window**: If `availableFromUtc` or `availableUntilUtc` are set, submissions are only allowed within that window.
6. **Late Submission Policy**: If `allowLateSubmissions` is false and deadline has passed, submissions are rejected.
7. **Max Attempts**: Students cannot start a new submission if they have reached `maxAttempts`.
8. **Active Submission**: Students can only have one `InProgress` submission at a time.
9. **Submitted Immutability**: Once submitted, answers cannot be changed.
10. **Questions Required**: OnlineQuestions and Hybrid modes require at least one question.
11. **Points Consistency**: Awarded marks cannot exceed the question's point value.

### Frontend Validation Recommendations

- Validate that `availableUntilUtc` is after `availableFromUtc`.
- Warn if `maxAttempts` is less than 1.
- Require at least one question for OnlineQuestions mode.
- Show deadline prominently and warn as deadline approaches.
- Implement auto-save with debouncing (save every 30-60 seconds).
- For students, show remaining attempts count.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Homework.HomeworkNotFound` | 404 | Homework does not exist or belongs to another tenant. | Refresh homework list. |
| `Homework.SubmissionNotFound` | 404 | Submission does not exist. | Refresh submission list. |
| `Homework.InvalidStatus` | 409 | Operation not allowed for current homework/submission status. | Check current status. |
| `Homework.MaxAttemptsReached` | 409 | Student has used all allowed attempts. | No action available. |
| `Homework.LateSubmissionNotAllowed` | 409 | Late submission rejected. | Deadline has passed. |
| `Homework.RevisionNotFound` | 404 | Question revision not found. | Refresh question. |
| `Homework.ActiveSubmissionExists` | 409 | Student already has an in-progress submission. | Resume existing submission. |
| `Tenancy.AccessDenied` | 403 | Tenant context mismatch. | Verify tenant selection. |

---

## 7. Frontend Integration Best Practices

1. **Homework Builder**: Create a multi-step form: (1) Basic info, (2) Questions, (3) Recipients, (4) Review & Publish.
2. **Question Picker**: Integrate a searchable question picker from the question bank. Show question preview on hover.
3. **Status Badges**: Color-code homework status (gray=Draft, green=Published, orange=Closed, muted=Archived).
4. **Deadline Display**: Show countdown timer when homework deadline is within 24 hours.
5. **Submission Timer**: For timed homework, show remaining time prominently.
6. **Auto-Save Indicator**: Show "Saving..." / "Saved" indicator during auto-save.
7. **Attempts Counter**: Show "Attempt 1 of 2" or "Last attempt" to students.
8. **File Upload UI**: For FileUpload mode, implement drag-and-drop file selection with progress.
9. **Grading Dashboard**: Build a submission list view with filters for Submitted, NeedsReview, Reviewed.
10. **Inline Grading**: For grading, show student answers alongside rubrics. Implement bulk grading for common questions.
11. **Late Flag**: Visually mark late submissions (after deadline but within grace period or allowed).
12. **Feedback Display**: Show feedback in a dedicated section after grading, with per-question breakdown.
