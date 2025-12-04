# FraudDetectionsuingvLLM
Fraud Detection using AI and vLLM

Month 1 — Foundations + AI Layer (Weeks 1–4)
Week 1

Finalize PRD, architecture diagrams, and API contracts.

Define ingestion schemas and lineage schema.

Set up environments for Flask, XGBoost, and vLLM.

Week 2

Build Flask API skeleton for fraud scoring.

Integrate XGBoost model + feature engineering pipeline.

Implement basic vLLM integration (local + batch inference support).

Week 3

Complete vLLM-based inference optimization:

batching

KV-cache reuse

latency benchmarks

Run API + load tests for initial fraud pipeline.

Integrate AI fraud score output into gateway workflow.

Week 4

Harden AI service (error handling, logging, retries).

Deploy Layer-1 AI Fraud Detection service to staging.

Start baseline model evaluation (latency + accuracy).

----

Month 2 — Data Lineage Layer (Weeks 5–8)
Week 5

Build lineage data model:

attribute-level tracking

metadata snapshot storage

Implement lineage capture middleware at gateway.

Week 6

Build lineage service with:

hop-by-hop tracking

field comparison logic

timestamp/IP/header drift detection

Week 7

Implement anomaly detection for manipulation/MITM signals.

Build lineage risk scoring logic.

Integrate lineage outputs with fraud score pipeline.

Week 8

End-to-end integration test:
Client → Gateway → AI Fraud Layer → Lineage Layer → Decision Engine

Create lineage dashboard (basic visualization + logs).

Deploy Layer-2 Lineage Service to staging.

------
Month 3 — Decision Engine + UAT + Rollout (Weeks 9–12)
Week 9

Implement Decision Engine:

fraud_score + lineage_risk_score fusion

Business rules for Approve/Challenge/Decline

Set thresholds + override rules.

Week 10

Performance + security hardening:

P99 latency target <150ms

Load testing at scale

Audit logging, compliance review

Week 11

UAT with Fraud Team, Ops, and Compliance.

Fix feedback + fine-tune lineage & fraud thresholds.

Prepare for controlled rollout.

Week 12

Pilot launch with selected client apps.

Monitor KPIs: fraud detection rate, anomalies, latency.

Stabilize and plan for full rollout.
