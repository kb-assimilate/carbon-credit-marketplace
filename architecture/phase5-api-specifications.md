# Phase 5: API Integration Specifications

**Version**: 1.0.0  
**Date**: 19 August 2026  
**Status**: Production-Grade Specification  
**Author**: Principal API Architect  
**Upstream Dependencies**: Phase 2 §9, Phase 3 Executive Summary, Phase 4 §11

---

## Table of Contents

1. [API Conventions & Cross-Cutting Concerns](#1-api-conventions--cross-cutting-concerns)
2. [Auth & KYC Endpoints (Workflow 1)](#2-auth--kyc-endpoints-workflow-1)
3. [Project Lifecycle Endpoints (Workflow 1)](#3-project-lifecycle-endpoints-workflow-1)
4. [VVB Audit & Verification Endpoints (Workflow 2)](#4-vvb-audit--verification-endpoints-workflow-2)
5. [Marketplace Search & Orders (Workflow 4)](#5-marketplace-search--orders-workflow-4)
6. [OTC Trade Endpoints (Workflow 4, Path C)](#6-otc-trade-endpoints-workflow-4-path-c)
7. [Credit Retirement Endpoints (Workflow 5)](#7-credit-retirement-endpoints-workflow-5)
8. [Payment & Settlement Endpoints (Workflow 6)](#8-payment--settlement-endpoints-workflow-6)
9. [Tokenization Endpoints (Workflow 3, Optional)](#9-tokenization-endpoints-workflow-3-optional)
10. [Registry Adapter Integration Schemas](#10-registry-adapter-integration-schemas)
11. [D-MRV Ingestion Schemas](#11-d-mrv-ingestion-schemas)
12. [Webhook Specifications](#12-webhook-specifications)
13. [Rate Limiting & Caching Strategy](#13-rate-limiting--caching-strategy)
14. [Executive API Summary for Phase 6](#14-executive-api-summary-for-phase-6)

---

## 1. API Conventions & Cross-Cutting Concerns

### 1.1 Base URL & Versioning

| Environment | Base URL |
|:---|:---|
| Production | `https://api.carbonmarketplace.io/v1` |
| Staging | `https://api.staging.carbonmarketplace.io/v1` |

All endpoints are versioned via URL path prefix (`/v1`). Breaking changes increment the major version.

### 1.2 Authentication

All requests (except `POST /auth/register`) require a Bearer token obtained via Keycloak OAuth 2.0 / OIDC:

```
Authorization: Bearer <access_token>
```

Token format: JWT (RS256-signed by Keycloak). The `realm_access.roles` claim carries the actor's Keycloak role(s).

### 1.3 Keycloak Roles (Phase 2 §2.1)

| Role | Description |
|:---|:---|
| `PROJECT_DEVELOPER` | Farmers (smallholder track) and institutional developers |
| `VVB_AUDITOR` | Validation/Verification Body auditors |
| `CORPORATE_BUYER` | Institutional credit purchasers |
| `RETAIL_BUYER` | Individual/small-volume credit purchasers |
| `BROKER` | Intermediaries facilitating OTC trades |
| `PLATFORM_ADMIN` | Platform operators and support staff |

### 1.4 Standard Request Headers

| Header | Required | Description |
|:---|:---|:---|
| `Authorization` | Yes (except registration) | Bearer JWT token |
| `Content-Type` | Yes (on POST/PUT/PATCH) | `application/json` |
| `Idempotency-Key` | Conditional | UUID v4. Required on all state-mutating endpoints marked with 🔑 below. Per Phase 2 §4.4: keys are stored in Redis with a 24-hour TTL. |
| `X-Request-Id` | Recommended | UUID v4 for distributed tracing correlation (OpenTelemetry) |
| `Accept-Language` | Optional | ISO 639-1 language code for localized error messages |

### 1.5 Standard Response Envelope

All API responses follow this envelope:

```json
{
  "status": "success | error",
  "data": { },
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO-8601",
    "pagination": {
      "page": 1,
      "page_size": 25,
      "total_items": 1042,
      "total_pages": 42
    }
  },
  "errors": [
    {
      "code": "DOMAIN_ERROR_CODE",
      "message": "Human-readable message",
      "field": "optional.field.path",
      "details": {}
    }
  ]
}
```

### 1.6 Standard Error Codes

In addition to workflow-specific error states (documented per endpoint), the following apply globally:

| HTTP Status | Error Code | Description |
|:---|:---|:---|
| `401` | `AUTH_TOKEN_EXPIRED` | JWT has expired; re-authenticate via Keycloak |
| `401` | `AUTH_TOKEN_INVALID` | JWT signature verification failed |
| `403` | `AUTH_INSUFFICIENT_ROLE` | Caller's Keycloak role does not permit this operation |
| `403` | `AUTH_MFA_REQUIRED` | Financial operation requires MFA step-up (Phase 2 §8) |
| `409` | `IDEMPOTENCY_KEY_REUSED` | Idempotency key already processed; returns the original response |
| `422` | `VALIDATION_ERROR` | Request body fails JSON Schema validation |
| `429` | `RATE_LIMIT_EXCEEDED` | Rate limit exceeded; see §13 for per-endpoint limits |
| `503` | `SERVICE_UNAVAILABLE` | Downstream service unavailable; retry with exponential backoff |

### 1.7 Pagination

All list endpoints support cursor-based pagination:

| Parameter | Type | Default | Description |
|:---|:---|:---|:---|
| `page` | integer | `1` | Page number (1-indexed) |
| `page_size` | integer | `25` | Items per page (max: 100) |
| `sort_by` | string | varies | Sort field |
| `sort_order` | string | `desc` | `asc` or `desc` |

### 1.8 Idempotency Handling (Phase 2 §4.4)

State-mutating endpoints marked with 🔑 require the `Idempotency-Key` header. Behavior:

1. First request with a given key: processed normally, response cached in Redis (TTL: 24h).
2. Subsequent requests with the same key: return the cached response with HTTP `200` and header `Idempotency-Replayed: true`.
3. Requests with a key that is currently being processed: return `409 IDEMPOTENCY_KEY_IN_FLIGHT`.

The idempotency key format is: `{actor_id}:{resource}:{operation}:{client_nonce}` where `client_nonce` is a caller-generated UUID v4.

---

## 2. Auth & KYC Endpoints (Workflow 1)

### 2.1 `POST /auth/register`

Register a new user account on the platform.

**Auth**: None (public endpoint)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per IP (strict — abuse-sensitive)

**Request Body**:

```json
{
  "email": "string (email format, required for institutional)",
  "phone": "string (E.164 format, required for smallholder)",
  "password": "string (min 12 chars) — mutually exclusive with passkey_credential",
  "passkey_credential": {
    "id": "string (base64url)",
    "public_key": "string (base64url)",
    "attestation": "string (base64url)"
  },
  "developer_type": "SMALLHOLDER | INSTITUTIONAL",
  "preferred_language": "string (ISO 639-1, default: en)",
  "country_code": "string (ISO 3166-1 alpha-3)"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "user_id": "uuid",
    "role": "PROJECT_DEVELOPER",
    "kyc_status": "KYC_PENDING",
    "developer_type": "SMALLHOLDER | INSTITUTIONAL",
    "created_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 1 §1.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `409` | `USER_EMAIL_EXISTS` | Email already registered |
| `409` | `USER_PHONE_EXISTS` | Phone number already registered |
| `422` | `INVALID_COUNTRY_CODE` | Country code not in supported list |
| `422` | `PASSWORD_TOO_WEAK` | Password does not meet complexity requirements |

---

### 2.2 `POST /kyc/submit`

Submit KYC/AML verification documents. Supports both smallholder farmer and institutional developer tracks (Phase 4 §1.2).

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER`  
**Idempotency**: 🔑 Required  
**Rate Limit**: 3 req/hour per user (prevent spam submissions)

**Request Body**:

```json
{
  "kyc_track": "SMALLHOLDER | INSTITUTIONAL",
  "documents": [
    {
      "document_type": "PHOTO_ID | AADHAAR_OTP | PASSPORT | CORPORATE_REGISTRATION | DIRECTOR_ID | UBO_DECLARATION | COMMUNITY_ATTESTATION",
      "file_reference": "string (presigned upload URL reference)",
      "file_hash_sha256": "string (hex, for integrity verification)",
      "document_country": "string (ISO 3166-1 alpha-3)"
    }
  ],
  "consent": {
    "data_processing_consent": true,
    "consent_language": "string (ISO 639-1)",
    "consent_mechanism": "DIGITAL_FORM | AUDIO_RECORDING | FIELD_AGENT",
    "audio_consent_reference": "string (optional — object storage key for audio consent recording)"
  },
  "aadhaar_otp_verification": {
    "aadhaar_reference_id": "string (masked, last 4 digits)",
    "otp_session_token": "string"
  },
  "field_agent_attestation": {
    "agent_id": "uuid",
    "community_leader_name": "string",
    "gps_coordinates": {
      "latitude": "number",
      "longitude": "number"
    },
    "site_visit_timestamp": "ISO-8601"
  }
}
```

> **Note**: `aadhaar_otp_verification` is used only for the smallholder track in India. `field_agent_attestation` is the fallback for farmers with no formal ID (Phase 4 §1.4).

**Response** (`202 Accepted`):

```json
{
  "data": {
    "kyc_submission_id": "uuid",
    "kyc_status": "KYC_PENDING | KYC_MANUAL_REVIEW",
    "estimated_review_time": "string (e.g., '24-48 hours')",
    "submitted_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 1 §1.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `400` | `KYC_INCOMPLETE_DOCUMENTS` | Required documents missing for the selected track |
| `403` | `KYC_ACCOUNT_BLOCKED` | Account is in `KYC_BLOCKED` state (sanctions match) — cannot resubmit |
| `409` | `KYC_ALREADY_VERIFIED` | User already has `KYC_VERIFIED` status |
| `422` | `KYC_INVALID_DOCUMENT_TYPE` | Document type not valid for selected track |
| `422` | `KYC_CONSENT_REQUIRED` | `data_processing_consent` must be `true` |
| `503` | `KYC_PROVIDER_UNAVAILABLE` | Jumio/Onfido provider timeout (>120s). Submission queued for retry. |

---

### 2.3 `GET /kyc/status`

Check current KYC verification status.

**Auth**: Bearer JWT — Role: any authenticated user  
**Rate Limit**: 30 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "user_id": "uuid",
    "kyc_status": "KYC_PENDING | KYC_MANUAL_REVIEW | KYC_DOCS_REQUESTED | KYC_VERIFIED | KYC_REJECTED | KYC_BLOCKED",
    "kyc_track": "SMALLHOLDER | INSTITUTIONAL",
    "submitted_at": "ISO-8601 | null",
    "verified_at": "ISO-8601 | null",
    "next_refresh_due": "ISO-8601 | null",
    "rejection_reason": "string | null",
    "additional_docs_requested": ["string"]
  }
}
```

---

## 3. Project Lifecycle Endpoints (Workflow 1)

### 3.1 `POST /projects`

Create a new carbon credit project in DRAFT status.

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (must be `KYC_VERIFIED`)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 10 req/min per user

**Request Body**:

```json
{
  "project_name": "string (max 255 chars)",
  "registry_source": "VERRA | GOLD_STANDARD | ACR | CAR | PURO_EARTH | ISOMETRIC",
  "registry_project_id": "string (optional — if already registered externally)",
  "methodology_id": "string (e.g., 'VM0047', 'GS-METHODOLOGY-001')",
  "project_type": "AVOIDANCE | REMOVAL",
  "credit_type": "REDD_PLUS | ARR | IFM | COOKSTOVE | BIOCHAR | DAC | ENHANCED_WEATHERING | OTHER",
  "country_code": "string (ISO 3166-1 alpha-3)",
  "region": "string (optional)",
  "description": "string (max 5000 chars)",
  "expected_annual_credits": "integer (estimated tonnes CO2e/year)",
  "crediting_period_start": "ISO-8601 date",
  "crediting_period_end": "ISO-8601 date",
  "sdg_contributions": [
    {
      "sdg_number": "integer (1-17)",
      "description": "string"
    }
  ],
  "co_benefits": ["string"]
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "project_id": "uuid",
    "project_status": "DRAFT",
    "registry_source": "GOLD_STANDARD",
    "created_at": "ISO-8601"
  }
}
```

**Error States**:

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `KYC_NOT_VERIFIED` | Developer has not completed KYC verification |
| `422` | `INVALID_METHODOLOGY` | Methodology ID not recognized for the specified registry |
| `422` | `INVALID_CREDITING_PERIOD` | End date before start date or period exceeds registry maximum |

---

### 3.2 `POST /projects/{id}/documents`

Upload Project Design Document (PDD) or supporting documentation.

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (project owner)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 20 req/min per user

**Request Body**:

```json
{
  "document_type": "PDD | MONITORING_REPORT | BASELINE_STUDY | STAKEHOLDER_CONSULTATION | ENVIRONMENTAL_IMPACT | SUPPORTING",
  "file_reference": "string (presigned upload URL reference from MinIO/S3)",
  "file_hash_sha256": "string (hex)",
  "file_size_bytes": "integer",
  "file_name": "string",
  "mime_type": "string (application/pdf | image/tiff | application/geo+json)"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "document_id": "uuid",
    "document_type": "PDD",
    "object_key": "string (MinIO/S3 key)",
    "uploaded_at": "ISO-8601"
  }
}
```

**Error States**:

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `PROJECT_NOT_OWNED` | Authenticated user is not the project owner |
| `404` | `PROJECT_NOT_FOUND` | Project ID does not exist |
| `409` | `PROJECT_NOT_IN_DRAFT` | Project is not in DRAFT status — documents cannot be added |
| `413` | `FILE_TOO_LARGE` | File exceeds maximum allowed size (100 MB for PDD) |
| `422` | `FILE_HASH_MISMATCH` | SHA-256 of uploaded file does not match declared hash |

---

### 3.3 `POST /projects/{id}/boundary`

Submit or update the project's spatial boundary (GeoJSON polygon).

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (project owner)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 10 req/min per user

**Request Body**:

```json
{
  "boundary": {
    "type": "Polygon | MultiPolygon",
    "coordinates": [[[longitude, latitude], ...]],
    "crs": "EPSG:4326"
  },
  "boundary_source": "GPS_FIELD | SATELLITE_DIGITIZED | CADASTRAL_SURVEY | MANUAL_DRAWING",
  "area_hectares": "number (calculated client-side, validated server-side)"
}
```

**Response** (`202 Accepted`):

```json
{
  "data": {
    "project_id": "uuid",
    "boundary_status": "VALIDATING",
    "validation_id": "uuid",
    "estimated_validation_time": "string (e.g., '5-30 seconds')"
  }
}
```

> **Note**: Boundary validation is asynchronous. The D-MRV Ingestion Service performs topology checks and overlap detection (Phase 4 §1.3 Phase C). Results are delivered via the `project.boundary.validated` webhook or polling `GET /projects/{id}`.

**Error States** (from Workflow 1 §1.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `PROJECT_NOT_OWNED` | Authenticated user is not the project owner |
| `404` | `PROJECT_NOT_FOUND` | Project ID does not exist |
| `409` | `PROJECT_NOT_IN_DRAFT` | Boundary can only be submitted/updated for DRAFT projects |
| `422` | `BOUNDARY_INVALID_TOPOLOGY` | GeoJSON polygon has self-intersections or invalid ring orientation |
| `422` | `BOUNDARY_OVERLAP_DETECTED` | Boundary overlaps with existing project (conflicting project ID returned in `details`) |
| `422` | `BOUNDARY_AREA_MISMATCH` | Server-calculated area differs from declared `area_hectares` by >5% |

---

### 3.4 `POST /projects/{id}/submit`

Submit a DRAFT project for platform review.

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (project owner)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per user

**Request Body**: Empty (no body required)

**Preconditions**: Project must be in `DRAFT` status with PDD uploaded and boundary validated.

**Response** (`200 OK`):

```json
{
  "data": {
    "project_id": "uuid",
    "project_status": "SUBMITTED",
    "submitted_at": "ISO-8601",
    "estimated_review_time": "5-10 business days"
  }
}
```

**Error States** (from Workflow 1 §1.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `PROJECT_NOT_OWNED` | Authenticated user is not the project owner |
| `409` | `PROJECT_NOT_IN_DRAFT` | Project is not in DRAFT status |
| `422` | `PROJECT_MISSING_PDD` | No PDD document has been uploaded |
| `422` | `PROJECT_BOUNDARY_NOT_VALIDATED` | Boundary has not been submitted or validation is still pending |

---

### 3.5 `GET /projects/{id}`

Retrieve full project details.

**Auth**: Bearer JWT — Role: any authenticated user (public project data). Owner-only fields require `PROJECT_DEVELOPER` (owner) or `PLATFORM_ADMIN`.  
**Rate Limit**: 60 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "project_id": "uuid",
    "project_name": "string",
    "project_status": "DRAFT | SUBMITTED | REVISION_REQUESTED | LISTED | VALIDATED | VERIFIED | CANCELLED | REJECTED",
    "registry_source": "GOLD_STANDARD",
    "registry_project_id": "string | null",
    "methodology_id": "VM0047",
    "project_type": "REMOVAL",
    "credit_type": "ARR",
    "country_code": "IND",
    "region": "Maharashtra",
    "description": "string",
    "boundary": {
      "type": "Polygon",
      "coordinates": [[[...]]],
      "area_hectares": 1250.5
    },
    "boundary_status": "VALIDATED | VALIDATING | REJECTED",
    "quality_labels": [
      {"label": "CCP_APPROVED", "methodology": "VM0047", "assessed_at": "ISO-8601"}
    ],
    "sdg_contributions": [{"sdg_number": 13, "description": "Climate Action"}],
    "expected_annual_credits": 50000,
    "crediting_period_start": "2024-01-01",
    "crediting_period_end": "2044-01-01",
    "created_at": "ISO-8601",
    "updated_at": "ISO-8601",
    "submitted_at": "ISO-8601 | null",
    "listed_at": "ISO-8601 | null",
    "revision_notes": "string | null",
    "documents": [
      {
        "document_id": "uuid",
        "document_type": "PDD",
        "file_name": "string",
        "uploaded_at": "ISO-8601"
      }
    ]
  }
}
```

---

## 4. VVB Audit & Verification Endpoints (Workflow 2)

### 4.1 `GET /projects/{id}/eligible-vvbs`

List VVBs eligible to verify a given project, filtered by accreditation, methodology scope, and rotation rules (Phase 4 §2 — developer-initiated VVB selection).

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (project owner)  
**Rate Limit**: 30 req/min per user

**Query Parameters**:

| Param | Type | Description |
|:---|:---|:---|
| `page` | integer | Page number |
| `page_size` | integer | Items per page (max 50) |

**Response** (`200 OK`):

```json
{
  "data": [
    {
      "vvb_id": "uuid",
      "vvb_name": "string",
      "accreditation_body": "string",
      "accreditation_id": "string",
      "accreditation_expiry": "ISO-8601",
      "methodology_scopes": ["VM0047", "VM0006"],
      "registry_accreditations": ["VERRA", "GOLD_STANDARD"],
      "country_code": "string",
      "past_verifications_count": "integer",
      "rotation_eligible": true,
      "last_engagement_with_project": "ISO-8601 | null"
    }
  ]
}
```

**Error States** (from Workflow 2 §2.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `404` | `PROJECT_NOT_FOUND` | Project ID does not exist |
| `409` | `PROJECT_NOT_LISTED` | Project must be in LISTED (or later) status to request VVB assignment |

---

### 4.2 `POST /projects/{id}/vvb-assignment`

Assign a VVB to verify the project. Developer-initiated (Phase 4 §11 structural decision #3).

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (project owner)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per user

**Request Body**:

```json
{
  "vvb_id": "uuid",
  "verification_type": "VALIDATION | VERIFICATION | COMBINED",
  "engagement_scope": "string (optional — free text describing scope of work)",
  "requested_completion_date": "ISO-8601 date (optional)"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "verification_id": "uuid",
    "project_id": "uuid",
    "vvb_id": "uuid",
    "verification_status": "PENDING_VVB_ACCEPTANCE",
    "verification_type": "VERIFICATION",
    "created_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 2 §2.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `404` | `PROJECT_NOT_FOUND` | Project ID does not exist |
| `404` | `VVB_NOT_FOUND` | VVB ID does not exist |
| `409` | `PROJECT_NOT_LISTED` | Project must be in LISTED status |
| `409` | `VVB_NOT_ELIGIBLE` | VVB is not accredited for this project's methodology or registry |
| `409` | `VVB_ROTATION_VIOLATION` | VVB has exceeded maximum consecutive engagement limit for this project |
| `409` | `VERIFICATION_ALREADY_ACTIVE` | An active verification engagement already exists for this project |
| `422` | `VVB_ACCREDITATION_EXPIRED` | VVB's accreditation has expired |

---

### 4.3 `POST /verifications/{id}/accept`

VVB accepts the verification engagement.

**Auth**: Bearer JWT — Role: `VVB_AUDITOR` (assigned VVB)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 10 req/min per user

**Request Body**: Empty

**Response** (`200 OK`):

```json
{
  "data": {
    "verification_id": "uuid",
    "verification_status": "IN_PROGRESS",
    "accepted_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 2 §2.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `VVB_NOT_ASSIGNED` | Authenticated VVB is not the assigned auditor for this verification |
| `404` | `VERIFICATION_NOT_FOUND` | Verification ID does not exist |
| `409` | `VERIFICATION_NOT_PENDING` | Verification is not in `PENDING_VVB_ACCEPTANCE` status |
| `409` | `VERIFICATION_EXPIRED` | VVB did not accept within the acceptance window (configurable, default 14 days) |

---

### 4.4 `POST /verifications/{id}/submit-report`

VVB submits the verification/validation report.

**Auth**: Bearer JWT — Role: `VVB_AUDITOR` (assigned VVB)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per user

**Request Body**:

```json
{
  "report_file_reference": "string (presigned upload URL reference)",
  "report_hash_sha256": "string (hex)",
  "verification_opinion": "POSITIVE | NEGATIVE | QUALIFIED",
  "verified_volume_tonnes": "integer (tonnes CO2e verified by VVB)",
  "methodology_deviations": [
    {
      "deviation_type": "string",
      "description": "string",
      "materiality": "MATERIAL | NON_MATERIAL"
    }
  ],
  "monitoring_period_start": "ISO-8601 date",
  "monitoring_period_end": "ISO-8601 date",
  "dmrv_data_reviewed": true,
  "site_visit_conducted": true,
  "site_visit_date": "ISO-8601 date | null"
}
```

**Response** (`200 OK`):

```json
{
  "data": {
    "verification_id": "uuid",
    "verification_status": "REPORT_SUBMITTED",
    "report_object_key": "string",
    "submitted_at": "ISO-8601"
  }
}
```

**Error States**:

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `VVB_NOT_ASSIGNED` | Authenticated VVB is not the assigned auditor |
| `404` | `VERIFICATION_NOT_FOUND` | Verification ID does not exist |
| `409` | `VERIFICATION_NOT_IN_PROGRESS` | Verification must be in `IN_PROGRESS` status |
| `413` | `REPORT_FILE_TOO_LARGE` | Report exceeds 200 MB limit |
| `422` | `REPORT_HASH_MISMATCH` | SHA-256 of uploaded file does not match declared hash |

---

### 4.5 `POST /verifications/{id}/decision`

Platform Admin records the verification decision (approve, request revision, or reject).

**Auth**: Bearer JWT — Role: `PLATFORM_ADMIN`  
**Idempotency**: 🔑 Required  
**Rate Limit**: 30 req/min per user

**Request Body**:

```json
{
  "decision": "APPROVED | REVISION_REQUESTED | REJECTED",
  "decision_notes": "string (required for REVISION_REQUESTED and REJECTED)",
  "verified_volume_override": "integer | null (admin can override VVB's verified volume)",
  "quality_labels_to_apply": [
    {
      "label": "CCP_APPROVED | CORSIA_ELIGIBLE | CCB_GOLD | ARTICLE6_AUTHORIZED",
      "metadata": {}
    }
  ]
}
```

**Response** (`200 OK`):

```json
{
  "data": {
    "verification_id": "uuid",
    "verification_status": "COMPLETED | REVISION_REQUESTED | REJECTED",
    "project_status": "VERIFIED | REVISION_REQUESTED | REJECTED",
    "decided_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 2 §2.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `404` | `VERIFICATION_NOT_FOUND` | Verification ID does not exist |
| `409` | `VERIFICATION_NOT_REPORT_SUBMITTED` | Verification must have a submitted report before decision |
| `409` | `VERIFICATION_CANCELLED_VVB_DISQUALIFIED` | VVB's accreditation was revoked by registry during engagement — verification is void |
| `409` | `PROJECT_CANCELLED_REGISTRY_OVERRIDE` | Registry delisted/cancelled the project during verification |

---

## 5. Marketplace Search & Orders (Workflow 4)

### 5.1 `GET /marketplace/search`

Search available credits for purchase with faceted filtering. Backed by OpenSearch (Phase 2 §2.7).

**Auth**: Bearer JWT — Role: `CORPORATE_BUYER`, `RETAIL_BUYER`, `BROKER`, `PLATFORM_ADMIN`  
**Rate Limit**: 120 req/min per user (high-frequency endpoint)

**Query Parameters**:

| Param | Type | Description |
|:---|:---|:---|
| `q` | string | Free-text search across project name, description, methodology |
| `registry` | string[] | Filter by registry: `VERRA`, `GOLD_STANDARD`, `ACR`, `CAR`, `PURO_EARTH`, `ISOMETRIC` |
| `project_type` | string[] | `AVOIDANCE`, `REMOVAL` |
| `credit_type` | string[] | `REDD_PLUS`, `ARR`, `IFM`, `COOKSTOVE`, `BIOCHAR`, `DAC`, etc. |
| `vintage_year_min` | integer | Minimum vintage year |
| `vintage_year_max` | integer | Maximum vintage year |
| `country_code` | string[] | ISO 3166-1 alpha-3 country codes |
| `quality_labels` | string[] | `CCP_APPROVED`, `CORSIA_ELIGIBLE`, `CCB_GOLD`, `ARTICLE6_AUTHORIZED` |
| `price_min` | number | Minimum unit price (USD) |
| `price_max` | number | Maximum unit price (USD) |
| `sdg` | integer[] | SDG numbers (1-17) |
| `available_quantity_min` | integer | Minimum available quantity |
| `is_tokenized` | boolean | Filter for tokenized credits only |
| `sort_by` | string | `price`, `vintage_year`, `quantity`, `relevance` (default: `relevance`) |
| `sort_order` | string | `asc`, `desc` |
| `page` | integer | Page number |
| `page_size` | integer | Items per page (max 100) |

**Response** (`200 OK`):

```json
{
  "data": {
    "results": [
      {
        "listing_id": "uuid",
        "project_id": "uuid",
        "project_name": "string",
        "registry_source": "GOLD_STANDARD",
        "methodology_id": "VM0047",
        "project_type": "REMOVAL",
        "credit_type": "ARR",
        "vintage_year": 2025,
        "country_code": "IND",
        "quality_labels": [
          {"label": "CCP_APPROVED", "methodology": "VM0047"}
        ],
        "sdg_contributions": [{"sdg_number": 13}],
        "available_quantity": 15000,
        "unit_price_usd": 12.50,
        "total_price_usd": 187500.00,
        "seller_pseudonym": "string (pseudonym hash — not real identity)",
        "is_tokenized": false,
        "listing_created_at": "ISO-8601"
      }
    ],
    "facets": {
      "registry": [{"value": "GOLD_STANDARD", "count": 342}],
      "project_type": [{"value": "REMOVAL", "count": 187}],
      "credit_type": [{"value": "ARR", "count": 92}],
      "country_code": [{"value": "IND", "count": 156}],
      "quality_labels": [{"value": "CCP_APPROVED", "count": 234}],
      "vintage_year": [{"value": 2025, "count": 412}],
      "price_histogram": [
        {"range": "0-5", "count": 89},
        {"range": "5-10", "count": 234},
        {"range": "10-20", "count": 312}
      ]
    }
  }
}
```

**Error States** (from Workflow 4 §4.6):

| HTTP | Code | Condition |
|:---|:---|:---|
| `422` | `INVALID_SEARCH_PARAMS` | Invalid filter combination or out-of-range values |
| `503` | `SEARCH_INDEX_DEGRADED` | OpenSearch cluster unavailable; response served from PostgreSQL fallback (limited faceting) — response includes header `X-Search-Degraded: true` |

---

### 5.2 `GET /marketplace/credits/{id}`

Get full detail for a specific credit listing.

**Auth**: Bearer JWT — Role: `CORPORATE_BUYER`, `RETAIL_BUYER`, `BROKER`, `PLATFORM_ADMIN`  
**Rate Limit**: 60 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "listing_id": "uuid",
    "project_id": "uuid",
    "project_name": "string",
    "project_description": "string",
    "registry_source": "GOLD_STANDARD",
    "registry_project_id": "string",
    "methodology_id": "VM0047",
    "project_type": "REMOVAL",
    "credit_type": "ARR",
    "vintage_year": 2025,
    "country_code": "IND",
    "region": "Maharashtra",
    "boundary": {
      "type": "Polygon",
      "coordinates": [[[...]]]
    },
    "quality_labels": [],
    "sdg_contributions": [],
    "co_benefits": [],
    "available_quantity": 15000,
    "unit_price_usd": 12.50,
    "serial_number_range": "GS-1234567 to GS-1249567",
    "batch_id": "uuid",
    "is_tokenized": false,
    "tokenization_eligible": true,
    "seller_pseudonym": "string",
    "dmrv_summary": {
      "latest_ndvi_score": 0.72,
      "baseline_ndvi_score": 0.45,
      "monitoring_period": "2025-01-01 to 2025-12-31",
      "data_sources": ["Sentinel-2", "IoT-Dendrometer"],
      "content_hash": "sha256:abcdef..."
    },
    "verification_history": [
      {
        "verification_id": "uuid",
        "vvb_name": "string",
        "verification_opinion": "POSITIVE",
        "completed_at": "ISO-8601"
      }
    ],
    "listing_created_at": "ISO-8601",
    "listing_updated_at": "ISO-8601"
  }
}
```

---

### 5.3 `POST /marketplace/orders`

Create a purchase order for credits. Initiates the credit locking and payment flow (Phase 4 §4.3 and §6.2).

**Auth**: Bearer JWT — Role: `CORPORATE_BUYER`, `RETAIL_BUYER` (must be `KYC_VERIFIED`)  
**Idempotency**: 🔑 Required  
**MFA**: Required for orders > $10,000 USD (Phase 2 §8)  
**Rate Limit**: 10 req/min per user

**Request Body**:

```json
{
  "listing_id": "uuid",
  "quantity": "integer (number of credits to purchase)",
  "currency": "USD | EUR | INR",
  "buyer_notes": "string (optional)"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "order_id": "uuid",
    "order_status": "AWAITING_PAYMENT",
    "listing_id": "uuid",
    "quantity": 5000,
    "unit_price_usd": 12.50,
    "total_amount": {
      "value": 62500.00,
      "currency": "USD"
    },
    "payment_deadline": "ISO-8601 (30 min for card/UPI, 72h for wire — per Workflow 6 §6.4)",
    "payment_instructions": {
      "payment_methods_available": ["CARD", "WIRE_TRANSFER", "UPI"],
      "payment_reference": "string"
    },
    "credits_locked_until": "ISO-8601",
    "created_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 4 §4.6 and Workflow 6 §6.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `KYC_NOT_VERIFIED` | Buyer has not completed KYC verification |
| `403` | `AUTH_MFA_REQUIRED` | Order exceeds MFA threshold; step-up authentication required |
| `404` | `LISTING_NOT_FOUND` | Listing ID does not exist |
| `409` | `CREDITS_INSUFFICIENT_QUANTITY` | Requested quantity exceeds available supply |
| `409` | `CREDITS_NO_LONGER_AVAILABLE` | Credits were sold to another buyer (optimistic lock version mismatch — Phase 2 §6 Problem 2) |
| `409` | `CREDITS_REGISTRY_OVERRIDE` | Registry retired/cancelled credits between listing and order |
| `422` | `INVALID_CURRENCY` | Unsupported currency for this listing |
| `503` | `CREDIT_LOCK_CONTENTION` | Redis distributed lock acquisition failed after 3 retries. "Credits temporarily unavailable — please retry." |

---

### 5.4 `POST /marketplace/orders/{id}/pay`

Initiate payment for an existing order.

**Auth**: Bearer JWT — Role: `CORPORATE_BUYER`, `RETAIL_BUYER` (order owner)  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per user

> ⚠ **PROVISIONAL**: The `payment_method` and `payment_details` fields are structural stubs. Actual payment-provider integration fields are pending Phase 4a payments-compliance research.

**Request Body**:

```json
{
  "payment_method": "CARD | WIRE_TRANSFER | UPI | STABLECOIN",
  "payment_details": {
    "⚠ PROVISIONAL": "Payment-provider-specific fields TBD pending Phase 4a research",
    "payment_reference": "string (caller-provided reference for reconciliation)"
  }
}
```

**Response** (`202 Accepted`):

```json
{
  "data": {
    "order_id": "uuid",
    "order_status": "PAYMENT_PROCESSING",
    "payment_session_id": "string",
    "payment_status_poll_url": "/orders/{id}/settlement-status",
    "estimated_settlement_time": "string (e.g., 'instant' for card, '2-5 business days' for wire)"
  }
}
```

**Error States**:

| HTTP | Code | Condition |
|:---|:---|:---|
| `404` | `ORDER_NOT_FOUND` | Order ID does not exist |
| `409` | `ORDER_NOT_AWAITING_PAYMENT` | Order is not in `AWAITING_PAYMENT` status |
| `409` | `ORDER_PAYMENT_DEADLINE_EXPIRED` | Payment deadline has passed; credits have been released |
| `422` | `PAYMENT_METHOD_UNSUPPORTED` | Selected payment method not available for this order |

---

## 6. OTC Trade Endpoints (Workflow 4, Path C)

### 6.1 `POST /otc/trades`

Record an OTC (over-the-counter) trade negotiated off-platform (Phase 4 §4.5).

**Auth**: Bearer JWT — Role: `BROKER` (or `CORPORATE_BUYER` acting as buyer)  
**Idempotency**: 🔑 Required  
**MFA**: Required (all OTC trades — Phase 2 §8)  
**Rate Limit**: 5 req/min per user

**Request Body**:

```json
{
  "seller_id": "uuid (pseudonym_hash reference)",
  "buyer_id": "uuid (pseudonym_hash reference)",
  "credit_ids": ["uuid (credit IDs or batch reference)"],
  "quantity": "integer",
  "agreed_price": {
    "value": "number",
    "currency": "USD | EUR | INR"
  },
  "deal_reference": "string (broker's external deal reference)",
  "deal_type": "SPOT | FORWARD",
  "settlement_date": "ISO-8601 date (for FORWARD deals)",
  "notes": "string (optional)"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "otc_trade_id": "uuid",
    "trade_status": "PENDING_CONFIRMATION",
    "seller_confirmed": false,
    "buyer_confirmed": false,
    "confirmation_deadline": "ISO-8601 (72 hours from creation — Phase 4 §4.6)",
    "created_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 4 §4.6):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `AUTH_MFA_REQUIRED` | MFA step-up required for OTC trade creation |
| `404` | `SELLER_NOT_FOUND` | Seller user reference not found |
| `404` | `BUYER_NOT_FOUND` | Buyer user reference not found |
| `409` | `CREDITS_NOT_AVAILABLE` | Credits are not in ACTIVE status or not owned by seller |
| `409` | `CREDITS_LOCKED` | Credits are locked for another pending trade (`SALE_PENDING`) |
| `422` | `INVALID_DEAL_TYPE` | Invalid deal type for specified credits |
| `422` | `FORWARD_DATE_INVALID` | Forward settlement date in the past or beyond maximum allowed window |

---

### 6.2 `POST /otc/trades/{id}/confirm`

Confirm participation in an OTC trade. Both seller and buyer must confirm separately.

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (seller), `CORPORATE_BUYER` (buyer), or `BROKER`  
**Idempotency**: 🔑 Required  
**Rate Limit**: 10 req/min per user

**Request Body**:

```json
{
  "confirmation_role": "SELLER | BUYER",
  "confirmed": true
}
```

**Response** (`200 OK`):

```json
{
  "data": {
    "otc_trade_id": "uuid",
    "trade_status": "PENDING_CONFIRMATION | AWAITING_PAYMENT",
    "seller_confirmed": true,
    "buyer_confirmed": false,
    "both_confirmed_at": "ISO-8601 | null"
  }
}
```

> When both parties have confirmed, the trade status transitions to `AWAITING_PAYMENT`.

**Error States** (from Workflow 4 §4.6):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `NOT_TRADE_PARTICIPANT` | Authenticated user is not the seller, buyer, or broker for this trade |
| `404` | `OTC_TRADE_NOT_FOUND` | OTC trade ID does not exist |
| `409` | `OTC_TRADE_EXPIRED` | Confirmation deadline (72h) has passed |
| `409` | `ALREADY_CONFIRMED` | This party has already confirmed |

---

### 6.3 `POST /otc/trades/{id}/payment-confirmed`

Broker confirms that off-platform payment has been settled.

**Auth**: Bearer JWT — Role: `BROKER`  
**Idempotency**: 🔑 Required  
**Rate Limit**: 5 req/min per user

> ⚠ **PROVISIONAL**: OTC payment settlement happens off-platform (wire transfer between parties). The platform records the confirmation from the broker. Actual payment verification is pending Phase 4a research.

**Request Body**:

```json
{
  "payment_reference": "string (bank wire reference / transaction ID)",
  "payment_method": "WIRE_TRANSFER | ESCROW | OTHER",
  "payment_amount": {
    "value": "number",
    "currency": "USD | EUR | INR"
  },
  "payment_date": "ISO-8601"
}
```

**Response** (`200 OK`):

```json
{
  "data": {
    "otc_trade_id": "uuid",
    "trade_status": "SETTLING",
    "settlement_initiated_at": "ISO-8601"
  }
}
```

> After payment confirmation, the Marketplace & Trading Service executes the ownership transfer through the Credit Ledger Service. Trade status transitions: `SETTLING` → `COMPLETED`.

**Error States**:

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `NOT_TRADE_BROKER` | Authenticated user is not the broker for this trade |
| `404` | `OTC_TRADE_NOT_FOUND` | OTC trade ID does not exist |
| `409` | `OTC_TRADE_NOT_AWAITING_PAYMENT` | Both parties have not yet confirmed, or payment already recorded |
| `422` | `PAYMENT_AMOUNT_MISMATCH` | Reported payment amount does not match agreed trade price |

---

## 7. Credit Retirement Endpoints (Workflow 5)

### 7.1 `POST /credits/retire`

Initiate credit retirement. Triggers registry-side retirement flow (Phase 4 §5.2 Path A for Web2, §5.3 Path B for tokenized credits).

**Auth**: Bearer JWT — Role: `CORPORATE_BUYER`, `RETAIL_BUYER`, `PROJECT_DEVELOPER` (credit owner)  
**Idempotency**: 🔑 Required  
**MFA**: Required (all retirements — Phase 2 §8, financial operation)  
**Rate Limit**: 5 req/min per user

**Request Body**:

```json
{
  "credit_ids": ["uuid"],
  "quantity": "integer (total credits to retire across specified IDs)",
  "retirement_beneficiary": {
    "beneficiary_name": "string",
    "beneficiary_entity_type": "CORPORATE | INDIVIDUAL | GOVERNMENT",
    "beneficiary_country": "string (ISO 3166-1 alpha-3)"
  },
  "retirement_purpose": "VOLUNTARY_OFFSET | VCMI_CLAIM | CORSIA_COMPLIANCE | RESALE_PROHIBITION | OTHER",
  "retirement_reason": "string (free-text description)",
  "vcmi_claim_data": {
    "⚠ PROVISIONAL — VCMI fields pending verification against latest Claims Code": true,
    "claim_tier": "SILVER | GOLD | PLATINUM",
    "scope_coverage": ["SCOPE_1", "SCOPE_2", "SCOPE_3"],
    "internal_reduction_evidence_ref": "string (object storage key for evidence document, optional)"
  },
  "corsia_data": {
    "airline_icao_code": "string (optional — for CORSIA compliance retirements)",
    "compliance_period": "string"
  }
}
```

**Response** (`202 Accepted`):

```json
{
  "data": {
    "retirement_id": "uuid",
    "retirement_status": "RETIREMENT_PENDING",
    "credits_locked": ["uuid"],
    "quantity": 10000,
    "registry_retirement_estimated_time": "string (e.g., 'minutes for API registries, up to 48h for portal-only registries')",
    "certificate_available": false,
    "initiated_at": "ISO-8601"
  }
}
```

**Error States** (from Workflow 5 §5.5):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `CREDITS_NOT_OWNED` | Authenticated user does not own the specified credits |
| `403` | `AUTH_MFA_REQUIRED` | MFA step-up required for retirement |
| `400` | `CREDITS_WRONG_STATUS` | Credits are not in `ACTIVE` or `TRANSFERRED` status |
| `409` | `CREDITS_SALE_PENDING` | Credits are locked for a pending trade. "Complete or cancel the trade first." |
| `409` | `CREDITS_RETIREMENT_PENDING` | Credits already have a pending retirement request |
| `409` | `CREDITS_ALREADY_RETIRED` | One or more credits are already in `RETIRED` status |
| `503` | `REGISTRY_RETIREMENT_UNAVAILABLE` | Registry API/portal temporarily unreachable. Retirement queued for manual execution (SLA: 48h for portal-only registries). |

---

### 7.2 `GET /retirements/{id}`

Check retirement status and retrieve certificate when available.

**Auth**: Bearer JWT — Role: credit owner or `PLATFORM_ADMIN`  
**Rate Limit**: 30 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "retirement_id": "uuid",
    "retirement_status": "RETIREMENT_PENDING | RETIRED | RETIREMENT_FAILED | RETIREMENT_ROLLED_BACK",
    "credits": [
      {
        "credit_id": "uuid",
        "serial_number": "string",
        "registry_source": "GOLD_STANDARD"
      }
    ],
    "quantity": 10000,
    "retirement_beneficiary": {},
    "retirement_purpose": "VOLUNTARY_OFFSET",
    "registry_retirement_reference": "string | null",
    "registry_retirement_confirmed_at": "ISO-8601 | null",
    "on_chain_tx_hash": "string | null (if tokenized credits)",
    "certificate_available": true,
    "certificate_url": "/credits/{id}/retirement-certificate",
    "failure_reason": "string | null",
    "initiated_at": "ISO-8601",
    "completed_at": "ISO-8601 | null"
  }
}
```

---

### 7.3 `GET /credits/{id}/retirement-certificate`

Download the retirement certificate PDF. Only available after registry-side retirement confirmation (Phase 4 §5.4 — "certificate is never generated on a pending retirement").

**Auth**: Bearer JWT — Role: credit owner or `PLATFORM_ADMIN`  
**Rate Limit**: 10 req/min per user

**Response** (`200 OK`):

```
Content-Type: application/pdf
Content-Disposition: attachment; filename="retirement-certificate-{retirement_id}.pdf"
```

**Certificate Data Fields** (per Workflow 5 §5.4):

| Field | Source |
|:---|:---|
| Credit serial numbers | System (Credit Ledger) |
| Vintage year | System (credit metadata) |
| Project reference (registry + platform ID) | System |
| Methodology ID and version | System |
| Quality labels (CCP, CORSIA, Article 6, CCB) | System |
| Beneficiary name and entity type | User input (at retirement) |
| Retirement purpose | User input |
| ⚠ PROVISIONAL: VCMI claim tier | User input |
| ⚠ PROVISIONAL: Scope coverage | User input |
| Retirement timestamp (UTC) | System |
| Registry retirement reference number | System (from registry confirmation) |
| On-chain transaction hash (if tokenized) | System (if applicable) |

**Error States** (from Workflow 5 §5.5):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `CREDITS_NOT_OWNED` | Authenticated user is not the retirement beneficiary or credit owner |
| `404` | `RETIREMENT_NOT_FOUND` | Retirement ID does not exist |
| `409` | `CERTIFICATE_NOT_READY` | Retirement is still pending registry confirmation. Certificate is not yet generated. |
| `409` | `CERTIFICATE_GENERATION_FAILED` | PDF generation failed. Retry asynchronously. "Certificate generation in progress — please check back." |

---

## 8. Payment & Settlement Endpoints (Workflow 6)

> ⚠ **PROVISIONAL**: This entire section models the structural payment flow only. Actual payment-provider integration fields, cross-border/forex handling, FEMA/RBI compliance, PA-PG licensing, and escrow model are pending dedicated payments-compliance research (Phase 4 §6.3). Fields marked ⚠ are stubs.

### 8.1 `POST /marketplace/orders/{id}/payment-confirmed`

Webhook/callback endpoint for payment provider to confirm payment receipt. Also callable by `PLATFORM_ADMIN` for manual payment confirmation (e.g., wire transfers).

**Auth**: Payment provider webhook signature verification (HMAC-SHA256) OR Bearer JWT — Role: `PLATFORM_ADMIN`  
**Idempotency**: 🔑 Required (payment confirmation must be idempotent)  
**Rate Limit**: 30 req/min (webhook endpoint)

**Request Body**:

```json
{
  "payment_reference": "string (payment provider transaction ID)",
  "payment_amount": {
    "value": "number",
    "currency": "USD | EUR | INR"
  },
  "payment_method": "CARD | WIRE_TRANSFER | UPI | STABLECOIN",
  "payment_status": "CONFIRMED | FAILED | PARTIAL",
  "settlement_date": "ISO-8601",
  "⚠ PROVISIONAL — provider_specific_fields": {
    "note": "Actual payment-provider response fields TBD pending Phase 4a research"
  }
}
```

**Response** (`200 OK`):

```json
{
  "data": {
    "order_id": "uuid",
    "order_status": "PAYMENT_CONFIRMED | PAYMENT_DISCREPANCY | PAYMENT_FAILED",
    "payment_confirmed_at": "ISO-8601",
    "settlement_status": "SETTLING | COMPLETED | FAILED"
  }
}
```

**Error States** (from Workflow 6 §6.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `404` | `ORDER_NOT_FOUND` | Order ID does not exist |
| `409` | `ORDER_ALREADY_SETTLED` | Order has already been settled |
| `409` | `ORDER_PAYMENT_TIMED_OUT` | Payment window expired; credits have been released |
| `409` | `PAYMENT_AMOUNT_MISMATCH` | Payment amount does not match order total. Order enters `PAYMENT_DISCREPANCY` status for manual review. |
| `409` | `SETTLEMENT_FAILED_REGISTRY_OVERRIDE` | Registry retired/cancelled credits between payment and ownership transfer. Buyer refunded. |

---

### 8.2 `GET /orders/{id}/settlement-status`

Poll the settlement status of an order. Also receives status updates via `order.settlement.updated` webhook (§12).

**Auth**: Bearer JWT — Role: order buyer, order seller, or `PLATFORM_ADMIN`  
**Rate Limit**: 30 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "order_id": "uuid",
    "order_status": "AWAITING_PAYMENT | PAYMENT_PROCESSING | PAYMENT_CONFIRMED | PAYMENT_DISCREPANCY | PAYMENT_FAILED | PAYMENT_TIMED_OUT | SETTLING | COMPLETED | SETTLEMENT_FAILED",
    "payment_status": {
      "method": "WIRE_TRANSFER",
      "amount": {"value": 62500.00, "currency": "USD"},
      "confirmed_at": "ISO-8601 | null",
      "⚠ PROVISIONAL — fx_rate": "number | null (exchange rate if currency conversion applied)",
      "⚠ PROVISIONAL — fx_rate_locked_at": "ISO-8601 | null"
    },
    "settlement_status": {
      "credits_transferred": true,
      "seller_payout_status": "⚠ PROVISIONAL — PENDING | COMPLETED | FAILED",
      "platform_fee_deducted": "⚠ PROVISIONAL — number | null"
    },
    "timeline": [
      {"event": "ORDER_CREATED", "at": "ISO-8601"},
      {"event": "PAYMENT_CONFIRMED", "at": "ISO-8601"},
      {"event": "CREDITS_TRANSFERRED", "at": "ISO-8601"},
      {"event": "SETTLEMENT_COMPLETED", "at": "ISO-8601"}
    ]
  }
}
```

---

## 9. Tokenization Endpoints (Workflow 3, Optional)

### 9.1 `POST /credits/{batch_id}/tokenize`

Request tokenization of a credit batch. Only available when `RegistryConfig.registryTokenizationEnabled == true` for the credit's registry (currently `false` for all — Phase 3 §7, Phase 4 §3.3).

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (batch owner, must have valid ONCHAINID)  
**Idempotency**: 🔑 Required  
**MFA**: Required  
**Rate Limit**: 3 req/min per user

**Request Body**:

```json
{
  "batch_id": "uuid",
  "quantity": "integer (number of credits from batch to tokenize)",
  "recipient_wallet_address": "string (0x... — ERC-4337 smart account address)"
}
```

**Response** (`202 Accepted`):

```json
{
  "data": {
    "tokenization_request_id": "uuid",
    "batch_id": "uuid",
    "status": "CUSTODY_TRANSFER_PENDING",
    "credits_locked": true,
    "lock_reason": "TOKENIZATION",
    "estimated_completion": "string (e.g., '5-30 minutes for API registries')",
    "initiated_at": "ISO-8601"
  }
}
```

> **Note**: Tokenization is asynchronous. The flow progresses through: `CUSTODY_TRANSFER_PENDING` → `MINTING` → `TOKENIZED` (or `FAILED`). Status updates are delivered via the `credit.tokenized` webhook (§12) or polling `GET /credits/{batch_id}/tokenization-status`.

**Error States** (from Workflow 3 §3.4):

| HTTP | Code | Condition |
|:---|:---|:---|
| `403` | `TOKENIZATION_DISABLED_FOR_REGISTRY` | Tokenization is not enabled for this credit's registry |
| `403` | `ONCHAINID_NOT_VALID` | User does not have a valid ONCHAINID with required claims |
| `403` | `AUTH_MFA_REQUIRED` | MFA step-up required |
| `400` | `CREDITS_NOT_ACTIVE` | Credits must be in `ACTIVE` status for tokenization |
| `409` | `CREDITS_ALREADY_TOKENIZED` | Credits are already tokenized |
| `409` | `CREDITS_LOCKED` | Credits are locked for another operation |
| `504` | `CUSTODY_TRANSFER_TIMEOUT` | Registry did not confirm custody transfer within 30 minutes. Credits auto-unlock after 2 hours. |

---

### 9.2 `GET /credits/{batch_id}/tokenization-status`

Check tokenization progress.

**Auth**: Bearer JWT — Role: `PROJECT_DEVELOPER` (batch owner), `PLATFORM_ADMIN`  
**Rate Limit**: 30 req/min per user

**Response** (`200 OK`):

```json
{
  "data": {
    "tokenization_request_id": "uuid",
    "batch_id": "uuid",
    "status": "CUSTODY_TRANSFER_PENDING | MINTING | TOKENIZED | FAILED | CANCELLED",
    "custody_confirmed_at": "ISO-8601 | null",
    "on_chain_tx_hash": "string | null",
    "token_id": "string | null (ERC-1155 tokenId)",
    "polygon_contract_address": "string | null",
    "failure_reason": "string | null",
    "initiated_at": "ISO-8601",
    "completed_at": "ISO-8601 | null"
  }
}
```

---

## 10. Registry Adapter Integration Schemas

### 10.1 `RegistrySyncEvent` Normalized Schema

This is the canonical event schema produced by all six registry adapters (Phase 2 §4.2, §4.3) and consumed by the Credit Ledger Service via the `registry.normalized` Kafka topic. It is the single integration contract between the Registry Sync Service and the rest of the platform.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RegistrySyncEvent",
  "description": "Normalized event emitted by registry adapters onto Kafka. All adapters — API-first, hybrid, or scraping-first — must produce events conforming to this schema.",
  "type": "object",
  "required": ["event_id", "idempotency_key", "event_type", "registry_source", "timestamp", "data"],
  "properties": {
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique event identifier (UUID v4)"
    },
    "idempotency_key": {
      "type": "string",
      "description": "Deterministic key: {registry_source}:{serial_number_or_project_id}:{event_type}:{registry_timestamp_epoch}. Used by Credit Ledger Service for deduplication (Phase 2 §4.4).",
      "pattern": "^[A-Z_]+:[A-Za-z0-9_-]+:[A-Z_]+:[0-9]+$",
      "examples": ["GOLD_STANDARD:GS-12345:CREDIT_ISSUED:1724025600"]
    },
    "event_type": {
      "type": "string",
      "enum": [
        "PROJECT_REGISTERED",
        "PROJECT_VALIDATED",
        "PROJECT_VERIFIED",
        "PROJECT_CANCELLED",
        "CREDIT_ISSUED",
        "CREDIT_TRANSFERRED",
        "CREDIT_RETIRED",
        "CREDIT_CANCELLED",
        "CREDIT_BUFFERED",
        "CREDIT_STATUS_CHANGED",
        "VVB_ACCREDITATION_CHANGED"
      ]
    },
    "registry_source": {
      "type": "string",
      "enum": ["VERRA", "GOLD_STANDARD", "ACR", "CAR", "PURO_EARTH", "ISOMETRIC"]
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp of the event on the source registry (not the adapter processing time)"
    },
    "adapter_processed_at": {
      "type": "string",
      "format": "date-time",
      "description": "Timestamp when the adapter processed this event"
    },
    "adapter_id": {
      "type": "string",
      "description": "Identifier of the adapter instance that produced this event"
    },
    "data_source": {
      "type": "string",
      "enum": ["API", "SCRAPING", "BULK_EXPORT", "THIRD_PARTY_AGGREGATOR", "MANUAL_UPLOAD"],
      "description": "How the data was obtained — critical for confidence scoring"
    },
    "confidence_score": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "description": "Data source confidence: 1.0 for API, 0.9 for bulk export, 0.7 for scraping, 0.5 for third-party aggregator"
    },
    "data": {
      "type": "object",
      "description": "Event-type-specific payload. See sub-schemas below.",
      "oneOf": [
        { "$ref": "#/$defs/CreditIssuedData" },
        { "$ref": "#/$defs/CreditTransferredData" },
        { "$ref": "#/$defs/CreditRetiredData" },
        { "$ref": "#/$defs/CreditStatusChangedData" },
        { "$ref": "#/$defs/ProjectStatusChangedData" }
      ]
    },
    "raw_payload_hash": {
      "type": "string",
      "description": "SHA-256 hash of the raw registry response for audit and reconciliation"
    }
  },
  "$defs": {
    "CreditIssuedData": {
      "type": "object",
      "required": ["project_id", "serial_numbers", "vintage_year", "quantity"],
      "properties": {
        "project_id": {
          "type": "string",
          "description": "Registry-assigned project ID"
        },
        "serial_numbers": {
          "type": "array",
          "items": {"type": "string"},
          "description": "List of individual serial numbers, or range notation"
        },
        "serial_number_range": {
          "type": "object",
          "properties": {
            "start": {"type": "string"},
            "end": {"type": "string"},
            "count": {"type": "integer"}
          }
        },
        "vintage_year": {"type": "integer"},
        "quantity": {
          "type": "integer",
          "description": "Tonnes CO2e"
        },
        "methodology_id": {"type": "string"},
        "buffer_pool_deduction": {
          "type": "integer",
          "description": "Credits deducted for non-permanence buffer"
        },
        "registry_issuance_ref": {
          "type": "string",
          "description": "Registry's internal issuance reference"
        },
        "quality_labels": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "label": {"type": "string"},
              "metadata": {"type": "object"}
            }
          }
        }
      }
    },
    "CreditTransferredData": {
      "type": "object",
      "required": ["serial_numbers", "from_account", "to_account"],
      "properties": {
        "serial_numbers": {
          "type": "array",
          "items": {"type": "string"}
        },
        "from_account": {
          "type": "string",
          "description": "Registry account of the transferor"
        },
        "to_account": {
          "type": "string",
          "description": "Registry account of the transferee"
        },
        "transfer_type": {
          "type": "string",
          "enum": ["SALE", "CUSTODY", "INTERNAL"]
        },
        "registry_transfer_ref": {"type": "string"}
      }
    },
    "CreditRetiredData": {
      "type": "object",
      "required": ["serial_numbers", "retirement_beneficiary"],
      "properties": {
        "serial_numbers": {
          "type": "array",
          "items": {"type": "string"}
        },
        "retirement_beneficiary": {"type": "string"},
        "retirement_reason": {"type": "string"},
        "retirement_date": {
          "type": "string",
          "format": "date"
        },
        "registry_retirement_ref": {"type": "string"}
      }
    },
    "CreditStatusChangedData": {
      "type": "object",
      "required": ["serial_numbers", "previous_status", "new_status"],
      "properties": {
        "serial_numbers": {
          "type": "array",
          "items": {"type": "string"}
        },
        "previous_status": {"type": "string"},
        "new_status": {"type": "string"},
        "reason": {"type": "string"}
      }
    },
    "ProjectStatusChangedData": {
      "type": "object",
      "required": ["project_id", "previous_status", "new_status"],
      "properties": {
        "project_id": {"type": "string"},
        "project_name": {"type": "string"},
        "previous_status": {"type": "string"},
        "new_status": {"type": "string"},
        "reason": {"type": "string"}
      }
    }
  }
}
```

### 10.2 Registry Authentication & Sync Patterns

Per Phase 2 §4.4, each registry adapter follows one of three patterns:

| Registry | Pattern | Auth Mechanism | Polling Interval | Webhook Support |
|:---|:---|:---|:---|:---|
| **Isometric** | A: API-First | OAuth 2.0 + API key | 1–5 min (cursor-based pagination) | ⚠ PROVISIONAL — check if webhook subscription available |
| **Gold Standard** | A: API-First | Public REST API (read-only, API key) | 5–15 min (timestamp-based) | No |
| **Puro.earth** | A: API-First | API key (MyPuro API + Public Registry API) | 5–15 min | No |
| **Verra** | B: Hybrid ⚠ PROVISIONAL | Track 1: OAuth 2.0 + API key (Digital Gateway). Track 2: Headless browser session cookies (Playwright). Track 3: Third-party aggregator API key. | Track 1: 5 min. Track 2: 30–60 min. Track 3: Daily. | No |
| **ACR** | C: Scraping-First | Headless browser session (Playwright) + bulk CSV/JSON export credentials | 30–60 min (scraping) + daily (bulk export) | No |
| **CAR** | C: Scraping-First | Headless browser session (Playwright) + bulk export credentials | 30–60 min (scraping) + daily (bulk export) | No |

**Scraping Resilience** (Phase 2 §4.4 Pattern C):
- DOM selectors externalized as configuration (not hardcoded)
- Screenshot-on-failure for debugging
- Anomaly detection: >20% record count drop triggers quarantine
- Manual fallback: CSV/PDF upload via admin dashboard

### 10.3 Adapter Health & Monitoring (Internal API)

**`GET /internal/registry-sync/health`** (Internal only — not exposed via API Gateway)

```json
{
  "adapters": [
    {
      "registry": "GOLD_STANDARD",
      "status": "HEALTHY | DEGRADED | UNHEALTHY",
      "last_successful_sync": "ISO-8601",
      "sync_lag_seconds": 120,
      "events_processed_last_hour": 42,
      "checkpoint_offset": "string",
      "canary_check_passed": true,
      "error_count_last_hour": 0
    }
  ]
}
```

---

## 11. D-MRV Ingestion Schemas

### 11.1 Satellite Tile Metadata Ingestion

Internal endpoint consumed by the D-MRV Ingestion Service pipeline. Satellite imagery (Sentinel-2, PlanetScope) is ingested as Cloud-Optimized GeoTIFF (COG) files; this endpoint records the metadata and content hash for blockchain referencing (Phase 2 §2.3, Phase 3 constraint: raw data never on-chain).

**`POST /internal/dmrv/satellite-tiles`** (Internal service-to-service, mTLS authenticated)

**Request Body**:

```json
{
  "tile_id": "uuid",
  "project_id": "uuid",
  "satellite_source": "SENTINEL_2 | PLANETSCOPE | LANDSAT_8 | CUSTOM",
  "acquisition_date": "ISO-8601",
  "tile_footprint": {
    "type": "Polygon",
    "coordinates": [[[longitude, latitude], ...]]
  },
  "spectral_bands": ["B02", "B03", "B04", "B08", "B11", "B12"],
  "spatial_resolution_meters": 10,
  "cloud_cover_percentage": 12.5,
  "file_format": "COG",
  "file_reference": {
    "object_key": "string (MinIO/S3 path)",
    "file_size_bytes": "integer",
    "storage_tier": "HOT | WARM | COLD"
  },
  "content_hash": {
    "algorithm": "SHA-256",
    "hash": "string (hex)",
    "description": "Hash of the derived product (NDVI map, biomass estimate) — this hash is what gets referenced on-chain per Phase 3 constraint"
  },
  "derived_products": [
    {
      "product_type": "NDVI_MAP | BIOMASS_ESTIMATE | CHANGE_DETECTION | DEFORESTATION_ALERT",
      "object_key": "string (MinIO/S3 path)",
      "content_hash": {
        "algorithm": "SHA-256",
        "hash": "string (hex)"
      },
      "computed_at": "ISO-8601"
    }
  ],
  "processing_pipeline_version": "string"
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "tile_id": "uuid",
    "project_id": "uuid",
    "content_hash": "sha256:abcdef...",
    "stored_at": "ISO-8601",
    "storage_tier": "HOT"
  }
}
```

### 11.2 IoT Telemetry Ingestion

Ingestion endpoint for IoT sensor data (dendrometers, flux towers, cookstove thermal loggers, DAC flow meters). Stored in TimescaleDB at full resolution for 90 days, then downsampled (Phase 2 §6 Problem 3).

**`POST /internal/dmrv/telemetry`** (Internal service-to-service, mTLS authenticated)

**Request Body** (batch submission):

```json
{
  "project_id": "uuid",
  "device_id": "string",
  "device_type": "DENDROMETER | FLUX_TOWER | COOKSTOVE_LOGGER | DAC_FLOW_METER | WEATHER_STATION | SOIL_SENSOR",
  "readings": [
    {
      "timestamp": "ISO-8601",
      "metrics": {
        "metric_name": "number (e.g., 'diameter_mm': 245.3, 'co2_flux_umol_m2_s': 12.7)"
      },
      "gps_coordinates": {
        "latitude": "number",
        "longitude": "number",
        "altitude_m": "number (optional)"
      },
      "battery_level_pct": "number (0-100, optional)",
      "signal_quality": "number (0-1, optional)"
    }
  ],
  "batch_content_hash": {
    "algorithm": "SHA-256",
    "hash": "string (hex — hash of this batch for integrity verification and on-chain referencing)"
  },
  "transmission_metadata": {
    "protocol": "MQTT | HTTP | LORAWAN | SATELLITE_BACKHAUL",
    "firmware_version": "string",
    "transmission_timestamp": "ISO-8601"
  }
}
```

**Response** (`201 Created`):

```json
{
  "data": {
    "ingestion_id": "uuid",
    "readings_accepted": "integer",
    "readings_rejected": "integer (e.g., duplicate timestamps)",
    "content_hash": "sha256:abcdef...",
    "stored_at": "ISO-8601"
  }
}
```

### 11.3 D-MRV Content Hash Reference Schema

This schema defines how D-MRV data is referenced from the blockchain layer (Phase 3 constraint: raw data never on-chain, only content hashes).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "DMRVContentReference",
  "description": "On-chain reference to off-chain D-MRV data. Stored in CreditVault metadata.",
  "type": "object",
  "required": ["content_hash", "data_type", "storage_uri"],
  "properties": {
    "content_hash": {
      "type": "string",
      "pattern": "^sha256:[a-f0-9]{64}$",
      "description": "SHA-256 hash of the derived D-MRV product (not raw data)"
    },
    "data_type": {
      "type": "string",
      "enum": ["SATELLITE_DERIVED", "IOT_SUMMARY", "MONITORING_REPORT", "VERIFICATION_REPORT"]
    },
    "storage_uri": {
      "type": "string",
      "format": "uri",
      "description": "URI to retrieve the full data from object storage (via authenticated API)"
    },
    "computed_at": {
      "type": "string",
      "format": "date-time"
    },
    "monitoring_period": {
      "type": "object",
      "properties": {
        "start": {"type": "string", "format": "date"},
        "end": {"type": "string", "format": "date"}
      }
    }
  }
}
```

---

## 12. Webhook Specifications

The platform emits webhooks to registered external parties for asynchronous event notifications. All webhooks use the same envelope format and signature verification scheme.

### 12.1 Webhook Envelope Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "WebhookEnvelope",
  "type": "object",
  "required": ["webhook_id", "event_type", "timestamp", "data"],
  "properties": {
    "webhook_id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique webhook delivery ID"
    },
    "event_type": {
      "type": "string",
      "enum": [
        "kyc.status.changed",
        "project.status.changed",
        "project.boundary.validated",
        "verification.status.changed",
        "credit.activated",
        "credit.tokenized",
        "credit.transferred",
        "credit.retirement.initiated",
        "credit.retirement.completed",
        "credit.retirement.failed",
        "order.created",
        "order.payment.confirmed",
        "order.settlement.completed",
        "order.settlement.failed",
        "otc.trade.confirmed",
        "otc.trade.settled",
        "registry.sync.anomaly",
        "registry.sync.conflict"
      ]
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "api_version": {
      "type": "string",
      "const": "v1"
    },
    "data": {
      "type": "object",
      "description": "Event-specific payload (see sub-schemas below)"
    }
  }
}
```

### 12.2 Signature Verification

All webhook deliveries include the following headers for verification:

| Header | Description |
|:---|:---|
| `X-Webhook-Signature` | HMAC-SHA256 signature of the raw request body using the subscriber's webhook secret |
| `X-Webhook-Timestamp` | Unix timestamp (seconds) of when the webhook was generated |
| `X-Webhook-Id` | Unique delivery ID (same as `webhook_id` in body, for convenience) |

**Signature computation**:

```
signature = HMAC-SHA256(
  key: webhook_secret,
  message: "{X-Webhook-Timestamp}.{raw_request_body}"
)
```

**Verification steps** (for webhook subscribers):

1. Extract `X-Webhook-Timestamp` and `X-Webhook-Signature` headers.
2. Reject if `X-Webhook-Timestamp` is more than 5 minutes old (replay protection).
3. Compute expected signature: `HMAC-SHA256(webhook_secret, "{timestamp}.{body}")`.
4. Compare computed signature with `X-Webhook-Signature` using constant-time comparison.

### 12.3 Retry Policy

| Attempt | Delay | Max Attempts |
|:---|:---|:---|
| 1st retry | 30 seconds | — |
| 2nd retry | 2 minutes | — |
| 3rd retry | 15 minutes | — |
| 4th retry | 1 hour | — |
| 5th retry | 4 hours | **Final attempt** |

If all 5 retries fail (non-2xx response), the webhook subscription is marked as `FAILING`. After 3 consecutive delivery failures across different events, the subscription is marked `DISABLED` and the subscriber is notified via email.

### 12.4 Key Webhook Event Payloads

#### `kyc.status.changed`

```json
{
  "user_id": "uuid",
  "previous_status": "KYC_PENDING",
  "new_status": "KYC_VERIFIED",
  "kyc_track": "INSTITUTIONAL",
  "changed_at": "ISO-8601"
}
```

#### `credit.retirement.completed`

```json
{
  "retirement_id": "uuid",
  "credit_ids": ["uuid"],
  "serial_numbers": ["GS-12345", "GS-12346"],
  "quantity": 10000,
  "registry_source": "GOLD_STANDARD",
  "registry_retirement_ref": "string",
  "beneficiary_name": "string",
  "retirement_purpose": "VOLUNTARY_OFFSET",
  "certificate_url": "/credits/{id}/retirement-certificate",
  "on_chain_tx_hash": "string | null",
  "retired_at": "ISO-8601"
}
```

#### `order.settlement.completed`

```json
{
  "order_id": "uuid",
  "buyer_pseudonym": "string",
  "seller_pseudonym": "string",
  "credit_ids": ["uuid"],
  "quantity": 5000,
  "total_amount": {"value": 62500.00, "currency": "USD"},
  "settled_at": "ISO-8601"
}
```

#### `registry.sync.conflict`

```json
{
  "registry_source": "VERRA",
  "conflict_type": "STATE_DIVERGENCE | SERIAL_COLLISION | UNAUTHORIZED_RETIREMENT",
  "affected_serial_numbers": ["string"],
  "platform_state": "ACTIVE",
  "registry_state": "RETIRED",
  "detected_at": "ISO-8601",
  "severity": "P1_CRITICAL | P2_HIGH | P3_MEDIUM"
}
```

### 12.5 Webhook Subscription Management

#### `POST /webhooks/subscriptions`

**Auth**: Bearer JWT — Role: any authenticated user  
**Rate Limit**: 5 req/min per user

```json
{
  "url": "string (HTTPS URL)",
  "events": ["kyc.status.changed", "credit.retirement.completed"],
  "secret": "string (min 32 chars — used for HMAC-SHA256 signature)"
}
```

#### `DELETE /webhooks/subscriptions/{id}`

**Auth**: Bearer JWT — Role: subscription owner

---

## 13. Rate Limiting & Caching Strategy

### 13.1 Rate Limiting (Kong API Gateway + Redis)

Rate limits are enforced at the Kong API Gateway using Redis-backed token bucket counters (Phase 2 §2.5).

| Endpoint Group | Rate Limit | Burst | Window | Key |
|:---|:---|:---|:---|:---|
| **Auth** (`/auth/register`) | 5 req/min | 2 | Per IP | `rl:ip:{ip}:auth` |
| **KYC** (`/kyc/*`) | 3 req/hour (submit), 30 req/min (status) | 1 | Per user | `rl:user:{uid}:kyc` |
| **Projects** (write) | 10 req/min | 3 | Per user | `rl:user:{uid}:proj:w` |
| **Projects** (read) | 60 req/min | 10 | Per user | `rl:user:{uid}:proj:r` |
| **Verifications** (write) | 10 req/min | 3 | Per user | `rl:user:{uid}:vrf:w` |
| **Marketplace Search** | 120 req/min | 20 | Per user | `rl:user:{uid}:search` |
| **Marketplace Orders** (write) | 10 req/min | 3 | Per user | `rl:user:{uid}:order:w` |
| **OTC Trades** (write) | 5 req/min | 2 | Per user | `rl:user:{uid}:otc:w` |
| **Retirement** | 5 req/min | 2 | Per user | `rl:user:{uid}:retire` |
| **Tokenization** | 3 req/min | 1 | Per user | `rl:user:{uid}:token` |
| **Settlement Status** (polling) | 30 req/min | 5 | Per user | `rl:user:{uid}:settle` |
| **Webhooks** (management) | 5 req/min | 2 | Per user | `rl:user:{uid}:whk` |
| **Internal APIs** | 1000 req/min | 100 | Per service | `rl:svc:{svc_id}` |

**Rate Limit Response Headers** (on every response):

| Header | Description |
|:---|:---|
| `X-RateLimit-Limit` | Maximum requests in the current window |
| `X-RateLimit-Remaining` | Remaining requests in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait (only on `429` responses) |

### 13.2 Caching Strategy (Redis — Phase 2 §2.5)

| Resource | Cache Strategy | TTL | Invalidation |
|:---|:---|:---|:---|
| **Credit state** | Write-through (Phase 2 §2.5) | N/A (always fresh) | Invalidated on every Credit Ledger state transition |
| **Marketplace search results** | TTL-based | 30 seconds | CDC event from Credit Ledger → OpenSearch → cache bust |
| **Marketplace credit detail** | TTL-based | 60 seconds | CDC event invalidation |
| **Project listings** | TTL-based | 60 seconds | Invalidated on project status change |
| **Eligible VVBs list** | TTL-based | 5 minutes | Invalidated on VVB accreditation change |
| **Quality labels / ICVCM CCP list** | TTL-based | 24 hours | Daily refresh from ICVCM source |
| **KYC status** | Write-through | N/A | Updated on every KYC state transition |
| **Registry sync checkpoint** | Write-through | N/A | Updated every sync cycle |
| **Facet counts (OpenSearch aggregations)** | TTL-based | 30 seconds | Natural expiry |

**Cache Key Convention**: `cache:{service}:{resource_type}:{resource_id}:{version}`

**Cache Headers** (on cacheable GET responses):

| Header | Value |
|:---|:---|
| `Cache-Control` | `private, max-age={ttl}` |
| `ETag` | `"{resource_version_hash}"` |
| `X-Cache` | `HIT` or `MISS` |

---

## 14. Executive API Summary for Phase 6

**Date**: 19 August 2026  
**Purpose**: Fixed context for Phase 6 (red-teaming/risk audit).

---

### Structural Decisions

**1. RESTful with OpenAPI 3.0 conventions**: All public endpoints follow REST semantics with JSON request/response bodies. Internal service-to-service communication uses the same JSON Schema contracts over mTLS.

**2. Idempotency on all state mutations**: Every write endpoint requires an `Idempotency-Key` header. Keys are cached in Redis (24h TTL). The idempotency key format is deterministic: `{actor_id}:{resource}:{operation}:{client_nonce}`.

**3. Workflow-specific error states**: Error responses are drawn from each workflow's failure-state table (Phase 4), not generic HTTP status codes. Every error includes a machine-readable `code` and human-readable `message`.

**4. Registry adapter normalization**: All six registry adapters (API-first, hybrid, scraping-first) produce events conforming to the `RegistrySyncEvent` JSON Schema (§10.1). The Credit Ledger Service consumes only the `registry.normalized` Kafka topic — it is registry-agnostic.

**5. D-MRV data never on-chain**: Satellite tile metadata and IoT telemetry are ingested via internal APIs (§11). Content hashes (SHA-256 of derived products) are the only D-MRV reference stored on-chain, per Phase 3's constraint.

**6. Webhook-first async notifications**: All async events (KYC status, retirement completion, settlement) are delivered via signed webhooks (HMAC-SHA256) with 5-attempt retry and replay protection (§12).

**7. Rate limiting via Kong + Redis**: Per-user and per-IP rate limits enforced at the API Gateway. Marketplace search has the highest throughput allowance (120 req/min). Auth registration has the strictest (5 req/min per IP).

### Endpoint Coverage vs. Phase 4 §11 Interface Table

| Workflow | Required Endpoints | Specified In |
|:---|:---|:---|
| **1. Developer Onboarding** | `POST /auth/register`, `POST /kyc/submit`, `GET /kyc/status`, `POST /projects`, `POST /projects/{id}/documents`, `POST /projects/{id}/boundary`, `POST /projects/{id}/submit`, `GET /projects/{id}` | §2, §3 |
| **2. VVB Audit** | `GET /projects/{id}/eligible-vvbs`, `POST /projects/{id}/vvb-assignment`, `POST /verifications/{id}/accept`, `POST /verifications/{id}/submit-report`, `POST /verifications/{id}/decision` | §4 |
| **3. Registry Issuance** | Internal (Kafka event-driven, no user-facing API). `POST /credits/{batch_id}/tokenize`, `GET /credits/{batch_id}/tokenization-status` | §9 |
| **4. Primary Sale & Trading** | `GET /marketplace/search`, `GET /marketplace/credits/{id}`, `POST /marketplace/orders`, `POST /marketplace/orders/{id}/pay`, `POST /otc/trades`, `POST /otc/trades/{id}/confirm`, `POST /otc/trades/{id}/payment-confirmed` | §5, §6 |
| **5. Credit Retirement** | `POST /credits/retire`, `GET /retirements/{id}`, `GET /credits/{id}/retirement-certificate` | §7 |
| **6. Corporate Payment** | `POST /marketplace/orders` (shared), `POST /marketplace/orders/{id}/payment-confirmed`, `GET /orders/{id}/settlement-status` | §8 |

### Items Flagged ⚠ PROVISIONAL

| Item | Section | Reason | Resolution Required Before Implementation |
|:---|:---|:---|:---|
| **Payment provider integration fields** | §5.4, §8.1, §8.2 | No payment provider selected. `payment_details` and provider-specific response fields are stubs. | Complete Phase 4a payments-compliance research. Select provider (Stripe/Razorpay/SWIFT). Define concrete request/response schemas. |
| **Cross-border FX handling** | §8.2 | `fx_rate` and `fx_rate_locked_at` fields are stubs. Rate locking mechanism, FX risk bearer, and LRS applicability undetermined. | Include in payments-compliance research. |
| **Seller payout / platform fee deduction** | §8.2 | `seller_payout_status` and `platform_fee_deducted` are stubs. Fee structure, GST applicability, and withholding tax treatment undetermined. | Commission Indian tax counsel opinion. Define fee schedule. |
| **VCMI Claims Code fields** | §7.1, §7.3 | `vcmi_claim_data` fields (`claim_tier`, `scope_coverage`, `internal_reduction_evidence_ref`) not verified against latest VCMI Claims Code of Practice. | Obtain latest VCMI specification. Map exact field requirements. Update schema. |
| **Verra adapter authentication** | §10.2 | Verra Digital Gateway API production access and scraping fallback are provisional. | Verify Verra developer portal access. Test production Digital Gateway endpoints. |
| **Isometric webhook support** | §10.2 | Webhook subscription capability for Isometric marked as provisional. | Verify with Isometric API documentation. |
| **OTC payment verification** | §6.3 | Platform records broker's payment confirmation without independent verification. | Determine if independent payment verification is required (bank API integration or escrow service). |
| **Escrow model for large trades** | §8 | Whether platform holds funds in escrow is TBD. | Determine RBI regulatory requirements. Evaluate third-party escrow partners. |
| **All registry custody agreements** | §9.1 | Tokenization endpoints exist but are gated behind `registryTokenizationEnabled` flag (currently `false` for all registries). | Execute bilateral custody agreements with Gold Standard, Puro.earth, Isometric before enabling. |

---

*End of Phase 5 API Integration Specifications Document.*
