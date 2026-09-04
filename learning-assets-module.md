# Learning Assets Module — Frontend Integration Guide

**Base URLs:**
- Learning Assets: `/api/tenants/{tenantId}/learning/assets`
- Resources: `/api/tenants/{tenantId}/learning/resources`

**Role Requirement:** Tenant context required (`X-Tenant-Id` header). Permissions: `learning.resources.manage` for create/update, `learning.resources.publish` for publishing.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [API Endpoints Reference](#2-api-endpoints-reference)
   - 2.1 [Initiate Asset Upload](#21-initiate-asset-upload)
   - 2.2 [Finalize Asset Upload](#22-finalize-asset-upload)
   - 2.3 [Create External Link Asset](#23-create-external-link-asset)
   - 2.4 [Create Resource Placement](#24-create-resource-placement)
   - 2.5 [Update Resource Details](#25-update-resource-details)
   - 2.6 [Publish Resource](#26-publish-resource)
   - 2.7 [Authorize Asset Access](#27-authorize-asset-access)
3. [Request & Response DTOs](#3-request--response-dtos)
4. [Enum Reference](#4-enum-reference)
5. [Validation & Business Rules](#5-validation--business-rules)
6. [Error Handling Patterns](#6-error-handling-patterns)
7. [Frontend Integration Best Practices](#7-frontend-integration-best-practices)

---

## 1. Architecture & Design

The Learning Assets module manages media files (videos, PDFs, documents) and external links used within courses. It follows a two-phase upload pattern: initiate an upload to get a pre-signed URL, upload directly to storage, then finalize to confirm completion. Resources are the contextual wrappers that place assets into courses, sections, or lessons.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **LearningAsset** | The actual file or external link. Has a `Kind` (Video, Pdf, File, ExternalLink) and `ProcessingStatus`. |
| **Resource** | The contextual placement of an asset within the learning hierarchy. Has a `PlacementType` (Course, Section, Lesson). |
| **Upload Flow** | Client-side upload via pre-signed URLs — files never pass through the API server. |
| **External Link** | URLs to third-party content (YouTube, external PDFs). No upload needed. |
| **ResourceStatus** | Draft resources are editable; Published resources are visible to students. |
| **Delivery Policy** | Controls how students can access the asset (Stream, View, Download, External). |

### Upload Flow

```
Frontend                          API Server                    Storage Provider
    |                                 |                               |
    |--- InitiateUpload ------------->|                               |
    |<-- { StorageKey, UploadUrl,    |<-- (generates signed URL) -----|
    |     FormFields, ExpiresAt } ---|                               |
    |                                 |                               |
    |--- PUT to UploadUrl ----------->|                               |
    |     (direct to storage)        |                               |
    |                                 |<-- (file lands in storage) ---|
    |                                 |                               |
    |--- FinalizeUpload ------------->|                               |
    |                                 |--- Trigger processing ------->|
    |<-- 202 Accepted ---------------|                               |
    |     (async processing)        |                               |
```

### Resource Placement Hierarchy

```
Course/Section/Lesson
        |
        v
Resource (links to LearningAsset)
        |
        v
LearningAsset (the actual file or external link)
```

### Important Frontend Implications

- **Direct-to-Storage Upload**: The API provides a pre-signed URL; the frontend uploads the file directly to the cloud storage. No streaming through your server.
- **Async Processing**: After upload finalization, video processing is asynchronous. Poll `GET /resources/{id}` to check status.
- **External Links**: External links skip the upload flow entirely — create the asset and then create the resource directly.
- **Resource Lifecycle**: Resources have their own status (Draft → Published) separate from the underlying asset.
- **Access Authorization**: Students must be authorized to access protected resources. Use the authorize endpoint before displaying content.

---

## 2. API Endpoints Reference

### 2.1 Initiate Asset Upload

#### `POST /api/tenants/{tenantId}/learning/assets/initiate-upload`

Initiates a file upload. Returns a pre-signed URL and form fields for direct upload to cloud storage.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "fileName": "lesson-video.mp4",
  "contentType": "video/mp4",
  "fileSizeBytes": 52428800,
  "kind": 1
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fileName` | `string` | Yes | Original file name with extension. Max 255 characters. |
| `contentType` | `string` | Yes | MIME type (e.g., `"video/mp4"`, `"application/pdf"`). |
| `fileSizeBytes` | `long` | Yes | File size in bytes. Must be within configured limits. |
| `kind` | `LearningAssetKind` | Yes | `1`=Video, `2`=Pdf, `3`=File. |

**Response:** `200 OK`

```json
{
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "storageKey": "assets/tenant-123/course-456/a1b2c3d4.mp4",
  "uploadUrl": "https://storage.example.com/signed-upload-url...",
  "expiresAtUtc": "2026-09-04T12:00:00Z",
  "formFieldsJson": "{\"key\":\"assets/...\",\"acl\":\"private\",\"Content-Type\":\"video/mp4\"}"
}
```

| Property | Type | Description |
|----------|------|-------------|
| `assetId` | `Guid` | The newly created asset ID. Use this when finalizing. |
| `storageKey` | `string` | The path where the file will be stored. |
| `uploadUrl` | `string` | Pre-signed URL for uploading the file. |
| `expiresAtUtc` | `DateTime` | When the signed URL expires. Upload must complete by then. |
| `formFieldsJson` | `string` | JSON of additional form fields required by the storage provider. |

**Possible Errors:**
- `400 Bad Request` — Validation error (`Asset.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.manage` permission or tenant access denied.
- `409 Conflict` — Storage provider not configured (`Asset.StorageProviderNotConfigured`).

---

### 2.2 Finalize Asset Upload

#### `POST /api/tenants/{tenantId}/learning/assets/finalize-upload`

Confirms that a file upload has completed. Triggers async processing for video transcoding.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789"
}
```

**Response:** `202 Accepted`

```json
{
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "status": "Processing",
  "message": "Upload confirmed. Video processing has started."
}
```

| Status | Meaning |
|--------|---------|
| `Ready` | File is ready immediately (PDFs, other files). |
| `Processing` | Video transcoding in progress. Poll for completion. |

**Possible Errors:**
- `400 Bad Request` — Validation error.
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.manage` permission or tenant access denied.
- `404 Not Found` — Asset not found (`Asset.AssetNotFound`).
- `409 Conflict` — Asset already finalized or in wrong state.

---

### 2.3 Create External Link Asset

#### `POST /api/tenants/{tenantId}/learning/assets/external-link`

Creates an asset representing an external URL (YouTube video, external document, etc.).

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Introduction to Algebra - YouTube",
  "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
  "kind": 4
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `string` | Yes | Display title for the link. Max 200 characters. |
| `url` | `string` | Yes | Valid HTTPS URL. Max 2000 characters. |
| `kind` | `LearningAssetKind` | Yes | Must be `4` (ExternalLink). |

**Response:** `201 Created`

```json
{
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "title": "Introduction to Algebra - YouTube",
  "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
  "kind": "ExternalLink",
  "status": "Ready"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error (`Asset.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.manage` permission or tenant access denied.

---

### 2.4 Create Resource Placement

#### `POST /api/tenants/{tenantId}/learning/resources`

Creates a resource that places an asset into a course, section, or lesson context.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "placementType": 3,
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Supplementary Reading",
  "description": "Additional reference material for this lesson",
  "deliveryPolicy": 2
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assetId` | `Guid` | Yes | The asset to place. |
| `placementType` | `ResourcePlacementType` | Yes | `1`=Course, `2`=Section, `3`=Lesson. |
| `courseId` | `Guid` | Yes | The course context. |
| `sectionId` | `Guid?` | No | Required if `placementType` is Section or Lesson. |
| `lessonId` | `Guid?` | No | Required if `placementType` is Lesson. |
| `title` | `string` | Yes | Resource display title. Max 200 characters. |
| `description` | `string?` | No | Resource description. Max 1000 characters. |
| `deliveryPolicy` | `ResourceDeliveryPolicy` | No | Defaults to `1` (StreamOnly). |

**Response:** `201 Created`

```json
{
  "resourceId": "r1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "placementType": "Lesson",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Supplementary Reading",
  "description": "Additional reference material for this lesson",
  "deliveryPolicy": "ViewOnly",
  "status": "Draft",
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `400 Bad Request` — Validation error (`Asset.InvalidInput`).
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.manage` permission or tenant access denied.
- `404 Not Found` — Asset not found (`Asset.AssetNotFound`).
- `409 Conflict` — Invalid placement (`Asset.InvalidPlacement`) or incompatible delivery policy (`Asset.IncompatibleDeliveryPolicy`).

---

### 2.5 Update Resource Details

#### `PUT /api/tenants/{tenantId}/learning/resources/{resourceId}`

Updates a draft resource's metadata. Only allowed for Draft resources.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "title": "Updated Resource Title",
  "description": "Updated description",
  "deliveryPolicy": 3
}
```

All fields are optional.

**Response:** `200 OK`

```json
{
  "resourceId": "r1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "placementType": "Lesson",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Updated Resource Title",
  "description": "Updated description",
  "deliveryPolicy": "DownloadAllowed",
  "status": "Draft",
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.manage` permission or tenant access denied.
- `404 Not Found` — Resource not found (`Asset.ResourceNotFound`).
- `409 Conflict` — Resource is not in Draft status.

---

### 2.6 Publish Resource

#### `POST /api/tenants/{tenantId}/learning/resources/{resourceId}/publish`

Publishes a draft resource, making it visible to students.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.publish`

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:** None

**Response:** `200 OK`

```json
{
  "resourceId": "r1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "assetId": "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f60789",
  "placementType": "Lesson",
  "courseId": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "sectionId": "s1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "lessonId": "l1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789",
  "title": "Supplementary Reading",
  "description": "Additional reference material for this lesson",
  "deliveryPolicy": "ViewOnly",
  "status": "Published",
  "createdAtUtc": "2026-09-04T10:00:00Z"
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing `learning.resources.publish` permission or tenant access denied.
- `404 Not Found` — Resource not found (`Asset.ResourceNotFound`).
- `409 Conflict` — Resource is not in Draft status or asset is not Ready.

---

### 2.7 Authorize Asset Access

#### `POST /api/tenants/{tenantId}/learning/resources/{resourceId}/authorize-access`

Checks if a student has access to a resource and returns the appropriate access URL/token.

**Authentication:** Required (Cookie-based)

**Authorization:** Permission `learning.resources.manage` (teachers) or student context

**Request Headers:**
- `X-CSRF-TOKEN` (required)

**Request Body:**

```json
{
  "studentId": "st1a2b3c4-d5e6-7890-a1b2-c3d4e5f60789"
}
```

**Response:** `200 OK`

```json
{
  "authorized": true,
  "accessUrl": "https://cdn.teacheros.com/stream/abc123...",
  "expiresAtUtc": "2026-09-04T13:00:00Z",
  "policy": "StreamOnly"
}
```

| Property | Type | Description |
|----------|------|-------------|
| `authorized` | `bool` | Whether access is granted. |
| `accessUrl` | `string?` | Pre-signed access URL (only if authorized). |
| `expiresAtUtc` | `DateTime?` | When the access URL expires. |
| `policy` | `string?` | The delivery policy applied. |

**Response (Not Authorized):** `200 OK`

```json
{
  "authorized": false,
  "accessUrl": null,
  "expiresAtUtc": null,
  "policy": null
}
```

**Possible Errors:**
- `401 Unauthorized` — Not authenticated.
- `403 Forbidden` — Missing permission or tenant access denied.
- `404 Not Found` — Resource not found (`Asset.ResourceNotFound`).

---

## 3. Request & Response DTOs

### Request DTOs

#### `InitiateAssetUploadRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `FileName` | `string` | Yes | 1–255 chars, valid extension. |
| `ContentType` | `string` | Yes | Valid MIME type. |
| `FileSizeBytes` | `long` | Yes | Must be within limits. |
| `Kind` | `LearningAssetKind` | Yes | Enum value. |

#### `FinalizeAssetUploadRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `AssetId` | `Guid` | Yes | Asset from initiate response. |

#### `CreateExternalLinkAssetRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `Title` | `string` | Yes | 1–200 chars. |
| `Url` | `string` | Yes | Valid HTTPS URL, 1–2000 chars. |
| `Kind` | `LearningAssetKind` | Yes | Must be `4` (ExternalLink). |

#### `CreateResourceRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `AssetId` | `Guid` | Yes | Valid asset reference. |
| `PlacementType` | `ResourcePlacementType` | Yes | Enum value. |
| `CourseId` | `Guid` | Yes | Valid course reference. |
| `SectionId` | `Guid?` | Conditional | Required for Section/Lesson placement. |
| `LessonId` | `Guid?` | Conditional | Required for Lesson placement. |
| `Title` | `string` | Yes | 1–200 chars. |
| `Description` | `string?` | No | Max 1000 chars. |
| `DeliveryPolicy` | `ResourceDeliveryPolicy` | No | Defaults to `StreamOnly`. |

#### `UpdateResourceRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `Title` | `string?` | No | 1–200 chars. |
| `Description` | `string?` | No | Max 1000 chars. |
| `DeliveryPolicy` | `ResourceDeliveryPolicy?` | No | Enum value. |

#### `AuthorizeAssetAccessRequest`

| Property | Type | Required | Constraints |
|----------|------|----------|-------------|
| `TenantId` | `Guid` | Yes | Must match route. |
| `StudentId` | `Guid` | Yes | Student to authorize. |

---

### Response DTOs

#### `InitiateAssetUploadResult`

| Property | Type | Description |
|----------|------|-------------|
| `AssetId` | `Guid` | New asset identifier. |
| `StorageKey` | `string` | Path in storage. |
| `UploadUrl` | `string` | Pre-signed upload URL. |
| `ExpiresAtUtc` | `DateTime` | URL expiration time. |
| `FormFieldsJson` | `string` | Additional form fields (JSON). |

#### `FinalizeAssetUploadResult`

| Property | Type | Description |
|----------|------|-------------|
| `AssetId` | `Guid` | The finalized asset. |
| `Status` | `string` | `Ready` or `Processing`. |
| `Message` | `string` | Status message. |

#### `LearningAssetSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Asset identifier. |
| `Title` | `string?` | Asset title (for external links). |
| `Kind` | `string` | Asset kind name. |
| `ProcessingStatus` | `string` | Processing status name. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |

#### `LearningResourceSummary`

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `Guid` | Resource identifier. |
| `AssetId` | `Guid` | Linked asset. |
| `PlacementType` | `string` | Placement context. |
| `CourseId` | `Guid` | Course reference. |
| `SectionId` | `Guid?` | Section reference if applicable. |
| `LessonId` | `Guid?` | Lesson reference if applicable. |
| `Title` | `string` | Resource title. |
| `Description` | `string?` | Resource description. |
| `DeliveryPolicy` | `string` | Delivery policy name. |
| `Status` | `string` | Resource status. |
| `CreatedAtUtc` | `DateTime` | Creation timestamp. |

#### `AuthorizeAssetAccessResult`

| Property | Type | Description |
|----------|------|-------------|
| `Authorized` | `bool` | Whether access is granted. |
| `AccessUrl` | `string?` | Pre-signed access URL if authorized. |
| `ExpiresAtUtc` | `DateTime?` | URL expiration if applicable. |
| `Policy` | `string?` | Applied delivery policy. |

---

## 4. Enum Reference

### `LearningAssetKind`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Video` | Video file (MP4, MOV, etc.). |
| `2` | `Pdf` | PDF document. |
| `3` | `File` | Generic file (DOCX, XLSX, etc.). |
| `4` | `ExternalLink` | URL to external content. |

### `AssetProcessingStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `PendingUpload` | Upload initiated but not completed. |
| `2` | `Processing` | Video being transcoded. |
| `3` | `Ready` | Asset is ready for use. |
| `4` | `Failed` | Processing failed. |
| `5` | `Archived` | Asset is archived. |

### `ResourcePlacementType`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Course` | Resource belongs to the course level. |
| `2` | `Section` | Resource belongs to a specific section. |
| `3` | `Lesson` | Resource belongs to a specific lesson. |

### `ResourceDeliveryPolicy`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `StreamOnly` | Can only be streamed/played. |
| `2` | `ViewOnly` | Can be viewed but not downloaded. |
| `3` | `DownloadAllowed` | Can be downloaded. |
| `4` | `External` | Opens in external viewer (iframes not used). |

### `LearningResourceStatus`

| Value | Name | Description |
|-------|------|-------------|
| `1` | `Draft` | Resource is being prepared. Editable. |
| `2` | `Published` | Resource is visible to students. |
| `3` | `Archived` | Resource is hidden. |

---

## 5. Validation & Business Rules

### Backend-Enforced Rules

1. **File Size Limit**: Uploads must not exceed the configured maximum file size (e.g., 2GB for videos).
2. **Content Type Validation**: Only whitelisted MIME types are accepted (video/mp4, video/webm, application/pdf, etc.).
3. **Signed URL Expiration**: Upload URLs expire after a configurable time (default 60 minutes). Finalization must occur within the validity window.
4. **Asset Readiness**: Resources can only be published when the underlying asset has status `Ready`.
5. **External Link HTTPS**: External links must use HTTPS (unless configured otherwise).
6. **Delivery Policy Compatibility**: Some asset types may not support certain delivery policies (e.g., DownloadAllowed may not apply to videos).
7. **Placement Hierarchy**: Lesson placements require both `sectionId` and `lessonId`; Section placements require `sectionId`.
8. **Draft-Only Edits**: Resource metadata can only be updated when status is `Draft`.

### Frontend Validation Recommendations

- Validate file type and size client-side before initiating upload.
- Show upload progress when uploading to the signed URL.
- Handle expired signed URLs gracefully — re-initiate if needed.
- For videos, poll the resource status after finalization until `Ready`.
- Validate external URLs format before submission.

---

## 6. Error Handling Patterns

### Module Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `Asset.AssetNotFound` | 404 | Asset does not exist. | Refresh asset list. |
| `Asset.ResourceNotFound` | 404 | Resource does not exist. | Refresh resource list. |
| `Asset.InvalidInput` | 400 | Invalid file, URL, or other input. | Check file type, URL format, size. |
| `Asset.StorageProviderNotConfigured` | 409 | Storage provider not set up for tenant. | Contact administrator. |
| `Asset.VideoProviderNotConfigured` | 409 | Video processing provider not configured. | Contact administrator. |
| `Asset.UnsupportedFileType` | 400 | File MIME type not allowed. | Use a supported format. |
| `Asset.InvalidPlacement` | 409 | Invalid placement type or context. | Check section/lesson selection. |
| `Asset.IncompatibleDeliveryPolicy` | 409 | Delivery policy not supported for asset type. | Use compatible policy. |
| `Tenancy.AccessDenied` | 403 | Tenant context mismatch. | Verify tenant selection. |

---

## 7. Frontend Integration Best Practices

1. **Upload Progress UI**: Show a progress bar when uploading to the signed URL. Use XMLHttpRequest for progress events.
2. **Processing Indicator**: After finalization, show "Processing video..." with a spinner. Poll the resource endpoint every 5 seconds until `Ready`.
3. **Failed Upload Handling**: If finalization fails or processing fails, show an error with retry option.
4. **Drag-and-Drop Uploads**: Implement drag-and-drop file selection as a UX enhancement.
5. **File Type Icons**: Show appropriate icons for different asset kinds (video icon for videos, PDF icon for PDFs).
6. **Resource Cards**: Display resources as cards with title, description, asset type, and delivery policy badge.
7. **External Link Preview**: For external links, show the URL domain and a favicon.
8. **Delivery Policy Badges**: Color-code delivery policies (green=Stream, blue=View, purple=Download, gray=External).
9. **Course Builder Integration**: Integrate resource creation into the course builder workflow — show an "Add Resource" button in section/lesson editors.
10. **Student Access Flow**: Before displaying a resource to students, call authorize-access to get the pre-signed URL.
