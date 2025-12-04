# FraudDetectionsuingvLLM
Fraud Detection using AI and vLLM

## Environment setup

Use Python 3.10–3.12 (tested locally with 3.11) when creating the virtual environment. Newer runtimes such as Python 3.14 will trigger build-resolution errors from upstream dependencies (e.g., pip trying to pull the unavailable `numpy==2.0.0rc1`). Create a virtual environment and install the stack-specific dependencies listed below. Each requirements file is scoped to the main components we will build in Month 1.

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

If you need to stick with a system-wide `python3`, confirm it resolves to 3.12 or below (e.g., `python3 --version`). Install the desired environment:

- Flask service: `pip install -r requirements-flask.txt`
- XGBoost modeling: `pip install -r requirements-xgboost.txt`
- vLLM inference: `pip install -r requirements-vllm.txt`

Notes:
- The vLLM environment assumes a CUDA-ready host when running GPU-backed inference; install the matching CUDA-enabled PyTorch wheels if needed.
- You can safely install multiple files into the same virtual environment if you want an all-in-one stack for local development.

## Build steps

Follow these steps to build and run the project locally:

1. **Create and activate a virtual environment** (recommended to keep dependencies isolated):
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

2. **Install dependencies** for the component you want to work on (or install all three if you need the full stack):
   ```bash
   pip install -r requirements-flask.txt
   pip install -r requirements-xgboost.txt
   pip install -r requirements-vllm.txt
   ```

3. **Validate the environment** by running a quick import check for key packages (helps confirm GPU-enabled installs):
   ```bash
   python - <<'PY'
   import flask, xgboost
   import torch
   print('Flask version:', flask.__version__)
   print('XGBoost version:', xgboost.__version__)
   print('Torch CUDA available:', torch.cuda.is_available())
   PY
   ```

4. **(Optional) Run the Flask skeleton** after installing `requirements-flask.txt`:
   ```bash
   export FLASK_APP=app.py  # update if your entrypoint differs
   flask run --host=0.0.0.0 --port=5000
   ```

These steps ensure dependencies are installed and the base service can start locally before layering on modeling or vLLM-specific workflows.

Month 1 — Foundations + AI Layer (Weeks 1–4)
Week 1

Finalize PRD, architecture diagrams, and API contracts.

Define ingestion schemas and lineage schema.

- [x] Canonical ingestion schemas and lineage schema documented in `ingestion_and_lineage_schemas.md`.

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
