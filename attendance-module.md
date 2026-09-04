# Attendance Module — Frontend Integration Guide

**Base URLs:**
- Session listing: `GET /api/tenants/{tenantId}/attendance/sessions`
- Session attendance: `/api/tenants/{tenantId}/sessions/{sessionId}/attendance`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Specific permissions: `attendance.view` for read operations, `attendance.record` for draft/submit/discard, `attendance.correct` for reopen corrections.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Attendance Session Listing](#21-attendance-session-listing)
   - 2.2 [Session Attendance Management](#22-session-attendance-management)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Attendance module manages daily/class attendance recording for sessions. It uses a draft/submit workflow with a revision system. Each session can have at most one attendance sheet, which evolves through states: **Draft** → **Submitted** → (reopened) → **Draft** again.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **AttendanceSheet** | The parent record for a session's attendance. One per session. Contains the current status, revision number, and submitted metadata. |
| **AttendanceEntry** | Individual student attendance record within a sheet: student, status (Present/Absent/Late/ExcusedAbsent), optional note. |
| **AttendanceEntryChange** | Audit trail record capturing each modification to an entry: who changed it, from what status to what, and why. |
| **Draft State** | Teachers can save incomplete attendance (not all students need to be marked). Drafts are editable. |
| **Submitted State** | All roster students must be marked before submission. Submitted sheets lock session operational details (teacher/branch/time cannot be changed). Reopening creates a new revision. |
| **Reopen / Correction** | Requires `attendance.correct` permission and a reason. Creates a new draft revision with full edit capability. |
| **RowVersion** | Concurrency token (base64-encoded byte array). Required for all write operations. Must be sent back on every subsequent request to detect concurrent modifications. |

### Lifecycle & State Transitions

```
            Save Draft (any number of times)
         +------------------------------------------+
         |                                          |
         v                                          |
    +---------+                                     |
    | Draft 1 | <----------------------------------+
    +----+----+       Reopen (new revision)
         |                         |
         | Submit                  | Save Draft
         | (all must be marked)    |
         v                         v
   +-----------------+      +--------+
   | Submitted Rev 1 | ----> | Draft 2|  (new revision #)
   +-----------------+ Reopen +--------+
                              |
                              | Submit
                              v
                       +-----------------+
                       | Submitted Rev 2|
                       +-----------------+
```

### Attendance Lock

When attendance is **submitted** for a session:
- The session's teacher, branch, and scheduled time become **locked** — any attempt to reschedule, substitute teacher, or relocate via the Scheduling module returns `409 Conflict` (`Sessions.LockedBySubmittedAttendance`).
- The session must be **reopened** before those operational fields can be changed.

### Important Frontend Implications

- **Concurrency Control**: The `rowVersion` field must be sent with every write request. If another user modified the sheet concurrently, the backend returns `409 Conflict` (`Attendance.ConcurrencyConflict`).
- **All Students Must Be Marked**: Submission requires every student in the roster to have a status. Draft saves allow partial marking.
- **Correction Reason**: Reopening a submitted sheet requires a `reason` field (stored in the audit trail).
- **Attendance State**: `AttendanceSessionResponse.AttendanceState` tells the frontend whether to show "Take Attendance", "Edit Attendance", or "View Attendance" for each session.

---

## 2. API Endpoints Reference

### 2.1 Attendance Session Listing

#### `GET /api/tenants/{tenantId}/attendance/sessions`

Lists all sessions within a date range with their attendance state and summary counts. Used to populate the attendance-taking dashboard/calendar.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.view`

**Query Parameters:**
- `from` (DateTime, required): Start of the range (UTC ISO 8601).
- `to` (DateTime, required): End of the range (UTC ISO 8601). Maximum span: **90 days**.
- `branchId` (Guid, optional): Filter by branch.
- `teacherId` (Guid, optional): Filter by assigned teacher.
- `groupId` (Guid, optional): Filter by group.
- `state` (AttendanceState integer, optional): Filter by `1` (NotStarted), `2` (Draft), or `3` (Submitted).

**Response:** `200 OK`

```json
[
  {
    "sessionId": "7e4a1b2c-3d5e-6f7a-8b9c-0d1e2f3a4b5c",
    "scheduledStartUtc": "2026-09-04T08:00:00Z",
    "scheduledEndUtc": "2026-09-04T09:00:00Z",
    "sessionStatus": 1,
    "groupId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "groupName": "Grade 10 - Mathematics - Section A",
    "teacherId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
    "teacherName": "Dr. Sarah Connor",
    "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
    "branchName": "Main Campus",
    "attendanceState": 2,
    "presentCount": 20,
    "absentCount": 2,
    "lateCount": 1,
    "excusedAbsentCount": 1,
    "totalRosterCount": 24
  }
]
```

**AttendanceState Values:**
- `1` = NotStarted (no draft exists yet)
- `2` = Draft (draft exists, not yet submitted)
- `3` = Submitted (fully submitted)

**Possible Errors:**
- `400 Bad Request` — Date range exceeds 90 days (`Attendance.RangeTooLarge`).
- `401 Unauthorized` / `403 Forbidden` — Permission denied.

---

### 2.2 Session Attendance Management

#### `GET /api/tenants/{tenantId}/sessions/{sessionId}/attendance`

Retrieves the full attendance sheet for a session: metadata, roster with current statuses, and recent change audit trail.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.view`

**Path Parameters:**
- `tenantId` (Guid, required)
- `sessionId` (Guid, required)

**Response:** `200 OK`

```json
{
  "sessionId": "7e4a1b2c-3d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "groupId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "groupName": "Grade 10 - Mathematics - Section A",
  "scheduledStartUtc": "2026-09-04T08:00:00Z",
  "scheduledEndUtc": "2026-09-04T09:00:00Z",
  "teacherId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "teacherName": "Dr. Sarah Connor",
  "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
  "branchName": "Main Campus",
  "sheetId": "f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "status": 1,
  "revisionNumber": 1,
  "submittedAtUtc": null,
  "submittedByMembershipId": null,
  "rowVersion": "AAAAAAAAB9I=",
  "presentCount": 20,
  "absentCount": 0,
  "lateCount": 0,
  "excusedAbsentCount": 0,
  "unmarkedCount": 4,
  "totalRosterCount": 24,
  "roster": [
    {
      "studentId": "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
      "studentCode": "STU-2024-001",
      "fullName": "Ahmed Hassan",
      "photoUrl": "https://assets.teacheros.io/photos/ahmed.jpg",
      "attendanceStatus": 1,
      "note": null
    },
    {
      "studentId": "s2t3u4v5-w6x7-8901-bcde-f23456789012",
      "studentCode": "STU-2024-002",
      "fullName": "Fatima Ali",
      "photoUrl": null,
      "attendanceStatus": null,
      "note": null
    }
  ],
  "recentChanges": [
    {
      "id": "c1h2a3n4-g5e6-7890-a1b2-c3d4e5f60789",
      "attendanceEntryId": "e1n2t3r4-y5o6-7890-a1b2-c3d4e5f60789",
      "studentId": "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
      "studentName": "Ahmed Hassan",
      "previousStatus": 2,
      "newStatus": 1,
      "reason": "Parent provided medical certificate",
      "changedByMembershipId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
      "changedAtUtc": "2026-09-04T09:15:00Z"
    }
  ]
}
```

**Possible Errors:**
- `401 Unauthorized` / `403 Forbidden` — Permission denied.
- `404 Not Found` — Session not found (`Attendance.SessionNotFound`).

---

#### `PUT /api/tenants/{tenantId}/sessions/{sessionId}/attendance`

Saves or updates the attendance draft. Supports partial saves (not all students must be marked).

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.record`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Path Parameters:**
- `tenantId` (Guid, required)
- `sessionId` (Guid, required)

**Request Body:**

```json
{
  "rowVersion": "AAAAAAAAB9I=",
  "entries": [
    {
      "studentId": "s1t2u3d4-e5f6-7890-abcd-ef1234567890",
      "status": 1,
      "note": null
    },
    {
      "studentId": "s2t3u4v5-w6x7-8901-bcde-f23456789012",
      "status": 2,
      "note": "Absent without prior notice"
    },
    {
      "studentId": "s3t4u5v6-x7y8-9012-cdef-345678901234",
      "status": 3,
      "note": "Arrived 10 minutes late"
    },
    {
      "studentId": "s4t5u6v7-x8y9-0123-def0-456789012345",
      "status": 4,
      "note": "Family emergency — approved"
    }
  ],
  "correctionReason": null
}
```

**Notes:**
- `rowVersion` is required. Omit on the very first save (create scenario) by passing `null`.
- `entries` replaces the entire draft state. Submit all student statuses on each save to avoid losing previous marks.
- `correctionReason` is only required when modifying a submitted attendance sheet (after reopen). Otherwise it is optional.

**Response:** `200 OK` — Updated `SessionAttendanceResponse` with new `rowVersion`

**Possible Errors:**
- `400 Bad Request` — Session is cancelled (`Attendance.SessionCancelled`).
- `401 Unauthorized` / `403 Forbidden` — Permission denied.
- `404 Not Found` — Session not found.
- `409 Conflict` — Concurrency conflict (`Attendance.ConcurrencyConflict`).

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/attendance/submit`

Submits the attendance sheet. All students in the roster must have a status marked. After submission, the session is locked from scheduling modifications.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.record`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "rowVersion": "AAAAAAAAB9I="
}
```

**Response:** `200 OK` — Updated `SessionAttendanceResponse` with `status: 2` (Submitted), new `rowVersion`, and `submittedAtUtc` / `submittedByMembershipId` populated.

**Possible Errors:**
- `400 Bad Request` — Not all students are marked (`Attendance.IncompleteRoster`), or session is cancelled.
- `401 Unauthorized` / `403 Forbidden` — Permission denied.
- `404 Not Found` — Session not found.
- `409 Conflict` — Already submitted (`Attendance.AlreadySubmitted`) or concurrency conflict.

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/attendance/reopen`

Reopens a submitted attendance sheet, creating a new draft revision. Requires `attendance.correct` permission and a correction reason.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.correct`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "rowVersion": "AAAAAAAAB9I=",
  "reason": "Parent provided medical certificate for the absent student"
}
```

**Response:** `200 OK` — Updated `SessionAttendanceResponse` with `status: 1` (Draft), incremented `revisionNumber`, new `rowVersion`, and `submittedAtUtc` / `submittedByMembershipId` cleared.

**Possible Errors:**
- `400 Bad Request` — Sheet is not submitted (`Attendance.NotSubmitted`).
- `400 Bad Request` — Reason is empty/missing (`Attendance.CorrectionReasonRequired`).
- `401 Unauthorized` / `403 Forbidden` — Permission denied (requires `attendance.correct`).
- `404 Not Found` — Session not found.
- `409 Conflict` — Concurrency conflict.

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/attendance/discard`

Discards the current draft and resets attendance to the last submitted state. Only available if the sheet has been submitted at least once (reopened drafts can be discarded back to the submitted state).

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `attendance.record`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "rowVersion": "AAAAAAAAB9I="
}
```

**Response:** `200 OK` — Empty body (refresh the attendance sheet by re-calling `GET /attendance`).

**Possible Errors:**
- `400 Bad Request` — Cannot discard a sheet that has never been submitted (`Attendance.CannotDiscardSubmittedHistory`).
- `401 Unauthorized` / `403 Forbidden` — Permission denied.
- `404 Not Found` — Session not found.
- `409 Conflict` — Concurrency conflict.

---

## 3. Request & Response DTOs

### Request DTOs

#### `SaveAttendanceDraftRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `rowVersion` | `string?` | Yes (after first save) | Concurrency token. Pass `null` only on first save. |
| `entries` | `Array<AttendanceEntryRequest>` | Yes | Full list of student attendance entries. |
| `correctionReason` | `string?` | Conditional | Required when modifying a submitted sheet (after reopen). |

#### `AttendanceEntryRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `studentId` | `Guid` | Yes | Student identifier. |
| `status` | `AttendanceStatus` integer | Yes | `1`=Present, `2`=Absent, `3`=Late, `4`=ExcusedAbsent. |
| `note` | `string?` | No | Free-text note (e.g., reason for absence). |

#### `SubmitAttendanceRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `rowVersion` | `string?` | Yes | Current concurrency token. |

#### `ReopenAttendanceRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `rowVersion` | `string?` | Yes | Current concurrency token. |
| `reason` | `string` | Yes | Non-empty reason for reopening (stored in audit trail). |

#### `DiscardAttendanceDraftRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `rowVersion` | `string?` | Yes | Current concurrency token. |

### Response DTOs

#### `AttendanceSessionResponse`

Summary of a session's attendance state for listing.

| Property | Type | Description |
|----------|------|-------------|
| `sessionId` | `Guid` | Session identifier. |
| `scheduledStartUtc` | `string` | Session start time (UTC). |
| `scheduledEndUtc` | `string` | Session end time (UTC). |
| `sessionStatus` | `SessionStatus` integer | `1`=Scheduled, `2`=Completed, `3`=Cancelled. |
| `groupId` | `Guid` | Group identifier. |
| `groupName` | `string` | Group display name. |
| `teacherId` | `Guid?` | Assigned teacher. |
| `teacherName` | `string?` | Teacher's full name. |
| `branchId` | `Guid` | Branch identifier. |
| `branchName` | `string` | Branch display name. |
| `attendanceState` | `AttendanceState` integer | `1`=NotStarted, `2`=Draft, `3`=Submitted. |
| `presentCount` | `integer` | Count of Present students. |
| `absentCount` | `integer` | Count of Absent students. |
| `lateCount` | `integer` | Count of Late students. |
| `excusedAbsentCount` | `integer` | Count of ExcusedAbsent students. |
| `totalRosterCount` | `integer` | Total students in the roster. |

#### `SessionAttendanceResponse`

Full attendance sheet for a session.

| Property | Type | Description |
|----------|------|-------------|
| `sessionId` | `Guid` | Session identifier. |
| `groupId` | `Guid` | Group identifier. |
| `groupName` | `string` | Group display name. |
| `scheduledStartUtc` | `string` | UTC start time. |
| `scheduledEndUtc` | `string` | UTC end time. |
| `teacherId` | `Guid?` | Assigned teacher. |
| `teacherName` | `string?` | Teacher name. |
| `branchId` | `Guid` | Branch identifier. |
| `branchName` | `string` | Branch name. |
| `sheetId` | `Guid?` | Attendance sheet ID. |
| `status` | `AttendanceSheetStatus?` integer | `1`=Draft, `2`=Submitted. `null` if not yet started. |
| `revisionNumber` | `integer` | Increments each time the sheet is reopened. |
| `submittedAtUtc` | `string?` | UTC timestamp when last submitted. |
| `submittedByMembershipId` | `Guid?` | Membership ID of the user who submitted. |
| `rowVersion` | `string?` | Concurrency token (send with every write). |
| `presentCount` | `integer` | Current Present count. |
| `absentCount` | `integer` | Current Absent count. |
| `lateCount` | `integer` | Current Late count. |
| `excusedAbsentCount` | `integer` | Current ExcusedAbsent count. |
| `unmarkedCount` | `integer` | Students not yet marked. |
| `totalRosterCount` | `integer` | Total roster size. |
| `roster` | `Array<RosterStudentResponse>` | Student list with attendance statuses. |
| `recentChanges` | `Array<AttendanceEntryChangeResponse>` | Audit trail of recent changes. |

#### `RosterStudentResponse`

| Property | Type | Description |
|----------|------|-------------|
| `studentId` | `Guid` | Student ID. |
| `studentCode` | `string` | Student enrollment code. |
| `fullName` | `string` | Full name. |
| `photoUrl` | `string?` | Profile photo URL. |
| `attendanceStatus` | `AttendanceStatus?` integer | `1`=Present, `2`=Absent, `3`=Late, `4`=ExcusedAbsent. `null` if not yet marked. |
| `note` | `string?` | Optional note. |

#### `AttendanceEntryChangeResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Change record ID. |
| `attendanceEntryId` | `Guid` | The entry that was changed. |
| `studentId` | `Guid` | Student ID. |
| `studentName` | `string` | Student full name. |
| `previousStatus` | `AttendanceStatus` integer | Status before the change. |
| `newStatus` | `AttendanceStatus` integer | Status after the change. |
| `reason` | `string` | Correction reason provided when reopening. |
| `changedByMembershipId` | `Guid` | Who made the change. |
| `changedAtUtc` | `string` | UTC timestamp of the change. |

---

## 4. Enum Reference

### `AttendanceState` (Session Listing Filter)

| Value | Name | Description |
|-------|------|-------------|
| `1` | `NotStarted` | No attendance record exists for this session yet. |
| `2` | `Draft` | A draft attendance record exists but is not yet submitted. |
| `3` | `Submitted` | Attendance has been submitted. |

### `AttendanceSheetStatus` (Sheet Lifecycle)

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Sheet is in draft mode; can be edited. |
| `2` | `Submitted` | Sheet is finalized; session is locked. |

### `AttendanceStatus` (Per-Student Status)

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Present` | Student was present. |
| `2` | `Absent` | Student was absent without excuse. |
| `3` | `Late` | Student arrived late. |
| `4` | `ExcusedAbsent` | Student was absent with an approved excuse. |

### `SessionStatus` (From Scheduling Module)

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Scheduled` | Session is planned. |
| `2` | `Completed` | Session has taken place. |
| `3` | `Cancelled` | Session was cancelled; attendance is not allowed. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **All Students Must Be Marked for Submission**: `POST /submit` fails with `Attendance.IncompleteRoster` if any student has `null` status.
2. **Student Must Be in Roster**: Entries can only be saved for students currently enrolled in the group.
3. **Correction Reason Required**: `POST /reopen` and `PUT /attendance` (when modifying a submitted sheet) require a non-empty `reason`.
4. **Concurrency Control**: Every write operation requires the current `rowVersion`. If the server state has changed since the client loaded it, `409 Conflict` is returned.
5. **No Discard Without Submission History**: `POST /discard` fails if the sheet has never been submitted (`Attendance.CannotDiscardSubmittedHistory`).
6. **Session Cancelled Block**: Attendance cannot be recorded for cancelled sessions (`Attendance.SessionCancelled`).
7. **Submitted Sheet Block**: Once submitted, a sheet must be reopened before entries can be modified.
8. **Revision Number**: Increments on each reopen. `revisionNumber` is informational for the frontend to show "Revision 2", "Revision 3", etc.

### Frontend Validation Recommendations

- Show a progress indicator: "18/24 students marked". Disable the Submit button until `unmarkedCount === 0`.
- Always send the complete `entries` array on each `PUT /attendance` — the backend replaces the draft state entirely.
- On the reopen confirmation dialog, require the user to enter a non-empty `reason`.
- Handle `409 Conflict` gracefully: reload the sheet and show a message "Attendance was modified by another user. Your changes have been discarded."

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Attendance.NotFound` | 404 | Attendance sheet does not exist. | Sheet may not have been initialized. |
| `Attendance.SessionNotFound` | 404 | Session does not exist. | Refresh the calendar view. |
| `Attendance.SessionCancelled` | 400 | Cannot record attendance for a cancelled session. | No action possible. |
| `Attendance.IncompleteRoster` | 400 | Not all students are marked. | Highlight unmarked students in the UI. |
| `Attendance.StudentNotInRoster` | 400 | Entry for a student not in the current roster. | Verify student is still enrolled. |
| `Attendance.AlreadySubmitted` | 409 | Sheet was already submitted. | Reopen first. |
| `Attendance.NotSubmitted` | 400 | Cannot reopen a sheet that is not submitted. | Sheet is already in draft state. |
| `Attendance.CorrectionReasonRequired` | 400 | Reason field missing on reopen. | Prompt for correction reason. |
| `Attendance.ConcurrencyConflict` | 409 | Concurrent modification detected. | Reload sheet and retry changes. |
| `Attendance.CannotDiscardSubmittedHistory` | 400 | Sheet has never been submitted; nothing to discard to. | Use the cancel button on the draft editor instead. |
| `Attendance.SessionLockedBySubmittedAttendance` | 409 | Returned by the Scheduling module when session is locked. | Reopen attendance first. |
| `Attendance.RangeTooLarge` | 400 | Date range exceeds 90 days. | Narrow the filter range. |

---

## 7. Frontend Integration Best Practices

1. **Attendance Dashboard**: Use `GET /attendance/sessions?from=...&to=...` to build a calendar/list view. Group sessions by date and show attendance state using color coding (gray=NotStarted, yellow=Draft, green=Submitted).
2. **Attendance Entry UI**: Render the roster as a list of student cards/rows. Each row has a status selector (Present/Absent/Late/Excused) and an optional note field. Show a live progress counter (e.g., "20/24 marked").
3. **Submit Flow**: The Submit button should be disabled until `unmarkedCount === 0`. Show a confirmation dialog before submitting.
4. **Reopen Flow**: Require `attendance.correct` permission to see the Reopen button. Prompt for a detailed reason before submitting the reopen request. Display the reason alongside the submitted timestamp.
5. **RowVersion Management**: Store `rowVersion` in component state. On every `PUT`, `POST /submit`, `POST /reopen`, or `POST /discard`, pass the latest `rowVersion`. On `409 Conflict`, re-fetch the sheet and show a toast notification.
6. **Optimistic UI**: On `PUT /attendance`, optimistically update the UI with the new statuses and show a success indicator. On failure, restore the previous state.
7. **Audit Trail Display**: Show the `recentChanges` array in a collapsible panel or side panel within the attendance detail view, so teachers can see who changed what and when.
8. **Session Lock Indicator**: If `status === 2` (Submitted), show a lock icon and disable editing. Show a banner: "This attendance was submitted on [date] by [teacher]. Contact an admin to reopen."
