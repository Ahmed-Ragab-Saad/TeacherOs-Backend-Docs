# Webhooks Module — Frontend Integration Guide

**Base URL:** `/api/webhooks/brevo/email-delivery`

**Authentication:** Brevo Bearer Token (configured server-side). No cookie-based auth or tenant context required.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Email Delivery Webhook](#21-email-delivery-webhook)
3. [Payload Reference](#3-payload-reference)
4. [Enum Reference](#4-enum-reference)
5. [Error Handling Patterns](#5-error-handling-patterns)
6. [Integration Notes](#6-integration-notes)

---

## 1. Architecture & Design

The Webhooks module receives and processes delivery status callbacks from Brevo (formerly Sendinblue), a transactional email provider. When TeacherOS sends an email through its outbox system, Brevo delivers it to the recipient and then notifies TeacherOS of the final delivery outcome (delivered, bounced, blocked, etc.) via this webhook endpoint.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Email Outbox** | TeacherOS uses an outbox pattern to send emails. Each sent email has a GUID (`outboxMessageId`) stored as the `X-Mailin-custom` header, allowing the webhook to correlate incoming delivery events back to the original outbox record. |
| **Brevo** | The transactional email service provider. Webhooks arrive from `brevo.com`. |
| **Brevo Custom Header** | The `X-Mailin-custom` header contains `"teacheros-outbox-v1:{guid}"`, where the GUID is the outbox message ID used for correlation. |
| **Bearer Token Auth** | The endpoint uses a shared secret bearer token (configured server-side in `EmailOptions.BrevoWebhookBearerToken`) to authenticate incoming webhook requests. |
| **Idempotent Processing** | The endpoint handles malformed, duplicate, and unsupported event types gracefully — returning `204 No Content` without error for events that don't require processing. |
| **Retry on Failure** | If processing fails, the endpoint returns `429 Too Many Requests` with a `Retry-After: 60` header, instructing Brevo to retry delivery. |

### Webhook Event Flow

```
TeacherOS sends email via Brevo API
  (includes X-Mailin-custom: "teacheros-outbox-v1:{guid}")
         |
         v
Brevo delivers email to recipient
         |
         v
Brevo POSTs delivery event to
  /api/webhooks/brevo/email-delivery
  (Bearer token in Authorization header)
         |
         v
BrevoEmailDeliveryWebhookParser.TryParse()
  - Validates payload structure
  - Extracts X-Mailin-custom → outboxMessageId
  - Maps Brevo event type → EmailDeliveryStatus
         |
         v
IEmailDeliveryEventProcessor.ProcessAsync()
  - Updates email outbox record with delivery status
  - Stores reason code/description for bounces/failures
```

### Payload Correlation

The `X-Mailin-custom` header in the original email contains the correlation key:
```
"teacheros-outbox-v1:{outboxMessageId}"
```

This allows the webhook processor to map any incoming delivery event back to the specific outbox message record, regardless of the Brevo `message-id`.

### Important Frontend Implications

- **No Frontend Integration Required**: The webhook is server-to-server communication between Brevo and TeacherOS. No browser or SPA is involved.
- **Brevo Configuration**: The webhook URL (`https://your-domain.com/api/webhooks/brevo/email-delivery`) and bearer token must be configured in the Brevo webhook settings panel.
- **Payload Limit**: Payloads exceeding 16 KB are rejected with `413 Payload Too Large`.
- **JSON Only**: Only `application/json` content types are accepted. Other media types return `415 Unsupported Media Type`.
- **Duplicate Events**: Brevo may deliver the same event multiple times. The processor should be idempotent when updating the outbox record.

---

## 2. API Endpoints Reference

### 2.1 Email Delivery Webhook

#### `POST /api/webhooks/brevo/email-delivery`

Receives email delivery status callbacks from Brevo and updates the corresponding email outbox records.

**Authentication:** Bearer token (configured in `EmailOptions.BrevoWebhookBearerToken`). The token is compared using constant-time comparison to prevent timing attacks.

**Request Headers:**
- `Authorization` (required): `Bearer {configured_secret}`. Rejected with `401 Unauthorized` if missing or incorrect.
- `Content-Type` (required): `application/json`. Rejected with `415 UnsupportedMediaType` if other types.
- `Content-Length` (optional): If present and exceeds 16 KB, rejected with `413 PayloadTooLarge`.

**Request Body (Brevo sends this on each email event):**

```json
{
  "event": "delivered",
  "id": 1234567890,
  "ts_event": 1725456000,
  "message-id": "<abc123@example.com>",
  "email": "student@parent.com",
  "X-Mailin-custom": "teacheros-outbox-v1:f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "reason": null,
  "error_code": null
}
```

**Response:** `204 No Content` — Event processed successfully or intentionally skipped.

**Possible Error Responses:**
- `400 Bad Request` — Malformed JSON or missing required fields.
- `401 Unauthorized` — Missing or invalid bearer token.
- `413 Payload Too Large` — Payload exceeds 16 KB.
- `415 Unsupported Media Type` — Content-Type is not `application/json`.
- `429 Too Many Requests` — Processing failed; Brevo should retry.

---

## 3. Payload Reference

### Required Fields

| Field | Type | Description | Validation |
|-------|------|-------------|------------|
| `event` | `string` | Event type from Brevo (e.g., `"delivered"`, `"hard_bounce"`). | Required. Max 50 chars. Mapped to `EmailDeliveryStatus`. |
| `id` | `integer` | Brevo's internal webhook event ID. | Required. Positive integer. |
| `ts_event` | `integer` | Unix timestamp (seconds) when the event occurred. | Required. Positive. Converted to UTC. |
| `message-id` | `string` | Brevo's unique message identifier for the email. | Required. Max 200 chars. |
| `email` | `string` | Recipient email address. | Required. Max 256 chars. |
| `X-Mailin-custom` | `string` | Correlation header. Must start with `"teacheros-outbox-v1:"` followed by a valid GUID. | Required. Max 64 chars. |

### Optional Fields

| Field | Type | Description | Validation |
|-------|------|-------------|------------|
| `reason` | `string?` | Human-readable reason for bounce/deferral. | Optional. Max 500 chars if present. |
| `error_code` | `string?` | Numeric/code reason for failure. | Optional. Max 100 chars. Can be string or number. |

### Supported Event Types

| Brevo Event | EmailDeliveryStatus | Notes |
|-------------|--------------------|-------|
| `request` / `sent` | `ProviderAccepted` | Brevo accepted the email. |
| `delivered` | `Delivered` | Email delivered to recipient's mail server. |
| `deferred` | `Deferred` | Temporary delay; Brevo will retry. |
| `soft_bounce` | `SoftBounce` | Soft bounce (mailbox full, etc.). |
| `hard_bounce` | `HardBounce` | Permanent delivery failure. |
| `blocked` | `Blocked` | Brevo blocked the message. |
| `invalid_email` / `error` | `Failed` | Invalid recipient or other error. |
| *(any other value)* | — | Returns `204 No Content` (unsupported event). |

---

## 4. Enum Reference

### `EmailDeliveryStatus`

The final delivery status of an email outbox message.

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Pending` | Email is queued and awaiting dispatch. |
| `2` | `Processing` | Email is being processed by the provider. |
| `3` | `ProviderAccepted` | Brevo accepted the email for delivery. |
| `4` | `Failed` | Delivery failed permanently (invalid email or error). |
| `5` | `Delivered` | Email was successfully delivered to the recipient. |
| `6` | `Deferred` | Delivery was temporarily delayed. |
| `7` | `SoftBounce` | Soft bounce — recipient's mailbox may be full or server temporary issue. |
| `8` | `HardBounce` | Hard bounce — recipient address is permanently invalid. |
| `9` | `Blocked` | Brevo blocked the message (spam, policy violation, etc.). |

---

## 5. Error Handling Patterns

### HTTP Status Codes

| Status | Meaning | Brevo Behavior |
|--------|---------|---------------|
| `204 No Content` | Successfully processed or intentionally skipped. | No retry. |
| `400 Bad Request` | Malformed or missing required fields. | Brevo may retry. |
| `401 Unauthorized` | Invalid bearer token. | Brevo should not retry — fix the token. |
| `413 Payload Too Large` | Payload exceeds 16 KB. | Brevo may retry. |
| `415 Unsupported Media Type` | Non-JSON content type. | Brevo may retry. |
| `429 Too Many Requests` | Processing failed (DB unavailable, etc.). Brevo is instructed to retry after 60 seconds. | Retry after 60 seconds. |

### Duplicate Events

Brevo may send the same event multiple times. The `IEmailDeliveryEventProcessor` implementation should be idempotent — updating the outbox record to the same status multiple times is safe and has no side effects.

### Malformed Payloads

- Missing required fields → `400 Bad Request`
- `event` is null/empty or unknown → `204 No Content` (silently ignored)
- Invalid JSON → `400 Bad Request`
- `X-Mailin-custom` missing or not matching the `teacheros-outbox-v1:` prefix → `400 Bad Request`
- Invalid GUID in `X-Mailin-custom` → `400 Bad Request`

---

## 6. Integration Notes

### Setting Up the Webhook in Brevo

1. Log in to Brevo and navigate to **Transactionals** → **Settings** → **Webhooks**.
2. Create a new webhook for **Email** events.
3. Set the URL to: `https://your-teacheros-domain.com/api/webhooks/brevo/email-delivery`
4. Set the authorization method to **Bearer Token** and enter the configured secret.
5. Select the events you want to subscribe to: `sent`, `delivered`, `deferred`, `soft_bounce`, `hard_bounce`, `blocked`, `error`.
6. Save and test with a single send.

### Testing Locally

Use `curl` to simulate a Brevo webhook call:

```bash
curl -X POST https://localhost:5001/api/webhooks/brevo/email-delivery \
  -H "Authorization: Bearer your-configured-secret" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "delivered",
    "id": 1234567890,
    "ts_event": 1725456000,
    "message-id": "<test@example.com>",
    "email": "student@parent.com",
    "X-Mailin-custom": "teacheros-outbox-v1:f1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"
  }'
```

Expected response: `204 No Content`

### Security Considerations

- The bearer token must be stored securely in server configuration, not in client-accessible code.
- Token comparison uses `CryptographicOperations.FixedTimeEquals` to prevent timing attacks.
- The endpoint is disabled for antiforgery (no CSRF token required) since it uses bearer authentication.
- All payloads are validated before processing to prevent injection attacks.
