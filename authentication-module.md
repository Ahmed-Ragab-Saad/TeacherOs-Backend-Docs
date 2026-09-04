# Authentication Module — Frontend Integration Guide

**Base URL:** `/api/auth`

**Role Requirement:** Mixed (some endpoints allow anonymous access, others require authentication)

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Antiforgery Token](#21-antiforgery-token)
   - 2.2 [Registration](#22-registration)
   - 2.3 [Login](#23-login)
   - 2.4 [Current Session](#24-current-session)
   - 2.5 [Logout](#25-logout)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Authentication module provides user registration, login, session management, and antiforgery protection for the TeacherOS platform. It implements a multi-tenant architecture where each registered user automatically creates their first tenant organization with full "Owner" permissions.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Cookie-Based Authentication** | Uses ASP.NET Core cookie authentication scheme (`TeacherOS.Application`). Sessions last 8 hours and are persistent across browser restarts. |
| **Antiforgery Protection** | All state-changing operations (POST, PATCH, DELETE) require both an antiforgery cookie (`__Host-TeacherOS.Antiforgery`) and the corresponding token in the `X-CSRF-TOKEN` header. |
| **Multi-Tenant Sessions** | A single user can be a member of multiple tenants. The current session includes all tenant memberships with their respective statuses. |
| **Tenant Context Selection** | After authentication, tenant-specific operations require the `X-Tenant-Id` header to select which tenant context to operate within. |
| **Rate Limiting** | Registration and login endpoints are rate-limited to prevent abuse. |

### Authentication Flow

1. **Registration**: User provides email, password, and tenant name. A new user account, tenant organization, "Owner" role, and active membership are created atomically in a transaction.
2. **Login**: User provides email and password. On success, a cookie-based session is established (8-hour expiration, persistent).
3. **Session Access**: The `/me` endpoint returns user info and all tenant memberships.
4. **Tenant Selection**: For tenant-scoped operations, the frontend sends the `X-Tenant-Id` header with each request.
5. **Logout**: Terminates the cookie-based session.

### Important Frontend Implications

- **Antiforgery tokens must be obtained before any POST/PATCH/DELETE request**. Call `GET /api/auth/antiforgery` first.
- **The antiforgery cookie is set automatically**; the frontend only needs to send the token value in the `X-CSRF-TOKEN` header.
- **After login, call `/api/auth/me`** to retrieve tenant memberships and allow the user to select a tenant.
- **Sessions expire after 8 hours** and do not auto-refresh. The frontend should handle 401 errors and redirect to login.
- **Password policies are enforced by ASP.NET Core Identity** but are not exposed through these endpoints. The backend will return validation errors if the password is too weak.

---

## 2. API Endpoints Reference

### 2.1 Antiforgery Token

#### `GET /api/auth/antiforgery`

Generates and returns an antiforgery token. This endpoint also sets the `__Host-TeacherOS.Antiforgery` cookie in the response.

**Authentication:** None (anonymous)

**Authorization:** None (anonymous)

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
{
  "token": "CfDJ8Gz...truncated"
}
```

**Response Headers:**
- `Cache-Control: no-store` — The token should not be cached.
- `Set-Cookie: __Host-TeacherOS.Antiforgery=...` — The antiforgery cookie.

**Important Notes:**
- Call this endpoint **before the first state-changing operation** (POST, PATCH, DELETE).
- The token is valid for the duration of the session.
- Store the token in memory (not localStorage) and include it in the `X-CSRF-TOKEN` header for subsequent mutations.
- If the cookie is lost or the token is rejected, call this endpoint again to refresh both.

---

### 2.2 Registration

#### `POST /api/auth/register`

Registers a new user, creates a new tenant organization, and establishes an active membership with the "Owner" role.

**Authentication:** None (anonymous)

**Authorization:** None (anonymous)

**Rate Limiting:** Applied (see [Error Handling](#6-error-handling-patterns) for 429 responses)

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token obtained from `/api/auth/antiforgery`

**Request Body:**

```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "tenantName": "My School"
}
```

**Response:** `201 Created`

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "user@example.com",
  "tenantId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

**Location Header:** `/api/auth/me`

**Possible Errors:**
- `400 Bad Request` — Validation error (missing/invalid email, password, or tenant name).
- `409 Conflict` — A user with the provided email already exists.
- `429 Too Many Requests` — Rate limit exceeded. Retry after the period specified in headers.

---

### 2.3 Login

#### `POST /api/auth/login`

Authenticates a user with email and password and establishes a cookie-based session.

**Authentication:** None (anonymous)

**Authorization:** None (anonymous)

**Rate Limiting:** Applied

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token

**Request Body:**

```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
```

**Response:** `200 OK`

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "user@example.com"
}
```

**Response Cookies:**
- `TeacherOS.Application` — The authentication cookie (HttpOnly, Secure, SameSite=Lax, 8-hour expiration, persistent).

**Possible Errors:**
- `400 Bad Request` — Email or password is missing.
- `401 Unauthorized` — Email or password is incorrect.
- `429 Too Many Requests` — Rate limit exceeded.

**Important Notes:**
- The session is **persistent** across browser restarts.
- The session **expires after 8 hours** and does not auto-refresh.
- After successful login, call `/api/auth/me` to retrieve tenant memberships.

---

### 2.4 Current Session

#### `GET /api/auth/me`

Retrieves the current authenticated user's session information, including all tenant memberships.

**Authentication:** Required (cookie-based)

**Authorization:** Any authenticated user

**Query Parameters:** None

**Request Body:** None

**Response:** `200 OK`

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "user@example.com",
  "selectedTenantId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "memberships": [
    {
      "tenantId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "tenantName": "My School",
      "tenantStatus": "Active",
      "membershipStatus": "Active"
    },
    {
      "tenantId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
      "tenantName": "Demo Tenant",
      "tenantStatus": "Trial",
      "membershipStatus": "Active"
    }
  ]
}
```

**Possible Errors:**
- `401 Unauthorized` — User is not authenticated or the session has expired.
- `403 Forbidden` — Session is authenticated but unavailable (rare edge case).

**Important Notes:**
- `selectedTenantId` is `null` if the `X-Tenant-Id` header was not sent in the request. It reflects the current tenant context if one was selected.
- The `memberships` array is ordered alphabetically by `tenantName`.
- `tenantStatus` and `membershipStatus` are serialized as strings (see [Enum Reference](#4-enum-reference)).

---

### 2.5 Logout

#### `POST /api/auth/logout`

Terminates the current authenticated session by clearing the authentication cookie.

**Authentication:** Required (cookie-based)

**Authorization:** Any authenticated user

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token

**Request Body:** None

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Antiforgery validation failed.
- `401 Unauthorized` — User is not authenticated.

**Important Notes:**
- After logout, the authentication cookie is cleared.
- Redirect the user to the login page after a successful logout.

---

## 3. Request & Response DTOs

### RegisterRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `email` | `string` | Yes | Must be a valid email address (max 256 characters). Validated using `System.Net.Mail.MailAddress`. |
| `password` | `string` | Yes | Non-empty string. ASP.NET Core Identity enforces additional password complexity rules (not exposed through API). |
| `tenantName` | `string` | Yes | Non-empty, non-whitespace string. Max 200 characters. Trimmed automatically. |

### RegisterResponse

| Field | Type | Description |
|-------|------|-------------|
| `userId` | `Guid` | The unique identifier of the newly created user. |
| `email` | `string` | The registered email address. |
| `tenantId` | `Guid` | The unique identifier of the newly created tenant organization. |

### LoginRequest

| Field | Type | Required | Constraints & Validation |
|-------|------|----------|--------------------------|
| `email` | `string` | Yes | Non-empty, non-whitespace string. Trimmed automatically. |
| `password` | `string` | Yes | Non-empty string. |

### LoginResponse

| Field | Type | Description |
|-------|------|-------------|
| `userId` | `Guid` | The unique identifier of the authenticated user. |
| `email` | `string` | The user's email address. |

### CurrentSessionResponse

| Field | Type | Description |
|-------|------|-------------|
| `userId` | `Guid` | The unique identifier of the authenticated user. |
| `email` | `string` | The user's email address. |
| `selectedTenantId` | `Guid?` | The currently selected tenant ID from the `X-Tenant-Id` header, or `null` if no tenant context is active. |
| `memberships` | `TenantMembershipResponse[]` | Array of all tenant memberships for the user, ordered alphabetically by `tenantName`. |

### TenantMembershipResponse

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | `Guid` | The unique identifier of the tenant. |
| `tenantName` | `string` | The display name of the tenant organization. |
| `tenantStatus` | `string` | The tenant's status (see [TenantStatus](#tenantstatus)). |
| `membershipStatus` | `string` | The user's membership status within this tenant (see [TenantMembershipStatus](#tenantmembershipstatus)). |

### AntiforgeryTokenResponse

| Field | Type | Description |
|-------|------|-------------|
| `token` | `string` | The antiforgery request token to be sent in the `X-CSRF-TOKEN` header. |

---

## 4. Enum Reference

### TenantStatus

Enum values are serialized as **strings** (e.g., `"Active"`, `"Trial"`).

| Value | Description |
|-------|-------------|
| `Trial` | Tenant is in trial mode (value: 1). |
| `Active` | Tenant is active and operational (value: 2). |
| `Suspended` | Tenant is temporarily suspended (value: 3). |
| `Expired` | Tenant's subscription or trial has expired (value: 4). |
| `Closed` | Tenant account is permanently closed (value: 5). |

### TenantMembershipStatus

Enum values are serialized as **strings** (e.g., `"Active"`, `"Suspended"`).

| Value | Description |
|-------|-------------|
| `Active` | User's membership in the tenant is active (value: 1). |
| `Suspended` | User's membership in the tenant is suspended (value: 2). |

---

## 5. Validation & Business Rules

### Email Validation

**Backend-enforced:**
- Must be a valid email format according to `System.Net.Mail.MailAddress`.
- Maximum length: 256 characters.
- Trimmed automatically.
- Case-insensitive comparison for duplicate detection.

**Error returned:** `Authentication.InvalidEmail` (400 Bad Request)

### Password Validation

**Backend-enforced:**
- Must not be empty.
- Additional password complexity rules are enforced by ASP.NET Core Identity (minimum length, required character types, etc.), but these are not explicitly documented in the API layer.

**Error returned:** `Authentication.PasswordRequired` (400 Bad Request) if missing, or Identity validation errors if password is too weak (error messages vary).

### Tenant Name Validation

**Backend-enforced:**
- Must not be empty or whitespace.
- Maximum length: 200 characters.
- Trimmed automatically.

**Error returned:**
- `Authentication.TenantNameRequired` (400 Bad Request)
- `Authentication.TenantNameTooLong` (400 Bad Request)

### Registration Business Rules

- **Atomicity:** User registration, tenant creation, "Owner" role creation, and membership assignment happen in a single database transaction. If any part fails, the entire operation is rolled back.
- **Duplicate email:** If a user with the provided email already exists, the registration fails with a 409 Conflict error.
- **Owner role:** The newly created user is automatically assigned the "Owner" role within the new tenant, granting all permissions.
- **Tenant status:** The new tenant is created with `TenantStatus.Active`.
- **Membership status:** The new membership is created with `TenantMembershipStatus.Active`.

### Login Business Rules

- **Credential validation:** Both email and password must be provided and correct.
- **Trimming:** Email is trimmed before authentication.
- **Rate limiting:** Excessive failed login attempts trigger rate limiting (429 response).

### Session and Tenant Context

- **Session duration:** 8 hours from the login time. Sessions do not auto-refresh.
- **Tenant context selection:** The `X-Tenant-Id` header is required for tenant-scoped operations. The middleware validates that:
  - The user is authenticated.
  - The tenant ID is a valid, non-empty GUID.
  - The user has an **active** membership in the specified tenant (both `tenantStatus` and `membershipStatus` are considered).
- **Membership filtering:** Only active memberships are considered when validating tenant access. Suspended memberships are rejected with a 403 Forbidden error.

### Antiforgery Validation

- **Required for mutations:** All POST, PATCH, and DELETE endpoints require antiforgery protection.
- **Token + cookie:** The antiforgery mechanism requires both the `__Host-TeacherOS.Antiforgery` cookie and the `X-CSRF-TOKEN` header to match.
- **Failure behavior:** If validation fails, the endpoint returns 400 Bad Request with code `Antiforgery.ValidationFailed`.

---

## 6. Error Handling Patterns

### Error Response Format

All errors follow the RFC 7807 Problem Details standard:

```json
{
  "type": "about:blank",
  "title": "The request is invalid.",
  "status": 400,
  "detail": "Email and password are required.",
  "code": "Authentication.CredentialsRequired"
}
```

| Field | Description |
|-------|-------------|
| `type` | Always `"about:blank"` for application errors. |
| `title` | Human-readable summary of the error category (based on status code). |
| `status` | HTTP status code. |
| `detail` | Specific error message describing what went wrong. |
| `code` | Stable error code for programmatic handling (see below). |

### Authentication Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `Authentication.CredentialsRequired` | 400 | Email or password is missing. |
| `Authentication.InvalidEmail` | 400 | Email format is invalid or exceeds 256 characters. |
| `Authentication.PasswordRequired` | 400 | Password is missing. |
| `Authentication.TenantNameRequired` | 400 | Tenant name is missing or whitespace. |
| `Authentication.TenantNameTooLong` | 400 | Tenant name exceeds 200 characters. |
| `Authentication.InvalidCredentials` | 401 | Email or password is incorrect. |
| `Authentication.SessionUnavailable` | 401 | The authenticated session is unavailable or expired. |
| `Authentication.DuplicateEmail` | 409 | A user with the provided email already exists. |
| `Antiforgery.ValidationFailed` | 400 | Antiforgery token validation failed. |
| `Tenancy.InvalidSelector` | 400 | The `X-Tenant-Id` header is missing, malformed, or contains multiple values. |
| `Tenancy.AccessDenied` | 403 | The user is not an active member of the selected tenant. |

### Status Code Summary

| Status | When It Occurs |
|--------|----------------|
| `200 OK` | Successful login or session retrieval. |
| `201 Created` | Successful registration. |
| `204 No Content` | Successful logout. |
| `400 Bad Request` | Validation error (missing fields, invalid format, antiforgery failure). |
| `401 Unauthorized` | Invalid credentials or expired/missing session. |
| `403 Forbidden` | User does not have access to the selected tenant. |
| `409 Conflict` | Duplicate email during registration. |
| `429 Too Many Requests` | Rate limit exceeded. |

### Rate Limiting

Registration and login endpoints are protected by rate limiting. When the limit is exceeded, the backend returns:

```json
{
  "type": "about:blank",
  "title": "Too many requests.",
  "status": 429,
  "detail": "Too many requests. Please wait before trying again."
}
```

**Frontend handling:**
- Parse the `Retry-After` header (if present) to determine when to retry.
- Display a user-friendly message and disable the form temporarily.

---

## 7. Frontend Integration Best Practices

### Initial Setup

1. **Obtain an antiforgery token on app load:**
   - Call `GET /api/auth/antiforgery` when the app initializes (or when the user navigates to a login/registration page).
   - Store the token in memory (e.g., in a React context or Vuex store).
   - Do NOT store the token in localStorage or sessionStorage (security risk).

2. **Include the token in all mutation requests:**
   - Add the `X-CSRF-TOKEN` header to all POST, PATCH, and DELETE requests.
   - If a request fails with `Antiforgery.ValidationFailed`, re-fetch the token and retry.

### Registration Flow

1. **Validate inputs on the frontend:**
   - Check email format (RFC 5322).
   - Ensure password meets minimum complexity (e.g., 8+ characters, at least one uppercase, one digit).
   - Ensure tenant name is non-empty and ≤ 200 characters.

2. **Submit registration request:**
   - Include the `X-CSRF-TOKEN` header.
   - Handle errors:
     - `400`: Display validation messages.
     - `409`: Email already exists—prompt the user to log in or use a different email.
     - `429`: Show rate limit message and disable the form temporarily.

3. **On success:**
   - The user account, tenant, and membership are created.
   - **Important:** Registration does NOT automatically log the user in. Redirect to the login page or immediately call the login endpoint with the same credentials.

### Login Flow

1. **Validate inputs on the frontend:**
   - Ensure email and password are non-empty.

2. **Submit login request:**
   - Include the `X-CSRF-TOKEN` header.
   - Handle errors:
     - `400`/`401`: Display "Invalid email or password" message.
     - `429`: Show rate limit message.

3. **On success:**
   - The authentication cookie is set automatically.
   - Immediately call `GET /api/auth/me` to retrieve tenant memberships.

4. **Tenant selection:**
   - If the user has multiple tenant memberships, present a selection UI.
   - Store the selected tenant ID in app state.
   - Include the `X-Tenant-Id` header in all subsequent tenant-scoped requests.

### Session Management

1. **Check authentication status on app load:**
   - Call `GET /api/auth/me` to verify the session is still valid.
   - If it returns `401`, redirect to the login page.

2. **Handle session expiration:**
   - Sessions expire after 8 hours without auto-refresh.
   - Set up a global error handler to catch `401 Unauthorized` responses and redirect to login.
   - Optionally, warn the user before expiration (e.g., show a modal at 7.5 hours).

3. **Tenant context:**
   - Always send the `X-Tenant-Id` header for tenant-scoped operations.
   - If the backend returns `403 Forbidden` with `Tenancy.AccessDenied`, the user no longer has access to the selected tenant (e.g., membership was suspended). Prompt the user to select a different tenant or show an access denied message.

### Logout Flow

1. **Submit logout request:**
   - Include the `X-CSRF-TOKEN` header.

2. **On success:**
   - Clear all client-side state (user info, tenant selection, tokens).
   - Redirect to the login page.

### Error Handling

- **Display user-friendly messages:**
  - Map error codes to localized messages (e.g., `Authentication.InvalidCredentials` → "The email or password is incorrect. Please try again.").
- **Retry logic:**
  - Do NOT automatically retry `401` or `403` errors (these require user action).
  - For transient errors (network failures), implement exponential backoff.
- **Rate limiting:**
  - Respect the `Retry-After` header.
  - Show a countdown timer or disable the form until the retry period elapses.

### Security Considerations

- **Never log or expose passwords** in frontend code, logs, or error messages.
- **Store antiforgery tokens in memory only**, not in localStorage or cookies.
- **Use HTTPS in production** to protect the authentication cookie and antiforgery token.
- **Implement CSRF protection** by always including the `X-CSRF-TOKEN` header for mutations.
- **Handle session expiration gracefully** to prevent data loss (e.g., save draft state before redirecting to login).

### Multi-Tenant Considerations

- **Tenant switching:**
  - Allow users to switch between tenants without logging out.
  - Update the `X-Tenant-Id` header and re-fetch tenant-specific data.
- **Membership status awareness:**
  - Check `membershipStatus` in the `/me` response. Only tenants with `Active` membership are accessible.
  - If a tenant's `tenantStatus` is `Suspended`, `Expired`, or `Closed`, inform the user and prevent operations.
- **Default tenant selection:**
  - If the user has only one active membership, auto-select it.
  - If multiple memberships exist, prompt the user to choose.

### Performance Optimization

- **Cache the antiforgery token:**
  - Fetch it once on app load and reuse it for the session.
  - Only re-fetch if a mutation returns `Antiforgery.ValidationFailed`.
- **Debounce login/registration requests:**
  - Prevent accidental double-submissions by disabling the submit button after the first click.
- **Lazy-load tenant-specific data:**
  - Do not fetch tenant-scoped data until a tenant is selected.

---

## Summary

The Authentication module provides cookie-based authentication with antiforgery protection, multi-tenant session management, and rate limiting. Key integration points:

- **Antiforgery tokens** are required for all mutations.
- **Registration** creates a user, tenant, and "Owner" role atomically but does NOT log the user in.
- **Login** establishes an 8-hour session (no auto-refresh).
- **Session retrieval** (`/me`) returns all tenant memberships.
- **Tenant context** is selected via the `X-Tenant-Id` header for tenant-scoped operations.
- **Error handling** follows RFC 7807 Problem Details with stable error codes.

Frontend developers should implement antiforgery protection, handle session expiration, and provide tenant selection UI for multi-tenant users.
