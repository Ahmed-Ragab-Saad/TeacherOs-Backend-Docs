# AI Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/learning/ai`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permission `ai.generate` required for creating AI generation jobs.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List AI Jobs](#21-list-ai-jobs)
   - 2.2 [Create AI Job](#22-create-ai-job)
   - 2.3 [Get AI Job by ID](#23-get-ai-job-by-id)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The AI module handles asynchronous content generation using AI providers (such as OpenAI). When a teacher requests AI-generated content (course outlines, question bank drafts, homework drafts, exam drafts), a job is created and processed in the background. The frontend polls for job completion and can then review and accept the generated draft.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **AIGenerationJob** | An asynchronous job representing a request to generate content using AI. |
| **AIGenerationTargetType** | The type of content being generated: CourseOutline, QuestionBankDraft, HomeworkDraft, ExamDraft. |
| **AIGenerationJobStatus** | The current state: Pending, Running, Succeeded, Failed, Cancelled. |
| **Parameters** | JSON configuration for the generation request (subject, grade level, topic, etc.). |
| **Generated Draft** | The AI-produced content in JSON format, ready for review and editing. |
| **Async Processing** | Jobs are processed asynchronously. The `POST` endpoint returns immediately with `202 Accepted`. |

### Generation Flow

```
Teacher Initiates AI Generation
        |
        v
Create AI Job (POST → 202 Accepted)
  (Job created with status: Pending)
        |
        v
Background Processing
  (AI provider generates content)
        |
        +-- Success --> Job status: Succeeded
        |             GeneratedDraftJson populated
        v
Teacher Polls for Completion
        |
        v
Teacher Reviews Draft
        |
        v
Teacher Accepts/Edits Draft
  (Creates actual content in target module)
```

### Supported Generation Targets

| Target Type | Description | Output Format |
|-------------|-------------|---------------|
| `CourseOutline` | Generates a course structure with sections and lessons | Course structure JSON |
| `QuestionBankDraft` | Generates quiz/exam questions | Question array JSON |
| `HomeworkDraft` | Generates a homework assignment | Homework structure JSON |
| `ExamDraft` | Generates an exam with questions | Exam structure JSON |

### Important Frontend Implications

- **Async Response**: The `POST` endpoint returns `202 Accepted` immediately. Do not wait for completion.
- **Polling Required**: The frontend must poll `GET /ai/jobs/{id}` to check job status until it reaches `Succeeded` or `Failed`.
- **Draft Review**: The generated content is a draft. Teachers should review and edit before creating the actual content in the target module.
- **Failure Handling**: If a job fails, the `failureReason` field contains details. Teachers can retry with adjusted parameters.
- **No Cancel UI Needed**: Cancellation is supported but typically not exposed in the UI unless the generation takes very long.

---

## 2. API Endpoints Reference

### 2.1 List AI Jobs

#### `GET /api/tenants/{tenantId}/learning/ai/jobs`

Retrieves a paginated list of AI generation jobs for the tenant.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `ai.generate`

**Query Parameters:**
- `targetType` (AIGenerationTargetType, optional): Filter by target type.
- `status` (AIGenerationJobStatus, optional): Filter by status.
- `page` (int, optional, default `1`): Page number.
- `pageSize` (int, optional, default `20`): Items per page (max 100).

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "j1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "targetType": "QuestionBankDraft",
      "status": "Succeeded",
      "failureReason": null,
      "createdAtUtc": "2026-09-04T10:00:00Z",
      "updatedAtUtc": "2026-09-04T10:02:30Z"
    },
    {
      "id": "j2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "targetType": "CourseOutline",
      "status": "Running",
      "failureReason": null,
      "createdAtUtc": "2026-09-04T10:05:00Z",
      "updatedAtUtc": "2026-09-04T10:05:30Z"
    },
    {
      "id": "j3c4d5e6-f7a8-9012-b3c4-d5e6f7089012",
      "targetType": "HomeworkDraft",
      "status": "Failed",
      "failureReason": "AI provider returned an invalid response format.",
      "createdAtUtc": "2026-09-04T09:30:00Z",
      "updatedAtUtc": "2026-09-04T09:31:00Z"
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
- `403 Forbidden` — Missing `ai.generate` permission or tenant access denied.

---

### 2.2 Create AI Job

#### `POST /api/tenants/{tenantId}/learning/ai/jobs`

Initiates an AI content generation job. Returns immediately with `202 Accepted`.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `ai.generate`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "targetType": 2,
  "parameters": {
    "subjectId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "gradeLevel": "Grade 8",
    "topic": "Photosynthesis",
    "difficulty": "intermediate",
    "questionCount": 10,
    "questionTypes": [1, 2, 4],
    "includeExplanation": true
  },
  "contextSourceIds": ["c1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `targetType` | `AIGenerationTargetType` | Yes | The type of content to generate. |
| `parameters` | `object` | Yes | Generation parameters. Structure varies by target type. |
| `contextSourceIds` | `Array<Guid>?` | No | IDs of existing content to use as reference (e.g., course ID for outline). |

#### Generation Parameters by Target Type

**CourseOutline:**
```json
{
  "title": "Introduction to Algebra",
  "subjectId": "s1a2b3c4-...",
  "gradeLevel": "Grade 7",
  "sectionCount": 5,
  "lessonsPerSection": 3
}
```

**QuestionBankDraft:**
```json
{
  "subjectId": "s1a2b3c4-...",
  "gradeLevel": "Grade 8",
  "topic": "Photosynthesis",
  "difficulty": "intermediate",
  "questionCount": 10,
  "questionTypes": [1, 2, 4],
  "includeExplanation": true
}
```

**HomeworkDraft:**
```json
{
  "courseId": "c1a2b3c4-...",
  "title": "Chapter 5 Homework",
  "difficulty": "intermediate",
  "questionCount": 15,
  "allowLateSubmission": true
}
```

**ExamDraft:**
```json
{
  "courseId": "c1a2b3c4-...",
  "title": "Midterm Exam",
  "durationMinutes": 60,
  "questionCount": 25,
  "difficulty": "intermediate",
  "includeEssay": true
}
```

**Response:** `202 Accepted`

```json
{
  "jobId": "j1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "targetType": "QuestionBankDraft",
  "status": "Pending",
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

| Property | Type | Description |
|----------|------|-------------|
| `jobId` | `Guid` | The newly created job ID. Poll this for completion. |
| `targetType` | `string` | Target type name. |
| `status` | `string` | Initial status (usually `Pending`). |
| `createdAtUtc` | `DateTime` | Job creation timestamp. |

**Possible Errors:**
- `400 Bad Request` — Validation error (`AI.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `ai.generate` permission or tenant access denied.
- `409 Conflict` — AI provider not configured (`AI.ProviderNotConfigured`).
- `422 UnprocessableEntity` — Source not authorized (`AI.SourceNotAuthorized`).

---

### 2.3 Get AI Job by ID

#### `GET /api/tenants/{tenantId}/learning/ai/jobs/{jobId}`

Retrieves the full details of an AI generation job, including the generated draft when successful.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `ai.generate`

**Response:** `200 OK`

**Success Example (Succeeded):**
```json
{
  "id": "j1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "targetType": "QuestionBankDraft",
  "status": "Succeeded",
  "parameters": {
    "subjectId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
    "gradeLevel": "Grade 8",
    "topic": "Photosynthesis",
    "difficulty": "intermediate",
    "questionCount": 10,
    "questionTypes": [1, 2, 4],
    "includeExplanation": true
  },
  "promptSnapshot": "Generate 10 questions about photosynthesis for Grade 8 students...",
  "generatedDraftJson": "{\"questions\":[{\"questionText\":\"What is photosynthesis?\",\"type\":2,...},...]}",
  "failureReason": null,
  "createdAtUtc": "2026-09-04T10:00:00Z",
  "updatedAtUtc": "2026-09-04T10:02:30Z"
}
```

**Failure Example:**
```json
{
  "id": "j3c4d5e6-f7a8-9012-b3c4-d5e6f7089012",
  "targetType": "HomeworkDraft",
  "status": "Failed",
  "parameters": {...},
  "promptSnapshot": "Generate a homework assignment...",
  "generatedDraftJson": null,
  "failureReason": "AI provider returned an invalid response format.",
  "createdAtUtc": "2026-09-04T09:30:00Z",
  "updatedAtUtc": "2026-09-04T09:31:00Z"
}
```

**Pending/Running Example:**
```json
{
  "id": "j2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
  "targetType": "CourseOutline",
  "status": "Running",
  "parameters": {...},
  "promptSnapshot": "Generate a course outline...",
  "generatedDraftJson": null,
  "failureReason": null,
  "createdAtUtc": "2026-09-04T10:05:00Z",
  "updatedAtUtc": "2026-09-04T10:05:30Z"
}
```

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Job identifier. |
| `targetType` | `string` | Target type name. |
| `status` | `string` | Current status. |
| `parameters` | `object` | The parameters used for generation. |
| `promptSnapshot` | `string` | The prompt sent to the AI (for debugging/audit). |
| `generatedDraftJson` | `string?` | The generated content JSON (only if `Succeeded`). |
| `failureReason` | `string?` | Error details (only if `Failed`). |
| `createdAtUtc` | `DateTime` | Job creation timestamp. |
| `updatedAtUtc` | `DateTime` | Last status change timestamp. |

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `ai.generate` permission or tenant access denied.
- `404 Not Found` — Job not found (`AI.JobNotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `CreateAIJobRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `TargetType` | `AIGenerationTargetType` | Yes | Enum value. |
| `Parameters` | `object` | Yes | Target-specific parameters object. |
| `ContextSourceIds` | `List<Guid>?` | No | Reference content IDs. |

---

### Response DTOs

#### `AIJobSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Job identifier. |
| `TargetType` | `string` | Target type name. |
| `Status` | `string` | Current status. |
| `FailureReason` | `string?` | Error details if failed. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |
| `UpdatedAtUtc` | `DateTime` | Last update timestamp. |

#### `AIJobDetails`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Job identifier. |
| `TargetType` | `string` | Target type name. |
| `Status` | `string` | Current status. |
| `Parameters` | `string` | Parameters JSON string. |
| `PromptSnapshot` | `string` | The prompt sent to AI. |
| `GeneratedDraftJson` | `string?` | Generated content JSON (if succeeded). |
| `FailureReason` | `string?` | Error details (if failed). |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |
| `UpdatedAtUtc` | `DateTime` | Last update timestamp. |

---

## 4. Enum Reference

### `AIGenerationTargetType`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `CourseOutline` | Generate a course structure with sections and lessons. |
| `2` | `QuestionBankDraft` | Generate quiz/exam questions for the question bank. |
| `3` | `HomeworkDraft` | Generate a homework assignment. |
| `4` | `ExamDraft` | Generate an exam with questions. |

### `AIGenerationJobStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Pending` | Job created, waiting for processing. |
| `2` | `Running` | AI is generating content. |
| `3` | `Succeeded` | Generation complete. `generatedDraftJson` is available. |
| `4` | `Failed` | Generation failed. `failureReason` contains details. |
| `5` | `Cancelled` | Job was cancelled by user or system. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Provider Configuration**: An AI provider must be configured for the tenant. If not configured, creation fails with `AI.ProviderNotConfigured`.
2. **Context Authorization**: If `contextSourceIds` are provided, the user must have access to that content.
3. **Parameter Validation**: Parameters are validated per target type. Required fields depend on the target.
4. **Job Ownership**: Jobs are scoped to the creating user and tenant.
5. **Polling Limit**: No hard limit on polling frequency, but implement reasonable backoff.

### Frontend Validation Recommendations

- Validate required parameters before submission.
- Show clear labels for each parameter.
- Provide sensible defaults for optional parameters.
- Handle the `202 Accepted` response immediately — do not wait for completion.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `AI.JobNotFound` | 404 | AI job does not exist or belongs to another tenant. | Refresh job list. |
| `AI.ProviderNotConfigured` | 409 | AI provider not set up for tenant. | Contact administrator to configure AI. |
| `AI.JobNotSucceeded` | 400 | Attempted to access draft from a non-succeeded job. | Wait for job to succeed or retry. |
| `AI.SourceNotAuthorized` | 422 | Context source content is not accessible. | Check permissions or select different content. |
| `Tenancy.AccessDenied` | 403 | Tenant context mismatch. | Verify tenant selection. |

---

## 7. Frontend Integration Best Practices

1. **Generation Wizard**: Build a step-by-step wizard for each target type:
   - Step 1: Select target type
   - Step 2: Configure parameters
   - Step 3: Review and submit
   - Step 4: Wait for completion (with polling)
   - Step 5: Review generated draft

2. **Polling Implementation**:
   ```javascript
   async function pollJobStatus(jobId, onUpdate) {
     while (true) {
       const job = await fetchJob(jobId);
       onUpdate(job);
       if (job.status === 'Succeeded' || job.status === 'Failed' || job.status === 'Cancelled') {
         break;
       }
       await sleep(2000); // Poll every 2 seconds
     }
   }
   ```

3. **Progress Indicator**: Show a progress UI while generating:
   - "AI is thinking..." with animated dots
   - "Generating questions..." etc.

4. **Draft Preview**: Parse `generatedDraftJson` and display a preview of the generated content:
   - For questions: Show question cards with options
   - For course outline: Show section/lesson tree
   - For homework/exam: Show structure with question count

5. **Edit Before Accept**: Allow teachers to edit the generated draft before creating the actual content. Open a pre-filled form with the AI content.

6. **Retry on Failure**: When a job fails, show the failure reason and offer a "Retry" button with adjusted parameters.

7. **Job History**: Show a history of generation jobs in the UI so teachers can revisit previous generations.

8. **Contextual Generation**: When generating from a course context, pre-fill subject and grade level from the course.

9. **Status Badges**: Color-code job status:
   - Gray = Pending
   - Blue = Running
   - Green = Succeeded
   - Red = Failed
   - Yellow = Cancelled

10. **Accept Draft Flow**: When a teacher accepts a draft, parse `generatedDraftJson` and create the actual entity in the target module (course, question bank, homework, or exam).

11. **Partial Failure Handling**: If the AI generates partially invalid content, still show the valid parts and highlight any issues.

12. **Cost Estimation**: If available, show estimated AI usage/cost to teachers before generation.
