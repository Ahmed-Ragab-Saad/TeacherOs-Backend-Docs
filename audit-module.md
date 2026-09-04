# Audit Module — Frontend Integration Guide

**Base URL:** `/api/tenants/{tenantId}/audit-events`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Requires permission `audit.view` for all read operations.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List Audit Events](#21-list-audit-events)
   - 2.2 [Get Audit Event by ID](#22-get-audit-event-by-id)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Validation & Business Rules](#4-validation--business-rules)
5. [Search, Sorting & Pagination](#5-search-sorting--pagination)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Audit module provides a read-only view into the immutable event log of tenant-scoped actions. It captures every significant domain event (create, update, delete, submit, reopen, etc.) across all modules, storing the actor, action, affected entity, and optional before/after state snapshots.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **AuditEvent** | Immutable record of a domain action. Never modified or deleted. |
| **Actor** | The membership/user who performed the action (`ActorMembershipId`). |
| **Action** | The type of action performed (e.g., `"Created"`, `"Updated"`, `"Submitted"`, `"Reopened"`). |
| **EntityType** | The type of entity affected (e.g., `"AttendanceSheet"`, `"Group"`, `"TeacherProfile"`). |
| **EntityId** | The GUID of the specific entity instance. |
| **BeforeState** | Optional JSON snapshot of the entity state before the action. |
| **AfterState** | Optional JSON snapshot of the entity state after the action. |
| **CorrelationId** | Optional request-tracing identifier linking related events. |
| **Tenant Scoping** | All queries are scoped to the tenant in the route. Cross-tenant access is denied. |

### What Events Are Recorded

The audit log captures events from multiple modules. Common action patterns include:

- **Attendance**: `Created`, `Updated`, `Submitted`, `Reopened`, `Discarded`
- **Finance**: `ChargeCreated`, `PaymentRecorded`, `SessionCharged`, `ChargeAdjusted`, `PaymentReversed`
- **Scheduling**: `SessionScheduled`, `SessionRescheduled`, `SessionCancelled`
- **Groups**: `Created`, `Updated`, `Deactivated`, `StudentEnrolled`, `StudentWithdrawn`
- **Memberships**: `Activated`, `Deactivated`, `RoleAssigned`

### Important Frontend Implications

- **Read-Only**: The Audit module provides no write operations. Audit records are immutable.
- **State Snapshots**: `BeforeState` and `AfterState` are stored as JSON strings. Parse them on the frontend to render diffs or state history.
- **Filtering**: Use `entityType` and `action` query parameters to narrow results (e.g., show only `AttendanceSheet` events, or only `"Submitted"` actions).
- **Actor Filtering**: Filter by `actorMembershipId` to see all actions taken by a specific user.
- **Date Range**: Always apply `fromUtc` and `toUtc` filters when loading the audit log for performance. The default page size is 50.
- **CorrelationId**: When recording sessions (scheduling module), events that belong to the same composite operation may share a `CorrelationId`.

---

## 2. API Endpoints Reference

### 2.1 List Audit Events

#### `GET /api/tenants/{tenantId}/audit-events`

Retrieves a paginated list of audit events with optional filters.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `audit.view`

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.

**Query Parameters:**
- `entityType` (string, optional): Filter by entity type (e.g., `"AttendanceSheet"`, `"Group"`). Case-insensitive substring match.
- `action` (string, optional): Filter by action name (e.g., `"Submitted"`, `"Created"`). Case-insensitive substring match.
- `actorMembershipId` (Guid, optional): Filter events by the user who performed the action.
- `fromUtc` (DateTime, optional): Start of the time range (UTC ISO 8601). Inclusive.
- `toUtc` (DateTime, optional): End of the time range (UTC ISO 8601). Inclusive.
- `page` (integer, optional, default `1`): 1-based page index (minimum: 1).
- `pageSize` (integer, optional, default `50`): Items per page (range: 1–100).

**Request Body:** None

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
      "actorMembershipId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
      "action": "Submitted",
      "entityType": "AttendanceSheet",
      "entityId": "7e4a1b2c-3d5e-6f7a-8b9c-0d1e2f3a4b5c",
      "reason": "Daily attendance completed",
      "correlationId": null,
      "occurredAtUtc": "2026-09-04T09:30:00Z"
    },
    {
      "id": "a2b3c4d5-e6f7-8901-b2c3-d4e5f6078901",
      "actorMembershipId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
      "action": "Created",
      "entityType": "Group",
      "entityId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "reason": "New class section for Grade 11",
      "correlationId": "req-2026-0904-001",
      "occurredAtUtc": "2026-09-04T08:00:00Z"
    }
  ],
  "totalCount": 2,
  "page": 1,
  "pageSize": 50
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `audit.view` permission or tenant access denied (`Tenancy.AccessDenied`).

---

### 2.2 Get Audit Event by ID

#### `GET /api/tenants/{tenantId}/audit-events/{auditEventId}`

Retrieves full details of a single audit event, including the `BeforeState` and `AfterState` JSON snapshots.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `audit.view`

**Path Parameters:**
- `tenantId` (Guid, required): Target tenant ID.
- `auditEventId` (Guid, required): Audit event ID.

**Request Body:** None

**Response:** `200 OK`

```json
{
  "id": "f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "tenantId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "actorMembershipId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "action": "Submitted",
  "entityType": "AttendanceSheet",
  "entityId": "7e4a1b2c-3d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "reason": "Daily attendance completed",
  "beforeState": "{\"status\":1,\"revisionNumber\":1}",
  "afterState": "{\"status\":2,\"revisionNumber\":1,\"submittedAtUtc\":\"2026-09-04T09:30:00Z\"}",
  "correlationId": null,
  "occurredAtUtc": "2026-09-04T09:30:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated.
- `403 Forbidden` — Missing `audit.view` permission or tenant access denied.
- `404 Not Found` — Audit event not found (`Audit.EventNotFound`).

---

## 3. Request & Response DTOs

### Response DTOs

#### `AuditEventSummaryResponse`

Returned in the list endpoint (without state snapshots).

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Unique identifier of the audit event. |
| `actorMembershipId` | `Guid` | The user who performed the action. |
| `action` | `string` | Action name (e.g., "Created", "Submitted"). Max 100 chars. |
| `entityType` | `string` | Type of entity affected. Max 100 chars. |
| `entityId` | `Guid` | Identifier of the specific entity instance. |
| `reason` | `string` | Context/reason for the action. Max 500 chars. |
| `correlationId` | `string?` | Optional request-tracing identifier. |
| `occurredAtUtc` | `DateTime` | UTC timestamp when the event occurred. |

#### `AuditEventDetailsResponse`

Returned by the single-event endpoint (includes state snapshots).

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Audit event ID. |
| `tenantId` | `Guid` | Tenant this event belongs to. |
| `actorMembershipId` | `Guid` | User who performed the action. |
| `action` | `string` | Action name. |
| `entityType` | `string` | Entity type. |
| `entityId` | `Guid` | Entity instance ID. |
| `reason` | `string` | Context/reason. |
| `beforeState` | `string?` | JSON snapshot of entity before action. |
| `afterState` | `string?` | JSON snapshot of entity after action. |
| `correlationId` | `string?` | Correlation ID. |
| `occurredAtUtc` | `DateTime` | UTC timestamp. |

---

## 4. Validation & Business Rules

### Backend-Enforced Rules

1. **Tenant Scoping**: All events are scoped to the tenant in the route. The `X-Tenant-Id` header must match.
2. **Pagination Bounds**: `page` defaults to `1`. `pageSize` defaults to `50`, clamps to `1`–`100`.
3. **Optional Filters**: `entityType`, `action`, `actorMembershipId`, `fromUtc`, and `toUtc` are all optional. When omitted, no filter is applied.
4. **No Delete/Modify**: Audit events are immutable. No update or delete endpoints exist.

### Frontend Validation Recommendations

- Always parse `beforeState` and `afterState` with `JSON.parse()` before rendering.
- Show a loading indicator while fetching audit events, especially for large date ranges.
- Debounce search/filter inputs to avoid excessive API calls.

---

## 5. Search, Sorting & Pagination

- **Search Filters**: Use `entityType` and `action` to narrow results. Both support case-insensitive substring matching.
- **Actor Filter**: `?actorMembershipId={guid}` returns only events performed by that user.
- **Date Range**: `?fromUtc=...&toUtc=...` filters by timestamp. Always apply a reasonable default range (e.g., last 30 days) when loading the audit log.
- **Ordering**: Results are ordered by `OccurredAtUtc` descending (most recent first).
- **Pagination**: Default `page=1`, `pageSize=50` (max: 100).

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Audit.EventNotFound` | 404 | Audit event does not exist or belongs to another tenant. | Verify the event ID. |
| `Tenancy.AccessDenied` | 403 | Tenant context does not match user access. | Verify `X-Tenant-Id` selection. |
| `Authentication.Unauthorized` | 401 | User is not authenticated. | Redirect to login. |

---

## 7. Frontend Integration Best Practices

1. **Audit Log Dashboard**: Build a filterable table view with columns: Timestamp, Actor, Action, Entity Type, Entity ID, Reason. Allow sorting by timestamp (default descending).
2. **Action Badges**: Color-code actions by type (e.g., green for "Created"/"Submitted", yellow for "Updated", red for "Deleted").
3. **Actor Resolution**: Display `actorMembershipId` by resolving it to a user name/email from the Members module.
4. **State Diff View**: When viewing event details, parse `beforeState` and `afterState` and render a visual diff (e.g., side-by-side or inline highlighting of changed fields).
5. **CorrelationId**: If `correlationId` is present, provide a way to find all related events sharing the same ID — useful for tracing multi-step operations.
6. **Entity Links**: Make `entityId` clickable — link back to the relevant module's detail page (e.g., clicking an `AttendanceSheet` entity ID opens the attendance detail view).
7. **Auto-Refresh**: If displaying live audit activity, poll the endpoint every 30–60 seconds or use long-polling for real-time updates.
8. **Date Range Picker**: Provide a date range picker (defaulting to the last 7 days) to filter events before making the request.
