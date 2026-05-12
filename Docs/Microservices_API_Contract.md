# Microservices API Contract Specification

## Project: HackForge AI – Intelligent Experimental Data Engine

---

## 1. API Design Standards

Protocol:
- External APIs: REST over HTTPS
- Internal Service Communication: gRPC
- Event Streaming: Kafka / NATS

Data Format:
- JSON (external REST)
- Protobuf (internal gRPC)

Authentication:
- OAuth2 + JWT
- Tenant ID required in all requests

All dataset generation operations are asynchronous.

---

## 2. Global Conventions

Base URL:
/api/v1

Headers:
Authorization: Bearer <JWT>
X-Tenant-ID: <tenant_id>
X-Request-ID: <uuid>

Standard Response Envelope:

{
  "status": "success | error",
  "data": {},
  "error": {
    "code": "STRING_CODE",
    "message": "Human readable message"
  }
}

---

## 3. Experiment Orchestrator Service API

Service Name: experiment-service

### 3.1 Create Experiment

POST /experiments

Request Body:
{
  "name": "Fraud Detection Dataset",
  "description": "High imbalance financial dataset",
  "schema": {...},
  "causal_graph": {...},
  "distribution_config": {...},
  "difficulty_level": "intermediate",
  "failure_injection": {...},
  "output_format": "parquet"
}

Response:
{
  "experiment_id": "exp_123456",
  "status": "queued"
}

---

### 3.2 Get Experiment Status

GET /experiments/{experiment_id}

Response:
{
  "experiment_id": "exp_123456",
  "status": "running | completed | failed",
  "progress_percentage": 72,
  "estimated_time_remaining_seconds": 140
}

---

### 3.3 Cancel Experiment

DELETE /experiments/{experiment_id}

---

## 4. Configuration Validator Service API

Service Name: validation-service

Internal gRPC Methods:

ValidateSchema(SchemaConfig) → ValidationResult
ValidateCausalGraph(GraphConfig) → ValidationResult
ValidateDistribution(DistributionConfig) → ValidationResult
ValidateResourceFeasibility(ResourceEstimateRequest) → ResourceEstimateResponse

ValidationResult:
{
  valid: bool,
  errors: [string],
  warnings: [string]
}

---

## 5. Resource Scheduler Service API

Service Name: scheduler-service

Internal gRPC Methods:

EstimateResources(ExperimentConfig) → ResourceEstimate
AllocateCompute(ExperimentID) → AllocationResult
ReleaseCompute(ExperimentID) → Ack

ResourceEstimate:
{
  cpu_cores: int,
  memory_gb: float,
  gpu_required: bool,
  estimated_cost_usd: float,
  estimated_runtime_seconds: int
}

---

## 6. Seed & Determinism Manager API

Service Name: seed-service

POST /seeds

Request:
{
  "experiment_id": "exp_123456"
}

Response:
{
  "seed": 938475923
}

GET /seeds/{experiment_id}

---

## 7. Statistical Generation Service API

Service Name: statistical-engine

Internal gRPC:

GenerateIndependentFeatures(FeatureSpec, Seed) → FeatureBatch
ValidateDistribution(GeneratedData) → DistributionReport

DistributionReport:
{
  ks_statistic: float,
  p_value: float,
  distribution_match_score: float
}

---

## 8. Causal Graph Engine API

Service Name: causal-engine

Internal gRPC:

BuildGraph(GraphConfig) → GraphID
SimulateDependentVariables(GraphID, FeatureBatch) → FeatureBatch
RunIntervention(GraphID, InterventionConfig) → CounterfactualDataset

---

## 9. Difficulty Scoring Engine API

Service Name: difficulty-engine

Internal gRPC:

ComputeDifficulty(DatasetRef) → DifficultyReport

DifficultyReport:
{
  linear_separability_score: float,
  noise_ratio: float,
  entropy_score: float,
  overfitting_risk_score: float,
  final_difficulty_index: float
}

---

## 10. Failure Injection Engine API

Service Name: failure-engine

Internal gRPC:

InjectMissingness(DatasetRef, MissingConfig) → DatasetRef
InjectAdversarialNoise(DatasetRef, NoiseConfig) → DatasetRef
InjectConceptDrift(DatasetRef, DriftConfig) → DatasetRef
InjectCovariateShift(DatasetRef, ShiftConfig) → DatasetRef
InjectLeakage(DatasetRef, LeakageConfig) → DatasetRef

---

## 11. Multi-Modal Generator APIs

Service Name: multimodal-engine

TabularGenerate(TabularConfig) → DatasetRef
TimeSeriesGenerate(TimeSeriesConfig) → DatasetRef
ImageTensorGenerate(ImageConfig) → DatasetRef
TextCorpusGenerate(TextConfig) → DatasetRef
GraphGenerate(GraphConfig) → DatasetRef

---

## 12. Artifact Service API

Service Name: artifact-service

GET /artifacts/{experiment_id}/download

Response:
{
  "signed_url": "https://storage/..."
}

GET /artifacts/{experiment_id}/metadata

---

## 13. WebSocket Event Stream

Endpoint:
/ws/experiments/{experiment_id}

Events:

{
  "event_type": "stage_update",
  "stage": "difficulty_evaluation",
  "progress": 60
}

{
  "event_type": "completed",
  "artifact_url": "signed_url"
}

---

## 14. Observability APIs

Service Name: monitoring-service

GET /metrics/experiment/{experiment_id}

Returns:
- compute_time
- compliance_score
- failure_injection_accuracy

---

## 15. Error Codes

Common Error Codes:

EXP_001 → Invalid configuration
EXP_002 → Resource allocation failure
EXP_003 → Statistical validation failed
EXP_004 → DAG cycle detected
EXP_005 → Seed conflict
EXP_006 → Artifact generation failure

---

## 16. Idempotency & Concurrency Rules

- POST /experiments must support Idempotency-Key header
- Same key returns same experiment_id
- Maximum concurrent experiments per tenant configurable

---

## 17. Versioning Strategy

- URI versioning (/api/v1)
- Backward compatibility maintained for minor updates
- Major breaking changes require /v2 namespace

---

## 18. Security & Compliance Requirements

- All endpoints enforce JWT validation
- Tenant isolation enforced at database query level
- Signed artifact URLs expire within configurable TTL
- All internal service-to-service communication mTLS secured

---

## 19. Event Topics (Kafka/NATS)

experiment.created
experiment.validated
experiment.scheduled
experiment.started
experiment.failed
experiment.completed
artifact.ready

---

## 20. Summary

This API contract specification defines strict service boundaries, request/response contracts, asynchronous lifecycle management, deterministic seed handling, validation enforcement, and compute orchestration mechanisms.

The microservices architecture ensures modular scalability while maintaining deterministic experimental reproducibility and statistical guarantees.

