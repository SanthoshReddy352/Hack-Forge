# HackForge AI – Step-by-Step Implementation Guide (Repository-Aligned Execution Plan)

This document provides a strict, implementation-first execution sequence for building HackForge AI according to the defined repository and system layout. It enforces:

- Contract-first development
- Control-plane / compute-plane separation
- Deterministic reproducibility invariants
- Tenant isolation guarantees
- Event-driven lifecycle execution

The sequence below must be followed in order. Do not skip layers.

------------------------------------------------------------

# PHASE 0 – Workspace & Repository Initialization

## 0.1 Create Top-Level Workspace

Create the root directory:

hackforge-ai/

Inside it, initialize repositories (polyrepo or mono-workspace layout):

- hackforge-contracts/
- hackforge-control-plane/
- hackforge-compute-plane/
- hackforge-ml-engines/
- hackforge-infra/
- hackforge-client/
- docs/

No business logic at this stage.

------------------------------------------------------------

# PHASE 1 – Contracts Layer (MUST BE FIRST)
Repository: hackforge-contracts/

This repository is the single source of truth for:
- OpenAPI schema
- Protobuf internal contracts
- Kafka event schemas
- Shared enums and error codes

Nothing else should be implemented before this layer compiles.

## 1.1 Folder Structure

hackforge-contracts/
  openapi/
  protobuf/internal/
  kafka/
  shared/

## 1.2 Implement OpenAPI (External Control Plane)

File:
openapi/hackforge-v1.yaml

Define:
- POST /experiments
- GET /experiments/{id}
- GET /experiments/{id}/status
- DELETE /experiments/{id}
- GET /artifacts/{id}/download
- POST /experiments/estimate

Include:
- JWT bearer security
- X-Tenant-ID header
- Idempotency-Key header
- Standard response envelope

Validate schema using an OpenAPI validator.

## 1.3 Implement Protobuf Contracts (Internal gRPC)

Define:
- ExperimentService
- ValidationService
- SchedulerService
- StatisticalEngine
- CausalEngine
- DifficultyEngine
- FailureEngine
- ArtifactService

Rules:
- All requests include TenantContext
- Additive-only field evolution
- No implementation logic here

Compile and generate stubs.

## 1.4 Define Kafka Event Schemas

Create Avro/JSON schemas for:
- experiment.created
- experiment.validated
- experiment.started
- experiment.completed
- artifact.ready
- resource.estimated

Define canonical event envelope.

This completes Phase 1.

------------------------------------------------------------

# PHASE 2 – Control Plane Foundation
Repository: hackforge-control-plane/

Implement metadata, lifecycle orchestration, and scheduling logic.

## 2.1 Project Structure

cmd/
  api-gateway/
  experiment-service/
  validation-service/
  scheduler-service/
  seed-service/
  artifact-service/

internal/
  domain/
  application/
  infrastructure/

## 2.2 Database Setup (First in Control Plane)

Implement migrations for:
- tenants
- users
- experiments (partitioned)
- experiment_configs (immutable)
- experiment_runs
- resource_allocations
- distribution_reports
- dataset_artifacts

Add:
- Unique (tenant_id, config_hash, seed, version)
- RLS policy template
- Required indexes

Run migrations locally.

## 2.3 Implement Experiment Creation Flow

Flow:

Client → POST /experiments
→ Compute config_hash
→ Persist Experiment + ExperimentConfig
→ Publish experiment.created event
→ Return 202

No compute logic yet.

## 2.4 Implement Validation Service

Subscribe to experiment.created
→ Validate schema
→ Validate DAG acyclicity
→ Publish experiment.validated or experiment.rejected

## 2.5 Implement Scheduler Service

Subscribe to experiment.validated
→ EstimateResources (pure function)
→ Enforce quota
→ Allocate run_id
→ Publish experiment.started

## 2.6 Implement Seed Service

Generate deterministic seed
Persist seed
Expose via API (role-restricted)

This completes metadata and orchestration spine.

------------------------------------------------------------

# PHASE 3 – ML Engines (Pure Deterministic Core)
Repository: hackforge-ml-engines/

This repository contains no infrastructure.
Only mathematical logic.

## 3.1 Implement Deterministic RNG Wrapper

Input:
- config_hash
- seed
- service_namespace

Output:
- PRNG instance

Add determinism tests.

## 3.2 Implement Statistical Engine

Modules:
- Normal distribution
- Poisson
- Lognormal
- Pareto
- Uniform
- KS-test validation
- Auto-correction loop

Add:
- Statistical compliance tests

## 3.3 Implement Causal Engine

- DAG builder
- Topological sort
- SEM executor
- Intervention simulation

## 3.4 Implement Difficulty Engine

- Linear separability metric
- Entropy
- Mutual information
- Overfitting probe

## 3.5 Implement Failure Injection

- MCAR
- MAR
- MNAR
- Adversarial noise
- Concept drift

All functions must be deterministic.

------------------------------------------------------------

# PHASE 4 – Compute Plane Integration
Repository: hackforge-compute-plane/

Compute plane consumes events and executes jobs.

## 4.1 Job Runner Skeleton

Subscribe to experiment.started
Load ExperimentConfig
Create run context

## 4.2 Integrate Statistical Engine

Call ML engine modules
Generate independent features
Validate distributions
Persist DistributionReport via control-plane API

## 4.3 Integrate Causal & Difficulty Engines

Simulate dependencies
Compute difficulty
Persist DifficultyReport

## 4.4 Artifact Packaging

Write dataset to object storage
Compute SHA256 checksum
Call ArtifactService
Publish artifact.ready

Compute jobs must not write directly to metadata DB.

------------------------------------------------------------

# PHASE 5 – Infrastructure Layer
Repository: hackforge-infra/

Implement Infrastructure-as-Code.

## 5.1 Terraform Modules

- VPC / network
- Kubernetes cluster
- PostgreSQL HA
- Kafka cluster
- Redis
- Object storage

## 5.2 Kubernetes Config

- Control-plane namespace
- Compute-plane namespace
- ResourceQuota
- NetworkPolicy

## 5.3 Observability

- Prometheus
- Grafana
- ELK
- OpenTelemetry

------------------------------------------------------------

# PHASE 6 – SLA, Rate Limiting & Cost Governance

Implement:
- Token bucket per tenant (Redis-backed)
- Concurrency cap enforcement
- Cost estimation endpoint
- Post-run cost reconciliation

Add metrics:
- api_latency_p99
- compliance_score
- reproducibility_failures

------------------------------------------------------------

# PHASE 7 – Client Layer
Repository: hackforge-client/

Implement:
- Experiment builder UI
- Causal graph editor
- Progress WebSocket listener
- Artifact download

Client must consume OpenAPI only.

------------------------------------------------------------

# PHASE 8 – Testing & Hardening

Implement test suites:

1. Determinism Suite
2. Isolation Suite
3. Cost Consistency Suite
4. SLA Latency Benchmarks
5. Event Replay Safety Tests

------------------------------------------------------------

# IMPLEMENTATION ORDER SUMMARY

1. Contracts repository
2. Control-plane skeleton + DB migrations
3. Minimal experiment creation flow
4. Validation + scheduling
5. ML engine deterministic core
6. Compute plane integration
7. Artifact storage
8. Infrastructure provisioning
9. SLA + rate limiting
10. Client UI

Never invert this order.

------------------------------------------------------------

# Final Implementation Rule

Contracts → Metadata → Lifecycle → Determinism → Compute → Infrastructure → Governance → UI

If this order is followed strictly, HackForge AI will evolve into a deterministic, event-driven, tenant-isolated experimental simulation engine with structural integrity preserved across all layers.

