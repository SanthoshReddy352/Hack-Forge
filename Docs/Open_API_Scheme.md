# HackForge AI – OpenAPI Schema (Expanded & Cross-Integrated)

This document defines the complete OpenAPI 3.0 specification outline for HackForge AI external REST APIs. It is tightly aligned with:

- PRD (deterministic reproducibility, statistical compliance ≥95%) fileciteturn0file2
- Technical Architecture Blueprint (control plane vs compute plane separation) fileciteturn0file0
- Microservices API Contract (async lifecycle & tenant headers) fileciteturn0file1
- Internal Data Models (Experiment, ExperimentRun, DatasetArtifact, ResourceAllocation) fileciteturn0file3
- Engineering Roadmap (phase-aligned lifecycle & production hardening) fileciteturn0file5
- Mathematical Definitions (config_hash + seed invariants) fileciteturn0file6
- Cost Estimation Model (scheduler EstimateResources integration) fileciteturn0file7
- Security & Compliance (JWT, RLS, signed artifacts) fileciteturn0file8
- PostgreSQL Schema & Indexing (immutability & uniqueness constraints) fileciteturn0file9
- Multi-Tenant Isolation (tenant-scoped boundaries) fileciteturn0file10
- SLA/SLO & Rate Limiting (quota & concurrency enforcement) fileciteturn0file11
- Kafka Event Contracts (event-driven lifecycle) fileciteturn0file12
- Protobuf Internal Contracts (control ↔ compute typing) fileciteturn0file13

The OpenAPI specification governs the external control-plane interface. All compute-plane communication remains gRPC-based and internal.

---------------------------------------------------------------------

# 1. OpenAPI Metadata

openapi: 3.0.3

info:
  title: HackForge AI API
  version: v1
  description: |
    Deterministic synthetic experimental data engine.
    All dataset generations are asynchronous.

servers:
  - url: https://api.hackforge.ai/api/v1

---------------------------------------------------------------------

# 2. Global Security & Headers

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []

Global Required Headers:

- Authorization: Bearer <JWT>
- X-Tenant-ID: <UUID>
- X-Request-ID: <UUID>
- Idempotency-Key: <STRING> (required for POST /experiments)

Tenant header must match JWT tenant claim.

---------------------------------------------------------------------

# 3. Standard Response Envelope

components:
  schemas:
    ApiResponse:
      type: object
      properties:
        status:
          type: string
          enum: [success, error]
        data:
          type: object
        error:
          $ref: '#/components/schemas/ErrorObject'

    ErrorObject:
      type: object
      properties:
        code:
          type: string
        message:
          type: string

All endpoints return ApiResponse.

---------------------------------------------------------------------

# 4. Experiment Lifecycle APIs

## 4.1 Create Experiment

POST /experiments

summary: Create new experiment (asynchronous)

requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/ExperimentCreateRequest'

responses:
  '202':
    description: Experiment accepted
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ExperimentCreateResponse'

Determinism Invariant:
config_hash computed server-side and stored immutably.

---

## 4.2 Get Experiment

GET /experiments/{experiment_id}

Returns metadata snapshot including:
- status
- difficulty_level
- seed (role-restricted)
- version

---

## 4.3 Get Experiment Status

GET /experiments/{experiment_id}/status

Returns:
- status
- progress_percentage
- estimated_time_remaining_seconds

Latency SLO: <200ms (p99)

---

## 4.4 Cancel Experiment

DELETE /experiments/{experiment_id}

Only allowed in:
- draft
- validated
- queued
- running

---------------------------------------------------------------------

# 5. Artifact APIs

## 5.1 Get Artifact Metadata

GET /artifacts/{experiment_id}/metadata

Returns:
- artifact_id
- checksum_sha256
- size_bytes
- format
- compliance_score

---

## 5.2 Generate Signed Download URL

GET /artifacts/{experiment_id}/download

Returns:
- signed_url
- expiry_timestamp

Security:
Signed URL TTL configurable per plan.

Artifact exposed only if compliance_score ≥ 0.95.

---------------------------------------------------------------------

# 6. Metrics & Observability APIs

GET /metrics/experiment/{experiment_id}

Returns:
- compute_time_seconds
- cpu_hours
- gpu_hours
- storage_gb
- compliance_score
- reproducibility_status

Used by enterprise dashboards.

---------------------------------------------------------------------

# 7. Cost Estimation Endpoint

## 7.1 Estimate Resources (Pre-Run)

POST /experiments/estimate

Purpose:
Expose scheduler EstimateResources logic externally without scheduling.

Request:
- row_count
- feature_count
- modal_type
- difficulty_level

Response:
- cpu_cores
- memory_gb
- gpu_required
- estimated_cost_usd
- estimated_runtime_seconds

Estimate must be pure function of config.

---------------------------------------------------------------------

# 8. WebSocket Interface

/ws/experiments/{experiment_id}

Event Types:
- stage_update
- compliance_update
- completed
- failed

Used for real-time progress tracking.

---------------------------------------------------------------------

# 9. Component Schemas

## 9.1 ExperimentCreateRequest

properties:
  name: string
  description: string
  schema_config: object
  distribution_config: object
  causal_graph: object
  failure_config: object
  difficulty_config: object
  output_config: object
  row_count: integer
  feature_count: integer

All configs serialized and hashed server-side.

---

## 9.2 ExperimentCreateResponse

properties:
  experiment_id: string
  status: string
  config_hash: string

---

## 9.3 ExperimentStatusResponse

properties:
  experiment_id: string
  status: string
  progress_percentage: integer
  estimated_time_remaining_seconds: integer

---

## 9.4 ArtifactMetadata

properties:
  artifact_id: string
  checksum_sha256: string
  size_bytes: integer
  format: string
  compliance_score: number
  config_hash: string
  seed: integer

---------------------------------------------------------------------

# 10. Error Codes (Extended)

EXP_001 – Invalid configuration
EXP_002 – Resource allocation failure
EXP_003 – Statistical validation failed
EXP_004 – DAG cycle detected
EXP_005 – Seed conflict
EXP_006 – Artifact generation failure
EXP_429 – Concurrency limit exceeded
EXP_430 – Quota exceeded
EXP_401 – Unauthorized
EXP_403 – Tenant mismatch

---------------------------------------------------------------------

# 11. Versioning Strategy

- URI versioning (/api/v1)
- Additive field evolution only
- Breaking changes require /v2
- Deprecation headers for sunset announcements

---------------------------------------------------------------------

# 12. Deterministic & Isolation Constraints

Invariant 1:
(config_hash, seed, version) uniquely define dataset semantics.

Invariant 2:
No endpoint allows mutation of ExperimentConfig after validation.

Invariant 3:
All operations must include X-Tenant-ID.

Invariant 4:
All artifact downloads validated against tenant scope.

Invariant 5:
Cost estimation does not mutate scheduler state.

---------------------------------------------------------------------

# 13. SLA & Rate Limiting Exposure

Headers Returned:

X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset

429 responses include Retry-After header.

Concurrency violations return EXP_429.
Quota violations return EXP_430.

---------------------------------------------------------------------

# 14. Strategic Outcome

This OpenAPI schema formalizes the external contract of HackForge AI’s control plane while preserving:

- Deterministic reproducibility guarantees
- Immutable experiment configuration
- Strict tenant isolation
- Cost-aware scheduling transparency
- Statistical compliance enforcement
- SLA-aligned lifecycle management

The REST interface is therefore a thin, strongly governed facade over a deterministic, event-driven, gRPC-backed experimental simulation engine.

