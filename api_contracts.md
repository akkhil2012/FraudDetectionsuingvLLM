# Fraud Detection API Contracts

This document outlines the HTTP interfaces for the Fraud Detection service built with Flask, XGBoost, and vLLM. The contracts focus on predictable request/response envelopes so clients can integrate consistently across real-time and batch scoring flows.

## Conventions
- **Base URL**: `/api`
- **Versioning**: All endpoints include a version prefix (e.g., `/v1`).
- **Authentication**: Bearer token in the `Authorization: Bearer <token>` header. Calls without valid tokens return `401`.
- **Idempotency**: Mutating endpoints accept `Idempotency-Key` headers to ensure safe retries.
- **Content type**: `application/json` for requests and responses. Binary payloads (e.g., CSV uploads) should use `multipart/form-data`.
- **Traceability**: The service echoes a `trace_id` in responses and supports an optional `X-Request-ID` header.

### Standard error envelope
```json
{
  "error": {
    "code": "string",         // machine readable
    "message": "string",      // human readable
    "details": { "field": "explanation" }
  },
  "trace_id": "uuid"
}
```

## Core resource schemas
### Transaction payload
Common fields used by scoring endpoints. Additional features may be supplied in `metadata`.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `transaction_id` | string | Yes | Unique per transaction. |
| `user_id` | string | Yes | End-customer identifier. |
| `amount` | number | Yes | Decimal value in the transaction currency. |
| `currency` | string | Yes | ISO 4217 code. |
| `merchant_id` | string | No | Merchant or counterparty identifier. |
| `timestamp` | string (ISO 8601) | Yes | Event time. |
| `channel` | string | No | e.g., `web`, `mobile`, `in_person`. |
| `ip_address` | string | No | IPv4/IPv6. |
| `device_id` | string | No | Device fingerprint. |
| `location` | object | No | `{ "lat": number, "lon": number }`. |
| `metadata` | object | No | Arbitrary feature map. |

### Score response payload
| Field | Type | Notes |
| --- | --- | --- |
| `transaction_id` | string | Mirrors input. |
| `score` | number | 0–1 probability of fraud. |
| `risk_label` | string | `low`, `medium`, or `high` derived from policy thresholds. |
| `reasons` | array[string] | Ordered list of model or rule explanations. |
| `model_version` | string | Semantic version or commit hash. |
| `latency_ms` | number | End-to-end processing time. |
| `trace_id` | string | Request trace identifier. |

## Endpoints

### POST /v1/fraud/score
Real-time scoring for a single transaction.

**Request**
```http
POST /api/v1/fraud/score
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: <uuid>
```
```json
{
  "transaction": { /* Transaction payload */ },
  "features": { "browser_language": "en-US" },   // optional derived features
  "explain": true                                    // request explanations
}
```

**Response 200**
```json
{
  "result": { /* Score response payload */ }
}
```

**Response codes**
- `200 OK` — Score computed.
- `400 Bad Request` — Validation failure.
- `401 Unauthorized` — Missing/invalid token.
- `429 Too Many Requests` — Rate limited.
- `500 Internal Server Error` — Unexpected failure.

### POST /v1/fraud/score/batch
Submit multiple transactions for synchronous or asynchronous scoring.

**Request**
```http
POST /api/v1/fraud/score/batch?async=true
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: <uuid>
```
```json
{
  "transactions": [ { /* Transaction payload */ }, ... ],
  "explain": false,
  "max_latency_ms": 2000      // optional SLA hint; if exceeded, request may fall back to async
}
```

**Response 200 (synchronous)**
```json
{
  "mode": "sync",
  "results": [ { /* Score response payload */ }, ... ],
  "trace_id": "uuid"
}
```

**Response 202 (asynchronous)**
```json
{
  "mode": "async",
  "job_id": "uuid",
  "estimated_completion_ms": 1200,
  "trace_id": "uuid"
}
```

**Response codes**
- `200 OK` — Batch completed synchronously.
- `202 Accepted` — Batch accepted for async processing.
- `400/401/429/500` — Standard error envelope.

### GET /v1/fraud/jobs/{job_id}
Fetch status and results for an async batch job.

**Request**
```http
GET /api/v1/fraud/jobs/{job_id}
Authorization: Bearer <token>
```

**Response 200**
```json
{
  "job_id": "uuid",
  "status": "pending | running | succeeded | failed",
  "submitted_at": "ISO-8601",
  "completed_at": "ISO-8601",
  "results": [ { /* Score response payload */ } ],   // present when status=succeeded
  "error": { "code": "string", "message": "string" }, // present when status=failed
  "trace_id": "uuid"
}
```

**Response codes**
- `200 OK` — Job found.
- `404 Not Found` — Unknown job.
- `401/429/500` — Standard error envelope.

### POST /v1/features/ingest
Ingest feature updates or labeled outcomes for downstream training and monitoring.

**Request**
```http
POST /api/v1/features/ingest
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: <uuid>
```
```json
{
  "records": [
    {
      "transaction_id": "txn_123",
      "label": "fraudulent",          // optional; when absent, treated as unlabeled feature update
      "features": { "chargeback": true, "dispute_reason": "card_not_present" },
      "occurred_at": "ISO-8601"
    }
  ]
}
```

**Response 202**
```json
{
  "ingested": 1,
  "failed": 0,
  "trace_id": "uuid"
}
```

**Response codes**
- `202 Accepted` — Records accepted for downstream processing.
- `400/401/413/429/500` — Standard error envelope.

### POST /v1/admin/models/refresh
Trigger model refresh or warm-start of the vLLM runtime.

**Request**
```http
POST /api/v1/admin/models/refresh
Authorization: Bearer <token>
Content-Type: application/json
```
```json
{
  "model": "fraud-v1",       // optional specific model; defaults to active model
  "warm": true                 // warms KV cache for hot paths
}
```

**Response 202**
```json
{
  "model": "fraud-v1",
  "action": "refresh_requested",
  "trace_id": "uuid"
}
```

**Response codes**
- `202 Accepted` — Refresh initiated.
- `401/403` — Unauthorized or forbidden (admin-only).
- `429/500` — Standard error envelope.

### GET /v1/health
Lightweight health probe for orchestration and load balancers.

**Request**
```http
GET /api/v1/health
```

**Response 200**
```json
{
  "status": "ok",
  "version": "2025-01-01",
  "uptime_s": 1234
}
```

## Rate limiting
Clients should expect rate limiting at the API gateway. Standard headers are returned when throttled:
- `Retry-After`
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`

## Observability
- `trace_id` is returned on all responses and should be propagated in client logs.
- Request/response bodies may be redacted in logs based on data-classification policy.
- Metrics emitted per endpoint: request count, latency (p50/p95/p99), error rate, model version, and queue depth for async jobs.
