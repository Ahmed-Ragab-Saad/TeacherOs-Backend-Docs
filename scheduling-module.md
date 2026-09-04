# Scheduling Module — Frontend Integration Guide

**Base URLs:**
- Recurring Schedules: `/api/tenants/{tenantId}/groups/{groupId}/schedules`
- Sessions: `/api/tenants/{tenantId}/sessions` and `/api/tenants/{tenantId}/groups/{groupId}/sessions`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Specific permissions: `schedules.view`/`schedules.manage` for recurring schedules; `sessions.view`/`sessions.manage`/`session.close` for sessions.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Recurring Schedules](#21-recurring-schedules)
   - 2.2 [Sessions — Tenant-Level](#22-sessions--tenant-level)
   - 2.3 [Sessions — Group-Level](#23-sessions--group-level)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Scheduling module manages two layers: **recurring schedules** (template definitions for recurring class times) and **sessions** (concrete occurrences that result from those templates or are created ad-hoc). Sessions are the primary unit of attendance recording and financial charging.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **RecurringSchedule** | A template defining a recurring weekly time slot for a group: day of week, local start time, duration in minutes, and an effective date range. Multiple active schedules can exist for the same group on different days. |
| **Session** | A concrete occurrence of a class session with a specific UTC time window, branch location, and (optionally) a teacher assignment. Sessions can be generated from recurring schedules or created ad-hoc ("one-off"). |
| **OccurrenceKey** | A string identifier (max 100 chars) linking a session back to its recurring schedule origin (e.g., `"2026-W09-Sun"`). Used to track session families and apply bulk operations. |
| **Manual Override** | When a session's time, teacher, or branch is changed from its recurring definition, `isManualOverride` is set to `true`. |
| **LocalStartTime** | Stored as `TimeOnly` in the recurring schedule. The frontend is responsible for converting to/from local time with appropriate timezone handling. |
| **Tenant-level Sessions** | Sessions are retrieved in aggregate at the tenant level for calendar views, filtered by date range and optional group/teacher/branch/status filters. |

### Recurring Schedule → Session Materialization

The backend provides a `ScheduleMaterializer` service that generates session instances from recurring schedules for a given date range. When a teacher views their calendar, the system materializes all sessions from all active recurring schedules within the requested window, plus any ad-hoc sessions created for the group.

### Session Lifecycle

```
 +------------------+    Reschedule / Substitute / Relocate
 |   Scheduled (1)  |<---------------------------------------+
 |                  |                                        |
 +--------+---------+                                        |
          |                                                  |
          | Complete                            Cancel        | Reopen
          v                                                  v
 +------------------+                               +------------------+
 |  Completed (2)   |                               |  Cancelled (3)   |
 +------------------+                               +------------------+
```

- **Scheduled → Completed**: Allowed via `POST /sessions/{sessionId}/complete` (requires `session.close` permission).
- **Scheduled → Cancelled**: Allowed via `POST /sessions/{sessionId}/cancel`.
- **Completed/Cancelled sessions are immutable**: No further reschedule, substitute, relocate, or re-cancel.
- **Attendance Lock**: Once attendance is submitted for a session, operational details (teacher, branch, time) are locked. The backend returns `409 Conflict` (`Sessions.LockedByAttendance`).

### Conflict Detection

When creating or rescheduling a session, the backend enforces two types of conflict prevention:
1. **Teacher Double Booking**: A teacher cannot have two sessions with overlapping time windows at the same time.
2. **Group Overlap**: A group cannot have two sessions with overlapping time windows.

---

## 2. API Endpoints Reference

### 2.1 Recurring Schedules

#### `GET /api/tenants/{tenantId}/groups/{groupId}/schedules`

Lists all recurring schedules for a specific group (all statuses).

**Authorization:** Permission `schedules.view`

**Response:** `200 OK`

```json
[
  {
    "id": "9d3f4e1a-2b5c-6d7e-8f90-a1b2c3d4e5f6",
    "groupId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "dayOfWeek": 0,
    "localStartTime": "08:00:00",
    "durationMinutes": 60,
    "effectiveFrom": "2026-09-01",
    "effectiveTo": "2026-12-31",
    "status": 1,
    "createdAtUtc": "2026-09-01T08:00:00Z",
    "updatedAtUtc": "2026-09-01T08:00:00Z"
  }
]
```

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/schedules`

Creates a recurring weekly schedule for a group.

**Authorization:** Permission `schedules.manage`

**Request Headers:** `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "dayOfWeek": 0,
  "localStartTime": "08:00:00",
  "durationMinutes": 60,
  "effectiveFrom": "2026-09-01",
  "effectiveTo": "2026-12-31"
}
```

**Field Details:**
- `dayOfWeek`: Integer from `0` (Sunday) to `6` (Saturday), matching `System.DayOfWeek`.
- `localStartTime`: `HH:mm:ss` format in the tenant's local timezone.
- `durationMinutes`: Integer between **15** and **480** (8 hours).
- `effectiveFrom`: Start date of the recurrence (inclusive), `YYYY-MM-DD`.
- `effectiveTo`: Optional end date (inclusive), `YYYY-MM-DD`. Must be >= `effectiveFrom` if provided.

**Response:** `201 Created`

**Location Header:** `/api/tenants/{tenantId}/groups/{groupId}/schedules/{scheduleId}`

**Possible Errors:**
- `400 Bad Request` — Invalid duration (outside 15–480 min range) or `effectiveTo` before `effectiveFrom`.
- `404 Not Found` — Group not found.
- `409 Conflict` — A schedule with overlapping time already exists for the group on this day/time.

---

#### `GET /api/tenants/{tenantId}/groups/{groupId}/schedules/{scheduleId}`

Retrieves a specific recurring schedule.

**Authorization:** Permission `schedules.view`

**Response:** `200 OK` — `RecurringScheduleResponse`

---

#### `PATCH /api/tenants/{tenantId}/groups/{groupId}/schedules/{scheduleId}`

Updates a recurring schedule's time, duration, or effective date range.

**Authorization:** Permission `schedules.manage`

**Request Headers:** `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "dayOfWeek": 3,
  "localStartTime": "14:30:00",
  "durationMinutes": 90,
  "effectiveFrom": "2026-09-01",
  "effectiveTo": "2027-03-31"
}
```

**Possible Errors:**
- `400 Bad Request` — Duration out of range.
- `404 Not Found` — Schedule not found.
- `409 Conflict` — Overlap with existing schedule after the change.

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/schedules/{scheduleId}/deactivate`

Deactivates a recurring schedule. Future materializations from this schedule stop; past sessions remain.

**Authorization:** Permission `schedules.manage`

**Response:** `200 OK` — `RecurringScheduleResponse` with `status: 2`

---

#### `POST /api/tenants/{tenantId}/groups/{groupId}/schedules/{scheduleId}/reactivate`

Reactivates an inactive recurring schedule.

**Authorization:** Permission `schedules.manage`

**Request Body:** Same as update.

**Response:** `200 OK` — `RecurringScheduleResponse` with `status: 1`

**Possible Errors:**
- `400 Bad Request` — Reactivating with the current time/day would cause immediate conflicts.
- `409 Conflict` — Would cause schedule overlap.

---

### 2.2 Sessions — Tenant-Level

#### `GET /api/tenants/{tenantId}/sessions`

Retrieves sessions across all groups for a tenant, filtered by a date range and optional criteria. This is the primary endpoint for calendar views.

**Authorization:** Permission `sessions.view`

**Query Parameters:**
- `from` (DateTime, required): Start of the date range (UTC ISO 8601).
- `to` (DateTime, required): End of the date range (UTC ISO 8601). Maximum span: **90 days**.
- `groupId` (Guid, optional): Filter by group.
- `teacherId` (Guid, optional): Filter by teacher.
- `branchId` (Guid, optional): Filter by branch.
- `status` (SessionStatus integer, optional): Filter by `1` (Scheduled), `2` (Completed), or `3` (Cancelled).

**Response:** `200 OK`

```json
[
  {
    "id": "7e4a1b2c-3d5e-6f7a-8b9c-0d1e2f3a4b5c",
    "tenantId": "5b6c7d8e-9f0a-1b2c-3d4e-5f6a7b8c9d0e1",
    "groupId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "groupName": "Grade 10 - Mathematics - Section A",
    "subjectId": "8f8b8941-6e3e-46a2-91eb-7a718c3c21a4",
    "subjectName": "Mathematics",
    "gradeLevelId": "f0e1d2c3-b4a5-6789-0abc-def123456789",
    "gradeLevelName": "Grade 10",
    "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13",
    "branchName": "Main Campus",
    "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
    "teacherFullName": "Dr. Sarah Connor",
    "recurringScheduleId": "9d3f4e1a-2b5c-6d7e-8f90-a1b2c3d4e5f6",
    "occurrenceKey": "2026-W36-Wed",
    "scheduledStartUtc": "2026-09-02T08:00:00Z",
    "scheduledEndUtc": "2026-09-02T09:00:00Z",
    "status": 1,
    "isManualOverride": false
  }
]
```

**Possible Errors:**
- `400 Bad Request` — Date range exceeds 90 days (`Sessions.RangeTooLarge`).

---

#### `GET /api/tenants/{tenantId}/sessions/{sessionId}`

Retrieves a single session by ID.

**Authorization:** Permission `sessions.view`

**Response:** `200 OK` — `SessionResponse`

**Possible Errors:**
- `404 Not Found` — Session not found.

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/reschedule`

Changes the scheduled start and end times for a session.

**Authorization:** Permission `sessions.manage`

**Request Headers:** `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "scheduledStartUtc": "2026-09-02T09:30:00Z",
  "scheduledEndUtc": "2026-09-02T10:30:00Z"
}
```

**Notes:**
- Sets `isManualOverride` to `true`.
- Validates no teacher double-booking or group overlap after the change.
- Fails if attendance has been submitted (`Sessions.LockedByAttendance`).

**Response:** `200 OK` — Updated `SessionResponse`

**Possible Errors:**
- `400 Bad Request` — End time must be after start time (`Sessions.InvalidInput`).
- `404 Not Found` — Session not found.
- `409 Conflict` — Teacher double booking or group overlap (`Sessions.TeacherDoubleBooking`, `Sessions.GroupOverlap`).
- `409 Conflict` — Session locked by attendance (`Sessions.LockedByAttendance`).
- `400 Bad Request` — Session is completed or cancelled.

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/teacher`

Substitutes the teacher for a session (e.g., cover teacher).

**Authorization:** Permission `sessions.manage`

**Request Headers:** `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "teacherProfileId": "c56a4180-65aa-42ec-a945-5fd21dec0538"
}
```

Pass `null` to unassign the teacher.

**Notes:**
- Sets `isManualOverride` to `true`.
- Teacher must be active and assigned to the group's subject (`Sessions.TeacherNotEligible`).

**Response:** `200 OK` — Updated `SessionResponse`

**Possible Errors:**
- `400 Bad Request` — Teacher is inactive or not eligible (`Sessions.TeacherNotEligible`).
- `409 Conflict` — Session locked by attendance (`Sessions.LockedByAttendance`).

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/branch`

Relocates the session to a different branch.

**Authorization:** Permission `sessions.manage`

**Request Body:**

```json
{
  "branchId": "b2c3d4e5-f6a7-8901-bcde-f23456789012"
}
```

**Response:** `200 OK` — Updated `SessionResponse`

**Possible Errors:**
- `404 Not Found` — Branch not found (`Sessions.BranchNotFound`).
- `409 Conflict` — Session locked by attendance (`Sessions.LockedByAttendance`).

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/cancel`

Cancels a scheduled session. Cancelled sessions cannot be completed or modified.

**Authorization:** Permission `sessions.manage`

**Response:** `200 OK` — `SessionResponse` with `status: 3`

**Possible Errors:**
- `400 Bad Request` — Session is already completed (`Sessions.CompletedImmutable`).

---

#### `POST /api/tenants/{tenantId}/sessions/{sessionId}/complete`

Marks a scheduled session as completed. This is typically called after attendance has been recorded.

**Authorization:** Permission `session.close`

**Response:** `200 OK` — `SessionResponse` with `status: 2`

**Possible Errors:**
- `400 Bad Request` — Session is already cancelled (`Sessions.CancelledImmutable`).

---

### 2.3 Sessions — Group-Level

#### `POST /api/tenants/{tenantId}/groups/{groupId}/sessions`

Creates a one-off (ad-hoc) session for a specific group, bypassing the recurring schedule.

**Authorization:** Permission `sessions.manage`

**Request Headers:** `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "scheduledStartUtc": "2026-09-10T14:00:00Z",
  "scheduledEndUtc": "2026-09-10T15:30:00Z",
  "teacherProfileId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "branchId": "0c9c34a2-1bf6-4c97-b248-cb5804368b13"
}
```

**Notes:**
- `teacherProfileId` and `branchId` are optional; if omitted, the group's default teacher/branch are used.
- `recurringScheduleId` and `occurrenceKey` are `null` for ad-hoc sessions.
- Validates no teacher double-booking or group overlap.

**Response:** `201 Created`

**Location Header:** `/api/tenants/{tenantId}/sessions/{sessionId}`

**Possible Errors:**
- `400 Bad Request` — End time not after start time, duration out of range, teacher not eligible.
- `404 Not Found` — Group not found (`Sessions.InactiveGroup` if group is archived).
- `409 Conflict` — Teacher double booking or group overlap.

---

## 3. Request & Response DTOs

### Request DTOs

#### `RecurringScheduleWriteRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `dayOfWeek` | `integer` (DayOfWeek) | Yes | `0` (Sunday) to `6` (Saturday). |
| `localStartTime` | `string` (TimeOnly) | Yes | `HH:mm:ss` format. |
| `durationMinutes` | `integer` | Yes | Between **15** and **480**. |
| `effectiveFrom` | `string` (DateOnly) | Yes | `YYYY-MM-DD`. |
| `effectiveTo` | `string?` (DateOnly) | No | `YYYY-MM-DD`. Must be >= `effectiveFrom`. |

#### `SessionCreateRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `scheduledStartUtc` | `string` (DateTime ISO 8601) | Yes | UTC. Must be before `scheduledEndUtc`. |
| `scheduledEndUtc` | `string` (DateTime ISO 8601) | Yes | UTC. Must be after `scheduledStartUtc`. |
| `teacherProfileId` | `Guid?` | No | Must be active and assigned to the group's subject. |
| `branchId` | `Guid?` | No | Must exist. Defaults to group's branch if omitted. |

#### `SessionRescheduleRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `scheduledStartUtc` | `string` | Yes | UTC. |
| `scheduledEndUtc` | `string` | Yes | UTC. Must be after start. |

#### `SessionTeacherRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `teacherProfileId` | `Guid?` | Yes | `null` to unassign; if set, must be active + eligible. |

#### `SessionBranchRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `branchId` | `Guid` | Yes | Must exist. |

### Response DTOs

#### `RecurringScheduleResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Schedule ID. |
| `groupId` | `Guid` | Associated group. |
| `dayOfWeek` | `integer` | `0`–`6`. |
| `localStartTime` | `string` | `HH:mm:ss` local time. |
| `durationMinutes` | `integer` | Duration in minutes. |
| `effectiveFrom` | `string` | `YYYY-MM-DD`. |
| `effectiveTo` | `string?` | `YYYY-MM-DD` or null. |
| `status` | `integer` | `1` = Active, `2` = Inactive. |
| `createdAtUtc` | `string` | ISO 8601 UTC. |
| `updatedAtUtc` | `string` | ISO 8601 UTC. |

#### `SessionResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Session ID. |
| `groupId` | `Guid` | Associated group. |
| `branchId` | `Guid` | Location branch. |
| `scheduledStartUtc` | `string` | UTC start time. |
| `scheduledEndUtc` | `string` | UTC end time. |
| `teacherProfileId` | `Guid?` | Assigned teacher (nullable). |
| `recurringScheduleId` | `Guid?` | Origin recurring schedule (null for ad-hoc). |
| `occurrenceKey` | `string?` | Human-readable occurrence identifier. |
| `status` | `integer` | `1` = Scheduled, `2` = Completed, `3` = Cancelled. |
| `isManualOverride` | `boolean` | `true` if time/teacher/branch was manually changed. |
| `createdAtUtc` | `string` | ISO 8601 UTC. |
| `updatedAtUtc` | `string` | ISO 8601 UTC. |

#### `CalendarSessionItem`

Same as `SessionResponse` plus enriched group/subject/teacher/branch names for calendar display.

| Additional Property | Type | Description |
|--------------------|------|-------------|
| `tenantId` | `Guid` | Tenant ID. |
| `groupName` | `string` | Group display name. |
| `subjectId` | `Guid` | Subject of the group. |
| `subjectName` | `string` | Subject display name. |
| `gradeLevelId` | `Guid` | Grade level. |
| `gradeLevelName` | `string` | Grade level display name. |
| `branchName` | `string` | Branch display name. |
| `teacherFullName` | `string?` | Teacher's full name. |

---

## 4. Enum Reference

### `RecurringScheduleStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Active` | Schedule is active; sessions will be materialized for its effective date range. |
| `2` | `Inactive` | Schedule is paused; no new sessions are materialized from it. |

### `SessionStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Scheduled` | Session is planned and can be modified. |
| `2` | `Completed` | Session has taken place; immutable. |
| `3` | `Cancelled` | Session was cancelled; immutable. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Duration Limits**: Session/schedule duration must be between **15** and **480 minutes** (8 hours).
2. **Time Window**: Session end time must be strictly after start time.
3. **Teacher Double Booking**: A teacher cannot have two sessions whose time windows overlap, even partially.
4. **Group Overlap**: A group cannot have two sessions with overlapping time windows.
5. **Teacher Eligibility**: Substitute teachers must be active and assigned to the group's subject. Null unassigns.
6. **Branch Existence**: Relocation targets must exist.
7. **Group Must Be Active**: Sessions cannot be created for archived groups.
8. **Session Immutability**: Completed and cancelled sessions cannot be rescheduled, substituted, relocated, cancelled, or completed again.
9. **Attendance Lock**: Once attendance is submitted for a session, operational fields (time, teacher, branch) cannot be changed (`Sessions.LockedByAttendance`).
10. **Calendar Range Limit**: Tenant-level session listings are capped at a **90-day** range.
11. **Recurring Schedule Date Range**: `effectiveTo` must be >= `effectiveFrom`.

### Frontend Validation Recommendations

- Enforce the 90-day maximum on the client side before calling `GET /sessions`.
- When rendering a teacher substitution dialog, filter available teachers by those assigned to the group's subject and active status.
- In calendar views, visually distinguish `isManualOverride: true` sessions (e.g., with an override icon) to show users that the session differs from the recurring template.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Sessions.NotFound` | 404 | Session does not exist. | Refresh calendar view. |
| `Sessions.InvalidInput` | 400 | End before start, invalid duration, or missing required fields. | Correct highlighted fields. |
| `Sessions.TeacherDoubleBooking` | 409 | Teacher is already booked during the requested time. | Choose a different time or teacher. |
| `Sessions.GroupOverlap` | 409 | Group already has a session during the requested time. | Choose a different time. |
| `Sessions.TeacherNotEligible` | 400 | Substitute teacher is inactive or not assigned to the subject. | Assign the teacher to the subject first. |
| `Sessions.BranchNotFound` | 404 | Relocation branch does not exist. | Refresh branch list. |
| `Sessions.CompletedImmutable` | 400 | Cannot modify or cancel a completed session. | No action possible. |
| `Sessions.CancelledImmutable` | 400 | Cannot complete or modify a cancelled session. | No action possible. |
| `Sessions.InactiveGroup` | 400 | Cannot create sessions for an archived group. | Reactivate the group first. |
| `Sessions.RangeTooLarge` | 400 | Calendar query exceeds 90-day limit. | Narrow the date range. |
| `Sessions.LockedByAttendance` | 409 | Session operational details locked because attendance was already submitted. | Reschedule at the attendance level or contact admin. |

---

## 7. Frontend Integration Best Practices

1. **Calendar View — Tenant-Level Endpoint**: Use `GET /api/tenants/{tenantId}/sessions?from=...&to=...` as the primary data source for teacher and admin calendar views. Apply filters (`groupId`, `teacherId`, `branchId`, `status`) to narrow results.
2. **Recurring Schedule Management**: Provide a dedicated "Schedule Templates" tab within the group detail view. Show all schedules for the group in a weekly timetable grid.
3. **Conflict Pre-Validation**: Before calling `POST /reschedule` or `POST /teacher`, the frontend should pre-check for overlap by calling the calendar endpoint with the new time window. However, the backend is authoritative.
4. **Timezone Handling**: `localStartTime` in recurring schedules is in the tenant's configured timezone. Always convert to UTC for display using the tenant's timezone. `scheduledStartUtc`/`scheduledEndUtc` are always UTC and should be displayed in the user's local time.
5. **Override Indicators**: Sessions with `isManualOverride: true` should be visually differentiated in calendar views (e.g., dashed border or override icon).
6. **Manual Override Propagation**: When editing a recurring schedule, the system does not automatically reschedule existing override sessions. The admin must handle those individually.
7. **Attendance Completion Flow**: After attendance is recorded and the session is completed, the `POST /complete` endpoint should be called to close the session. This is separate from the attendance submission workflow.
