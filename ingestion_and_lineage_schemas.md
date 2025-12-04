# Ingestion and Lineage Schemas

This document defines canonical schemas for ingesting data into the fraud detection platform and capturing lineage metadata. Schemas are designed to align with the API contracts (see `api_contracts.md` and `openapi.yaml`) while remaining storage- and pipeline-agnostic.

## 1. Transaction Ingestion Schema
Primary real-time input for scoring and feature generation.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transaction_id` | string | Yes | Unique identifier for the transaction (idempotency key across systems). |
| `user_id` | string | Yes | End-customer identifier. |
| `amount` | number | Yes | Decimal currency amount. |
| `currency` | string | Yes | ISO 4217 currency code (e.g., `USD`). |
| `merchant_id` | string | No | Merchant or counterparty identifier. |
| `timestamp` | string (ISO 8601) | Yes | Event time of the transaction. |
| `channel` | string | No | Interaction channel such as `web`, `mobile`, or `in_person`. |
| `ip_address` | string | No | IPv4/IPv6 address captured at authorization time. |
| `device_id` | string | No | Device fingerprint or hardware identifier. |
| `location` | object | No | `{ "lat": number, "lon": number }` GPS coordinates or geocoder output. |
| `metadata` | object | No | Arbitrary feature map for partner- or experiment-specific attributes. |

**Example payload**
```json
{
  "transaction_id": "txn_123",
  "user_id": "user_42",
  "amount": 199.99,
  "currency": "USD",
  "timestamp": "2025-01-01T12:00:00Z",
  "channel": "web",
  "ip_address": "203.0.113.10",
  "location": { "lat": 37.7749, "lon": -122.4194 },
  "metadata": { "promo_code": "NY25" }
}
```

## 2. Feature/Label Ingestion Schema
Used to push derived features, feedback labels, and outcomes for training, monitoring, and model recalibration.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transaction_id` | string | Yes | Ties the record back to the original transaction. |
| `label` | string | No | Outcome label such as `fraudulent`, `legit`, or `disputed`. When absent, the record is treated as an unlabeled feature update. |
| `features` | object | No | Map of derived features (e.g., `chargeback`, `dispute_reason`, `velocity_score`). |
| `occurred_at` | string (ISO 8601) | Yes | Time the feedback or feature observation occurred. |
| `source` | string | No | Optional emitter identifier (e.g., `chargeback_system`, `support_ui`, `etl_job`). |
| `trace_id` | string (UUID) | No | Correlation ID to connect to upstream request logs. |

**Example payload**
```json
{
  "records": [
    {
      "transaction_id": "txn_123",
      "label": "fraudulent",
      "features": { "chargeback": true, "dispute_reason": "card_not_present" },
      "occurred_at": "2025-01-02T09:00:00Z",
      "source": "chargeback_system",
      "trace_id": "3c5fd784-3e44-4fb6-bc8d-59f53e407bfb"
    }
  ]
}
```

## 3. Lineage Schema
Captures how data moves through the fraud detection stack for reproducibility, audits, and debugging. Each lineage record represents the relationship between a data asset and its parent.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `lineage_id` | string (UUID) | Yes | Unique identifier for the lineage event. |
| `asset_name` | string | Yes | Logical asset name (e.g., `feature_store.transactions`, `model_outputs.scores`). |
| `asset_type` | string | Yes | `dataset`, `feature_table`, `model_artifact`, or `inference_output`. |
| `parent_assets` | array[string] | Yes | Upstream asset names this asset depends on. |
| `transformation` | string | No | Human-readable description of the transform (e.g., `join transactions with device graph`). |
| `code_ref` | object | No | `{ "repo": "git@...", "commit": "abc123", "path": "pipelines/feature_job.py" }`. |
| `schedule` | object | No | `{ "cron": "0 * * * *", "timezone": "UTC" }` or `{ "trigger": "event" }`. |
| `producer` | object | Yes | `{ "service": "fraud-api", "version": "1.2.0", "host": "api-1" }`. |
| `run_id` | string | No | Execution identifier from orchestration (e.g., Airflow DAG run ID). |
| `created_at` | string (ISO 8601) | Yes | Time the lineage event was emitted. |
| `trace_id` | string (UUID) | No | Correlates with API/gateway traces for debugging. |

**Example lineage record**
```json
{
  "lineage_id": "edb2d3d3-4a5b-4c2c-9b9d-2dba7e3f4c2a",
  "asset_name": "feature_store.velocity_features",
  "asset_type": "feature_table",
  "parent_assets": ["raw.transactions", "device_graph.edges"],
  "transformation": "aggregate txn counts over 24h and join device graph",
  "code_ref": {
    "repo": "git@github.com:company/fraud-pipelines.git",
    "commit": "a1b2c3d",
    "path": "pipelines/velocity_features.py"
  },
  "schedule": { "cron": "0 * * * *", "timezone": "UTC" },
  "producer": { "service": "feature-pipeline", "version": "0.9.0", "host": "airflow-worker-1" },
  "run_id": "airflow_dag_run_2025-01-02T09:00:00Z",
  "created_at": "2025-01-02T09:01:00Z",
  "trace_id": "7c4f9d1a-29b4-4f2e-9d6f-1a4b2c3d4e5f"
}
```

## 4. Validation and Governance Notes
- Schemas map directly to `openapi.yaml` components `TransactionPayload` and `FeaturesIngestRecord` for consistency across API, storage, and ML pipelines.
- Enforce required fields via JSON Schema or Pydantic models at ingestion boundaries; reject or dead-letter non-conformant records.
- Attach lineage events alongside ingestion jobs to ensure reproducibility of feature generation and scoring outputs.
- Prefer UUIDs for identifiers (`lineage_id`, `trace_id`) to enable cross-system correlation.
