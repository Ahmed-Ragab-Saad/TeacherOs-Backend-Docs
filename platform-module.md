# Platform Module — Frontend Integration Guide

**Base URL:** `/api/platform`

**Role Requirement:** Platform-level admin access required. Not tenant-scoped. Requires `RequirePlatformAdmin()` authorization.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [List All Tenants](#21-list-all-tenants)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Error Handling Patterns](#5-error-handling-patterns)
6. [Frontend Integration Best Practices](#6-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Platform module provides a cross-tenant, super-admin view of the entire TeacherOS installation. It is scoped at the `/api/platform` path (not `/api/tenants/{tenantId}`) and is accessible only to platform administrators — not regular tenant users.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Platform Admin** | A user with platform-level access who can see and manage all tenants in the system. |
| **Tenant** | An independent organization/branch managed within the platform. Each tenant has its own data, users, and configuration. |
| **TenantStatus** | Lifecycle state of a tenant: Trial, Active, Suspended, Expired, Closed. |
| **TimeZoneId** | IANA/Olson time zone identifier (e.g., `"America/New_York"`, `"Europe/Cairo"`). Used to display local times within each tenant. |
| **Tenant Scoping** | All other modules in TeacherOS are scoped to a specific tenant via the `X-Tenant-Id` header. The Platform module is the only exception — it operates at the platform level. |

### Important Frontend Implications

- **No Tenant Context**: The Platform module does not require `X-Tenant-Id`. It operates on all tenants simultaneously.
- **Platform Admin Only**: Only users with platform-level admin privileges can access these endpoints.
- **Status Display**: Use `TenantSummaryResponse.Status` to render status badges or filter the tenant list in the admin dashboard.

---

## 2. API Endpoints Reference

### 2.1 List All Tenants

#### `GET /api/platform/tenants`

Retrieves a summary list of all tenants in the platform. Used by platform administrators to manage the full tenant roster, monitor trial expirations, or suspend/close accounts.

**Authentication:** Required (Cookie-based)

**Authorization:** Platform Admin role (not tenant-scoped).

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
[
  {
    "id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
    "name": "Springfield Learning Center",
    "status": "Active"
  },
  {
    "id": "d67b5291-7bb1-43f3-b156-6ae32de06449",
    "name": "Downtown Tutoring Hub",
    "status": "Trial"
  },
  {
    "id": "e78c6302-8cc2-54g4-c267-7bf43ef1755a",
    "name": "Westside Academy",
    "status": "Suspended"
  }
]
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — User is not a platform administrator.

---

## 3. Request & Response DTOs

### Response DTOs

#### `TenantSummaryResponse`

| Property | Type | Description |
|----------|------|-------------|
| `id` | `Guid` | Unique identifier of the tenant. |
| `name` | `string` | Display name of the tenant. Max 200 characters. |
| `status` | `string` | Tenant status as a string (e.g., `"Active"`, `"Trial"`). |

---

## 4. Enum Reference

### `TenantStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Trial` | Tenant is on a free trial period. |
| `2` | `Active` | Tenant is fully operational and subscribed. |
| `3` | `Suspended` | Tenant is temporarily suspended (e.g., for non-payment). |
| `4` | `Expired` | Trial or subscription has expired. |
| `5` | `Closed` | Tenant account has been permanently closed. |

---

## 5. Error Handling Patterns

### Common Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Authentication.Unauthorized` | 401 | Not authenticated. | Redirect to login. |
| `Platform.AccessDenied` | 403 | User is not a platform administrator. | Contact a super-admin. |

---

## 6. Frontend Integration Best Practices

1. **Tenant Management Dashboard**: Build a table/card view of all tenants with columns: Name, Status, Time Zone. Allow filtering by status (Active, Trial, Suspended, Expired, Closed).
2. **Status Badges**: Color-code tenant status (e.g., green=Active, blue=Trial, yellow=Suspended, red=Expired/Closed).
3. **Platform-Level Navigation**: Platform admin views should not carry tenant context. Design a separate admin shell with platform-level navigation (tenant management, platform settings, etc.).
4. **Trial Expiry Alerts**: Use the `Trial` and `Expired` statuses to highlight tenants needing attention in the admin dashboard.
5. **Time Zone Display**: Show the `TimeZoneId` (e.g., `"America/New_York"`) alongside tenant details so admins know which timezone the tenant operates in.
6. **Tenant Linking**: Each tenant row/card should link to the tenant's dashboard (switching context to that tenant) or open a detail panel with management actions.
