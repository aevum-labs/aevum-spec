# Aevum Protocol Specification
## Section 10: HTTP API

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

---

### 10.1 Overview

The `aevum-server` package exposes the five functions over HTTP for
non-Python consumers. It is a separate package; FastAPI MUST NOT be
a dependency of `aevum-core`.

Two distinct surfaces are defined:
- **Public Data API** at `/v1/` — what external systems call
- **Admin API** at `/_aevum/v1/` — what operators use

### 10.2 Authentication

All endpoints except `GET /v1/health` MUST require authentication.

**API Key:** `X-Aevum-Key: <key>` request header.
Unauthenticated or invalid-key requests MUST return HTTP 401.

**OIDC Bearer Token:** `Authorization: Bearer <token>` request header.
Token validation MUST be delegated to the `aevum-oidc` complication.

### 10.3 Public Data API — /v1/

#### POST /v1/ingest

Calls the `ingest` function.

Request body (JSON):
```json
{
  "data": {},
  "provenance": {
    "source_id": "string",
    "ingest_audit_id": "string",
    "chain_of_custody": [],
    "classification": 0,
    "model_id": null
  },
  "purpose": "string",
  "subject_id": "string"
}
```
All fields REQUIRED except `model_id`.

Supports `Idempotency-Key` request header (UUID v4).
When provided, duplicate requests with the same key MUST return the
original response without re-executing.

Success: HTTP 200 with OutputEnvelope JSON body.
Barrier triggered: HTTP 200 with OutputEnvelope status="error" or status="crisis".
Validation error: HTTP 422 with RFC 9457 problem body.
Auth failure: HTTP 401 with RFC 9457 problem body.

#### POST /v1/query

Calls the `query` function.

Request body (JSON):
```json
{
  "purpose": "string",
  "subject_ids": ["string"],
  "constraints": {},
  "classification_max": 0
}
```
`purpose` and `subject_ids` REQUIRED. Others optional.

Success: HTTP 200 with OutputEnvelope JSON body.

#### POST /v1/commit

Calls the `commit` function.

Request body (JSON):
```json
{
  "event_type": "string",
  "payload": {}
}
```
Both fields REQUIRED. `event_type` MUST NOT use kernel-reserved prefixes.

Supports `Idempotency-Key` header. Same semantics as `/v1/ingest`.

Success: HTTP 200 with OutputEnvelope JSON body.

#### GET /v1/replay/{audit_id}

Calls the `replay` function. `audit_id` is URL-encoded.

Optional query parameters:
- `scope`: comma-separated list of fields to include in reconstruction

Success: HTTP 200 with OutputEnvelope JSON body.
Not found: HTTP 404 with RFC 9457 problem body.
Consent failure: HTTP 403 with RFC 9457 problem body.

#### GET /v1/review/{audit_id}

Returns current status of a pending review.

Success: HTTP 200 with OutputEnvelope JSON body showing current review status.
Not found: HTTP 404 with RFC 9457 problem body.

#### POST /v1/review/{audit_id}/approve

Records human approval of a pending review.

Request body (JSON): `{}` (empty object, no body required)

Success: HTTP 200 with OutputEnvelope JSON body.
Not found: HTTP 404.
Already resolved: HTTP 409 with RFC 9457 problem body.

#### POST /v1/review/{audit_id}/veto

Records human veto of a pending review.

Request body (JSON): `{}` (empty object, no body required)

Same response codes as approve.

#### GET /v1/health

Health check. MUST NOT require authentication.

Success: HTTP 200
```json
{"status": "ok", "version": "0.1.0"}
```
Unhealthy: HTTP 503
```json
{"status": "degraded", "detail": "string"}
```

#### GET /v1/openapi.json

Returns the OpenAPI 3.1 schema for this surface.
MUST match the normative schema at `schemas/http-api.schema.json`.

### 10.4 Admin API — /_aevum/v1/

Requires authentication. All responses are JSON.

    GET  /_aevum/v1/complications
    POST /_aevum/v1/complications/{id}/approve
    POST /_aevum/v1/complications/{id}/suspend
    GET  /_aevum/v1/complications/{id}/health
    GET  /_aevum/v1/usage
    GET  /_aevum/v1/federation/peers

Detailed schemas for admin endpoints are defined in the OpenAPI schema
(`schemas/http-api.schema.json`).

### 10.5 Error Format — RFC 9457

All error responses MUST use RFC 9457 Problem Details format with
`Content-Type: application/problem+json`.

Problem type URIs MUST be under `https://aevum.build/problems/`.

Standard problem types:
- `consent-required` — HTTP 403
- `barrier-triggered` — HTTP 403
- `policy-denied` — HTTP 403
- `authentication-required` — HTTP 401
- `validation-error` — HTTP 422
- `replay-not-found` — HTTP 404
- `review-expired` — HTTP 409
- `review-already-resolved` — HTTP 409
- `complication-unavailable` — HTTP 503
- `rate-limited` — HTTP 429

All error responses MUST include `request_id` (from `X-Request-ID`)
and `audit_id` (where an audit entry was created) as extension fields.

### 10.6 Correlation and Tracing

Every request MUST be assigned a correlation ID:
- Read from `X-Request-ID` header if provided by the client.
- Generate a UUID v4 if not provided.

The correlation ID MUST be:
- Echoed in the response `X-Request-ID` header.
- Propagated to the AuditEvent as `correlation_id`.
- Present in all structured log lines for this request.

The `X-Aevum-Key` header value MUST be sanitized from all
OpenTelemetry span attributes. Auth tokens MUST NOT appear in traces.

### 10.7 Rate Limiting

Implementations MAY enforce rate limits. When a rate limit is exceeded:
- Return HTTP 429.
- Include RFC 9457 problem body with type `rate-limited`.
- Include `Retry-After` header with seconds until the limit resets.
- Include `X-RateLimit-Limit`, `X-RateLimit-Remaining`,
  `X-RateLimit-Reset` headers on all rate-limited responses.

### 10.8 Security Headers

Every HTTP response MUST include:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Strict-Transport-Security: max-age=31536000`
- `Content-Security-Policy: default-src 'none'`
