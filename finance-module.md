# Finance Module — Frontend Integration Guide

**Base URLs:**
- Payments: `POST /api/tenants/{tenantId}/payments`
- Charge Management: `POST /api/tenants/{tenantId}/session-charges/{chargeId}/mark-paid`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permissions: `payment.record` for recording payments, `payment.adjust` for marking charges paid manually.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Record Payment](#21-record-payment)
   - 2.2 [Mark Session Charge Paid](#22-mark-session-charge-paid)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Finance module handles billing for sessions and cash collection from students/guardians. It is built around two core entities: **SessionCharges** (issued automatically when attendance is recorded) and **Payments** (recorded manually by reception/front-desk staff). Payments allocate amounts to one or more SessionCharges.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **SessionCharge** | A billing line item created per student per session attended. Amount is a snapshot of the group's price at charge time. |
| **Payment** | Cash-in event recorded by a receptionist. Contains the total amount received and optional allocations to specific charges. |
| **PaymentAllocation** | Links a payment to a specific session charge, representing partial or full settlement of that charge. |
| **Idempotency** | Payment recording is idempotent via a `Idempotency-Key` header. Retrying the same key returns the original result without creating duplicates. |
| **Currency** | All charges and allocations within a single payment must use the same currency. Currency codes are normalized to uppercase 3-letter ISO 4217 format (e.g., `"USD"`, `"EUR"`). |
| **Charge Status** | `Pending → PartiallyPaid → Paid`. Charges can also be marked Paid manually via the `mark-paid` endpoint. |
| **Amount Snapshot** | `SessionCharge.Amount` is immutable. It records the group's price at the moment of attendance submission. Group price changes after that do not affect existing charges. |

### Data Flow

```
Session Attendance Submitted
        |
        v
RecordSessionChargeHandler
  (creates one SessionCharge per student)
        |
        v
SessionCharge (status: Pending, amount snapshot)
        |
        v
RecordPaymentHandler
  (creates Payment + PaymentAllocations)
        |
        v
PaymentAllocation links Payment → SessionCharge
  (allocates part or all of payment amount)
        |
        v
SessionCharge status updated:
  - PartiallyPaid if total allocations < Amount
  - Paid if total allocations >= Amount
```

### Important Frontend Implications

- **Session Charges Are Automatic**: Session charges are created automatically when attendance is submitted. The frontend does not need to create charges explicitly.
- **Idempotency Is Critical for Payments**: Always generate a unique `Idempotency-Key` per user-initiated payment attempt. Network retries must send the same key.
- **Currency Consistency**: A payment cannot mix currencies. All allocations must use the same currency as the payment itself.
- **Payment Allocations**: A single payment can settle multiple charges (e.g., paying for three months at once). Each allocation targets one charge.
- **No Charge Over-Allocation**: The total allocated to a charge across all payments cannot exceed the charge's original amount.
- **CSRF + Rate Limiting**: Payment recording is protected by both antiforgery tokens and per-user rate limiting to prevent abuse.

---

## 2. API Endpoints Reference

### 2.1 Record Payment

#### `POST /api/tenants/{tenantId}/payments`

Records a cash, bank transfer, card, or other payment from a student/guardian and allocates it to one or more outstanding session charges.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `payment.record`

**Rate Limit:** Per-user rate limit policy (`PaymentRecordRateLimitPolicy`). Returns `429 Too Many Requests` if exceeded.

**Request Headers:**
- `X-CSRF-TOKEN` (required): Antiforgery token.
- `X-Tenant-Id` (required): Active tenant identifier.
- `Idempotency-Key` (required): A unique key for this payment attempt. Max 64 characters. Retrying with the same key returns the original result without duplication.
- `X-Idempotency-Fingerprint` (required): SHA-like fingerprint of the request body. Used to detect when the same key is reused with a different body (which is rejected as a conflict).

**Request Body:**

```json
{
  "receivedAtUtc": "2026-09-04T10:30:00Z",
  "method": 1,
  "currency": "USD",
  "allocations": [
    {
      "sessionChargeId": "c1a2b3d4-e5f6-7890-a1b2-c3d4e5f60789",
      "amount": 150.00
    },
    {
      "sessionChargeId": "d2e3f4a5-b6c7-8901-b2c3-d4e5f6078901",
      "amount": 150.00
    }
  ],
  "reference": "TRF-2026-0904-001",
  "notes": "Monthly fee for September 2026"
}
```

**Response:** `201 Created` (or `200 OK` on idempotent replay — same response body)

**Location Header:** `/api/tenants/{tenantId}/payments/{paymentId}`

```json
{
  "paymentId": "f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "amount": 300.00,
  "currency": "USD",
  "method": "Cash",
  "receivedAtUtc": "2026-09-04T10:30:00Z",
  "receivedByMembershipId": "2d1a3e67-8573-455b-b9f1-a1e56b46e300",
  "reference": "TRF-2026-0904-001",
  "notes": "Monthly fee for September 2026",
  "allocations": [
    {
      "allocationId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
      "sessionChargeId": "c1a2b3d4-e5f6-7890-a1b2-c3d4e5f60789",
      "amount": 150.00,
      "allocatedAtUtc": "2026-09-04T10:30:00Z"
    },
    {
      "allocationId": "b2c3d4e5-f6a7-8901-b2c3-d4e5f6078901",
      "sessionChargeId": "d2e3f4a5-b6c7-8901-b2c3-d4e5f6078901",
      "amount": 150.00,
      "allocatedAtUtc": "2026-09-04T10:30:00Z"
    }
  ]
}
```

**Possible Errors:**
- `400 Bad Request` — Missing or invalid input (`Finance.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `payment.record` permission or tenant mismatch.
- `404 Not Found` — Referenced session charge not found (`Finance.SessionNotFound`).
- `409 Conflict` — Fingerprint mismatch on duplicate key (`Finance.IdempotencyKeyFingerprintMismatch`), duplicate allocation (`Finance.DuplicatePaymentAllocation`), or key in concurrent use (`Finance.IdempotencyKeyInUse`).
- `422 UnprocessableEntity` — Currency mismatch (`Finance.CurrencyMismatch`) or over-allocation (`Finance.ChargeOverAllocated`).
- `429 Too Many Requests` — Rate limit exceeded.

---

### 2.2 Mark Session Charge Paid

#### `POST /api/tenants/{tenantId}/session-charges/{chargeId}/mark-paid`

Manually marks a session charge as fully paid. Used for offline payments, scholarships, or waivers that bypass the standard payment recording flow. Idempotent: re-calling on an already-Paid charge is a no-op.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `payment.adjust`

**Rate Limit:** Per-user rate limit policy (`PaymentAdjustRateLimitPolicy`).

**Request Headers:**
- `X-CSRF-TOKEN` (required)
- `X-Tenant-Id` (required)

**Path Parameters:**
- `tenantId` (Guid, required)
- `chargeId` (Guid, required)

**Request Body:**

```json
{
  "reason": "Scholarship waiver approved by director"
}
```

**Response:** `204 No Content`

**Possible Errors:**
- `400 Bad Request` — Empty or missing reason (`Finance.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `payment.adjust` permission or tenant mismatch.
- `404 Not Found` — Charge not found (`Finance.SessionChargeNotFound`).
- `429 Too Many Requests` — Rate limit exceeded.

---

## 3. Request & Response DTOs

### Request DTOs

#### `RecordPaymentRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `receivedAtUtc` | `DateTime` | Yes | Must not be default/empty. UTC timestamp of when payment was received. |
| `method` | `PaymentMethod` integer | Yes | `1`=Cash, `2`=BankTransfer, `3`=Card, `99`=Other. |
| `currency` | `string` | Yes | 3-letter uppercase ISO 4217 code (e.g., `"USD"`, `"EUR"`). |
| `allocations` | `Array<PaymentAllocationInput>` | Yes | At least one allocation. Each targets one session charge. |
| `reference` | `string?` | No | Max 200 characters. External reference (e.g., bank transaction ID). |
| `notes` | `string?` | No | Max 500 characters. Internal notes. |

#### `PaymentAllocationInput`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `sessionChargeId` | `Guid` | Yes | The charge to allocate payment toward. |
| `amount` | `decimal` | Yes | Must be strictly positive. |

#### `MarkSessionChargePaidRequest`

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `reason` | `string` | Yes | Non-empty reason (max 500 chars). Stored in the audit trail. |

### Response DTOs

#### `PaymentResponse`

| Property | Type | Description |
|----------|------|-------------|
| `paymentId` | `Guid` | Unique payment identifier. |
| `amount` | `decimal` | Total payment amount (sum of allocations). |
| `currency` | `string` | Currency code (e.g., "USD"). |
| `method` | `string` | Payment method name (e.g., "Cash"). |
| `receivedAtUtc` | `DateTime` | When the payment was received. |
| `receivedByMembershipId` | `Guid` | Membership ID of the receptionist who recorded it. |
| `reference` | `string?` | External reference if provided. |
| `notes` | `string?` | Internal notes if provided. |
| `allocations` | `Array<PaymentAllocationSummaryResponse>` | List of charge allocations. |

#### `PaymentAllocationSummaryResponse`

| Property | Type | Description |
|----------|------|-------------|
| `allocationId` | `Guid` | Allocation identifier. |
| `sessionChargeId` | `Guid` | The targeted charge. |
| `amount` | `decimal` | Amount allocated to this charge. |
| `allocatedAtUtc` | `DateTime` | UTC timestamp of the allocation. |

---

## 4. Enum Reference

### `PaymentMethod`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Cash` | Physical cash payment at the reception desk. |
| `2` | `BankTransfer` | Wire transfer or bank deposit. |
| `3` | `Card` | Credit or debit card (POS terminal). |
| `99` | `Other` | Any other payment method. |

### `SessionChargeStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Pending` | Charge created, not yet fully settled by payments. |
| `2` | `PartiallyPaid` | At least one allocation received, but total < charge amount. |
| `3` | `Paid` | Fully settled. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **Idempotency Key Required**: `Idempotency-Key` and `X-Idempotency-Fingerprint` headers are mandatory on all payment requests.
2. **Key + Fingerprint Pairing**: Using the same `Idempotency-Key` with a different body (different fingerprint) returns `409 Conflict` (`Finance.IdempotencyKeyFingerprintMismatch`).
3. **At Least One Allocation**: Payments must allocate to at least one charge. Zero-allocation payments are rejected.
4. **Currency Consistency**: All allocations must reference charges in the same currency as the payment itself.
5. **No Duplicate Allocations**: A single payment cannot allocate to the same charge twice.
6. **No Over-Allocation**: The sum of all allocations to a charge across all payments cannot exceed the charge's original amount.
7. **Positive Amounts**: Payment amount and each allocation amount must be strictly positive (> 0).
8. **Valid Currency Code**: Currency must be exactly 3 uppercase letters.
9. **Reason Required for Manual Mark-Paid**: The `reason` field cannot be empty on the `mark-paid` endpoint.
10. **Reason Length**: Reason is limited to 500 characters (same as audit event reason limit).
11. **Rate Limiting**: Both payment endpoints are rate-limited. Exceeding the limit returns `429 Too Many Requests`.

### Frontend Validation Recommendations

- Always generate a new UUID v4 for `Idempotency-Key` on each user-initiated payment form submission.
- Compute `X-Idempotency-Fingerprint` as a stable hash (e.g., SHA-256) of the normalized JSON body — not the raw string, to normalize whitespace/key ordering.
- Sum allocation amounts and confirm they match the expected total before submitting.
- Validate currency matches across all selected charges before submitting.
- On `409 Conflict` from an idempotent key retry, silently accept the replayed response — do not show an error to the user.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Finance.NotAuthenticated` | 401 | Caller is not authenticated. | Redirect to login. |
| `Finance.TenantMismatch` | 403 | Tenant context does not match user access. | Verify tenant selection. |
| `Finance.InvalidInput` | 400 | Missing required fields, invalid format, or empty reason. | Highlight invalid fields. |
| `Finance.SessionNotFound` | 404 | Referenced session charge does not exist. | Refresh the charges list. |
| `Finance.SessionChargeNotFound` | 404 | Charge to mark paid does not exist. | Refresh the charges list. |
| `Finance.GroupHasNoPrice` | 422 | Charge creation failed because group has no price. | Configure group price in Groups module. |
| `Finance.DuplicateSessionCharge` | 409 | A charge already exists for this student at this session. | Charge already auto-created; no action needed. |
| `Finance.IdempotencyKeyFingerprintMismatch` | 409 | Same key used with different request body. | Generate a new idempotency key for the corrected request. |
| `Finance.DuplicatePaymentAllocation` | 409 | Same charge allocated twice in one payment. | Remove duplicate allocation from request. |
| `Finance.CurrencyMismatch` | 422 | Payment currency differs from one or more target charge currencies. | Select charges in the same currency, or change payment currency. |
| `Finance.ChargeOverAllocated` | 422 | Allocation would exceed the charge's remaining balance. | Reduce allocation amount or select different charges. |
| `Finance.IdempotencyKeyInUse` | 409 | Key is currently being processed by another request. | Retry after a short delay. |
| `Tenancy.AccessDenied` | 403 | Tenant access denied. | Verify X-Tenant-Id header. |

---

## 7. Frontend Integration Best Practices

1. **Payment Recording Form**: Build a multi-step form: (1) Select charges to pay, (2) Enter payment amount/method/reference, (3) Confirm and submit. Validate currency consistency in step 1.
2. **Outstanding Charges Dashboard**: Display a list of `Pending` and `PartiallyPaid` charges per student with remaining balance. Allow multi-select for batch payment allocation.
3. **Idempotency Implementation**: Use a UUID v4 for `Idempotency-Key` generated at form submission time. Store it in session storage or a hidden field so retries reuse the same key.
4. **Fingerprint Computation**: Compute the fingerprint as `SHA256(sorted-json-stringify(body))` for stable ordering across JSON serializers.
5. **Optimistic UI**: After submitting, show a loading spinner and wait for the response. On `201 Created`, show success. On `200 OK` (idempotent replay), show "Payment already recorded" without error.
6. **Rate Limit Handling**: If `429` is received, show "Please wait a moment and try again" and implement exponential backoff.
7. **Auto-Refresh Charges**: After a successful payment, re-fetch the student's charge list to reflect updated `Paid`/`PartiallyPaid` status.
8. **Mark-Paid for Waivers**: Provide a dedicated UI for admin users to mark charges as paid with a reason (scholarship, director waiver, etc.) using the `mark-paid` endpoint.
9. **Audit Trail**: Payment operations are recorded in the Audit module. Link to the audit event from the payment confirmation screen.
10. **Currency Display**: Always show currency codes alongside amounts (e.g., "150.00 USD") and validate that all charges being paid share the same currency.
