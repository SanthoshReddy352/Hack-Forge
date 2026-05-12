# HackForge AI – Unified Project Structure (Repository & System Layout)

This document defines the complete project structure for HackForge AI and aligns every repository, service, and infrastructure component with the following canonical documents:

- Product Requirements Document (PRD) fileciteturn0file2
- Technical Architecture Blueprint fileciteturn0file0
- Microservices API Contract fileciteturn0file1
- Internal Data Models fileciteturn0file3
- Engineering Roadmap fileciteturn0file5
- Mathematical & Algorithmic Definitions fileciteturn0file6
- Cost Estimation Model fileciteturn0file7
- Security & Compliance Deep Dive fileciteturn0file8
- PostgreSQL Schema & Indexing fileciteturn0file9
- Multi-Tenant Isolation Model fileciteturn0file10
- SLA/SLO & Rate Limiting fileciteturn0file11
- Kafka Event Contracts fileciteturn0file12
- Protobuf Definitions fileciteturn0file13
- OpenAPI Schema fileciteturn0file14
- Infrastructure Summary fileciteturn0file15

The structure below is designed for a production-grade, event-driven, multi-tenant distributed system with strict control-plane / compute-plane separation.

---------------------------------------------------------------------

# 1. Repository Strategy

Recommended approach: Polyrepo with shared contracts repository.

Top-Level Repositories:

hackforge-contracts/
hackforge-control-plane/
hackforge-compute-plane/
hackforge-infra/
hackforge-client/
hackforge-ml-engines/

Each repository has a clearly defined architectural responsibility.

---------------------------------------------------------------------

# 2. hackforge-contracts (Single Source of Truth)

Purpose:
- All API definitions
- Protobuf contracts
- Kafka schema registry definitions
- OpenAPI specs

Structure:

hackforge-contracts/
  openapi/
    hackforge-v1.yaml
  protobuf/
    internal/
      experiment.proto
      scheduler.proto
      statistical.proto
      causal.proto
      difficulty.proto
      artifact.proto
  kafka/
    experiment-created.avsc
    experiment-completed.avsc
  shared/
    error-codes.md
    enums.md

This repository enforces:
- Backward-compatible schema evolution
- Versioned API governance
- Deterministic contract boundaries

---------------------------------------------------------------------

# 3. hackforge-control-plane

Implements:
- experiment-service
- validation-service
- scheduler-service
- seed-service
- artifact-service (metadata side)
- monitoring-service

Structure:

hackforge-control-plane/
  cmd/
    api-gateway/
    experiment-service/
    validation-service/
    scheduler-service/
  internal/
    domain/
      experiment/
      tenant/
      artifact/
    application/
      commands/
      queries/
      lifecycle/
    infrastructure/
      db/
      kafka/
      redis/
      security/
    config/
  migrations/
    001_init.sql
    002_partitions.sql
  Dockerfile

Design Pattern:
- Clean Architecture
- Domain layer aligned with Internal Data Models
- Application layer aligned with lifecycle state machine
- Infrastructure layer aligned with PostgreSQL schema & Kafka events

---------------------------------------------------------------------

# 4. hackforge-compute-plane

Implements execution engines triggered via Kafka.

Structure:

hackforge-compute-plane/
  jobs/
    tabular-generator/
    causal-engine/
    difficulty-engine/
    failure-engine/
    multimodal-engine/
  shared/
    rng/
    config-loader/
    artifact-writer/
  entrypoints/
    job-runner/
  Dockerfile

Each job:
- Stateless
- Run-scoped
- Tenant-scoped
- Writes artifacts only to object storage
- Persists reports via control-plane APIs

Compute jobs must not directly mutate metadata tables.

---------------------------------------------------------------------

# 5. hackforge-ml-engines

Contains heavy statistical and ML logic.

Structure:

hackforge-ml-engines/
  statistical/
    distributions/
    ks_test/
    auto_correction/
  causal/
    dag_builder/
    sem_executor/
    intervention/
  difficulty/
    linear_probe/
    entropy/
    mutual_information/
  failure_injection/
    mcar/
    mar/
    mnar/
    adversarial/
    drift/
  multimodal/
    timeseries/
    gan/
    vae/
    nlp/
    graph/

All algorithms must align with Mathematical Definitions document.

This repository should be test-heavy with:
- Determinism tests
- Statistical compliance tests
- Reproducibility checksum tests

---------------------------------------------------------------------

# 6. hackforge-infra

Infrastructure-as-Code (IaC).

Structure:

hackforge-infra/
  terraform/
    network/
    kubernetes/
    kafka/
    postgres/
    redis/
    object-storage/
  helm/
    control-plane/
    compute-plane/
    kafka/
  k8s/
    namespaces/
    rbac/
    network-policies/
  monitoring/
    prometheus/
    grafana/
  ci-cd/
    github-actions/

Must reflect Infrastructure Summary and Security documents.

Includes:
- RLS enforcement configuration
- Namespace-per-tenant templates (enterprise)
- ResourceQuota definitions

---------------------------------------------------------------------

# 7. hackforge-client

User-facing interfaces.

Structure:

hackforge-client/
  web/
    src/
      pages/
      components/
      experiment-builder/
      causal-graph-editor/
  sdk/
    python/
    javascript/
  cli/

Must align strictly with OpenAPI schema.

Client never directly interacts with compute plane.

---------------------------------------------------------------------

# 8. Database Migration Strategy

Single migration stream under control-plane repo.

migrations/
  tenants.sql
  users.sql
  experiments.sql
  experiment_configs.sql
  experiment_runs.sql
  reports.sql
  artifacts.sql

Rules:
- No destructive migrations
- Additive-only schema changes
- Preserve reproducibility constraints

---------------------------------------------------------------------

# 9. Event-Driven Flow Mapping

Experiment creation flow:

Client → API Gateway → experiment-service
  → experiment.created (Kafka)
  → validation-service
  → experiment.validated
  → scheduler-service
  → experiment.started
  → compute-plane job
  → distribution.validation.completed
  → difficulty.calculated
  → experiment.completed
  → artifact.ready

All transitions must occur via Kafka events.

---------------------------------------------------------------------

# 10. Testing Structure

Each repository must include:

/tests/
  unit/
  integration/
  determinism/
  compliance/

Special Test Suites:

1. Reproducibility Suite
   - Same (config_hash, seed) → identical checksum

2. Isolation Suite
   - Cross-tenant access rejection tests

3. Cost Estimation Consistency Suite
   - EstimateResources deterministic validation

4. SLA Latency Benchmarks
   - Status query <200ms p99

---------------------------------------------------------------------

# 11. CI/CD Pipeline Structure

Pipeline Stages:

1. Lint & Static Analysis
2. Unit Tests
3. Contract Compatibility Tests
4. Integration Tests (ephemeral environment)
5. Determinism Regression Tests
6. Docker Build & Sign
7. Image Scan
8. Deploy to Staging
9. Canary Release (Production)

Compute images and control-plane images versioned independently.

---------------------------------------------------------------------

# 12. Environment Configuration Layout

Environment-specific configs:

/config/
  dev.yaml
  staging.yaml
  production.yaml

Must include:
- Kafka brokers
- DB connection
- RLS enforcement flag
- Cost constants (κ_cpu, κ_gpu)
- SLA thresholds

No secrets stored in repository.

---------------------------------------------------------------------

# 13. Folder-Level Isolation Rules

Hard Rules:

1. Compute-plane cannot import control-plane domain layer.
2. Control-plane cannot embed statistical generation logic.
3. Contracts repo cannot contain implementation logic.
4. Infrastructure repo cannot contain business logic.
5. ML engines must remain pure and deterministic.

These preserve architectural separation from Technical Architecture Blueprint.

---------------------------------------------------------------------

# 14. Production Directory Layout (Kubernetes View)

Cluster Namespaces:

- hackforge-control
- hackforge-compute
- hackforge-kafka
- hackforge-monitoring

Enterprise mode:
- hackforge-tenant-{tenant_id}

Node Pools:
- cpu-pool
- gpu-pool

All workloads labeled with:
- tenant_id
- run_id
- experiment_id

---------------------------------------------------------------------

# 15. Governance & Documentation Structure

/docs/
  architecture/
  api/
  security/
  cost/
  sla/
  runbooks/
  incident-response/

Every major feature must update:
- API contracts
- Protobuf
- Event schema
- Migration
- Documentation

---------------------------------------------------------------------

# 16. Strategic Summary

This project structure enforces:

- Deterministic reproducibility via strict separation of config, seed, and execution
- Multi-tenant isolation at repository, service, and namespace level
- Event-driven lifecycle governance
- Cost-aware scheduling boundaries
- Immutable statistical compliance reporting
- SLA-aligned query optimization

The result is a repository and service structure that mirrors the architectural, mathematical, cost, security, and isolation guarantees defined across the HackForge AI documentation set.

This layout should be treated as the canonical implementation blueprint for building HackForge AI from Phase 0 to full enterprise production.



# HackForge AI – Unified Project Structure (With Visual Folder Tree)

This document extends the previously defined project structure and now includes a full visual directory tree representation for implementation clarity.

All structures remain aligned with:
PRD fileciteturn0file2
Technical Architecture fileciteturn0file0
API Contract fileciteturn0file1
Internal Data Models fileciteturn0file3
Engineering Roadmap fileciteturn0file5
Mathematical Definitions fileciteturn0file6
Cost Model fileciteturn0file7
Security Model fileciteturn0file8
PostgreSQL Schema fileciteturn0file9
Multi‑Tenant Isolation fileciteturn0file10
SLA/SLO Model fileciteturn0file11
Kafka Contracts fileciteturn0file12
Protobuf Contracts fileciteturn0file13
OpenAPI Schema fileciteturn0file14
Infrastructure Summary fileciteturn0file15

---

# 1. Top-Level Workspace Structure

hackforge-ai/
├── hackforge-contracts/
├── hackforge-control-plane/
├── hackforge-compute-plane/
├── hackforge-ml-engines/
├── hackforge-infra/
├── hackforge-client/
├── docs/
└── README.md

---

# 2. hackforge-contracts (API & Event Authority)

hackforge-contracts/
├── openapi/
│   └── hackforge-v1.yaml
├── protobuf/
│   └── internal/
│       ├── experiment.proto
│       ├── scheduler.proto
│       ├── statistical.proto
│       ├── causal.proto
│       ├── difficulty.proto
│       ├── artifact.proto
│       └── monitoring.proto
├── kafka/
│   ├── experiment.created.avsc
│   ├── experiment.validated.avsc
│   ├── experiment.started.avsc
│   ├── experiment.completed.avsc
│   ├── artifact.ready.avsc
│   └── cost.reconciled.avsc
└── shared/
├── error-codes.md
├── enums.md
└── versioning-policy.md

---

# 3. hackforge-control-plane

hackforge-control-plane/
├── cmd/
│   ├── api-gateway/
│   ├── experiment-service/
│   ├── validation-service/
│   ├── scheduler-service/
│   ├── seed-service/
│   └── artifact-service/
├── internal/
│   ├── domain/
│   │   ├── experiment/
│   │   ├── tenant/
│   │   ├── artifact/
│   │   ├── reports/
│   │   └── cost/
│   ├── application/
│   │   ├── commands/
│   │   ├── queries/
│   │   ├── lifecycle/
│   │   └── event-handlers/
│   ├── infrastructure/
│   │   ├── db/
│   │   │   ├── repositories/
│   │   │   └── migrations/
│   │   ├── kafka/
│   │   ├── redis/
│   │   ├── security/
│   │   └── observability/
│   └── config/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── determinism/
│   └── isolation/
├── Dockerfile
└── Makefile

---

# 4. hackforge-compute-plane

hackforge-compute-plane/
├── jobs/
│   ├── tabular-generator/
│   ├── causal-engine/
│   ├── difficulty-engine/
│   ├── failure-engine/
│   └── multimodal-engine/
├── shared/
│   ├── rng/
│   ├── config-loader/
│   ├── artifact-writer/
│   └── compliance-checker/
├── entrypoints/
│   └── job-runner/
├── tests/
│   ├── integration/
│   ├── reproducibility/
│   └── performance/
├── Dockerfile
└── Makefile

---

# 5. hackforge-ml-engines

hackforge-ml-engines/
├── statistical/
│   ├── distributions/
│   ├── ks_test/
│   └── auto_correction/
├── causal/
│   ├── dag_builder/
│   ├── sem_executor/
│   └── intervention/
├── difficulty/
│   ├── linear_probe/
│   ├── entropy/
│   └── mutual_information/
├── failure_injection/
│   ├── mcar/
│   ├── mar/
│   ├── mnar/
│   ├── adversarial/
│   └── drift/
├── multimodal/
│   ├── timeseries/
│   ├── gan/
│   ├── vae/
│   ├── nlp/
│   └── graph/
└── tests/
├── statistical/
├── determinism/
└── regression/

---

# 6. hackforge-infra

hackforge-infra/
├── terraform/
│   ├── network/
│   ├── kubernetes/
│   ├── kafka/
│   ├── postgres/
│   ├── redis/
│   └── object-storage/
├── helm/
│   ├── control-plane/
│   ├── compute-plane/
│   └── kafka/
├── k8s/
│   ├── namespaces/
│   ├── rbac/
│   ├── network-policies/
│   └── resource-quotas/
├── monitoring/
│   ├── prometheus/
│   ├── grafana/
│   └── alertmanager/
└── ci-cd/
├── github-actions/
└── pipelines/

---

# 7. hackforge-client

hackforge-client/
├── web/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── experiment-builder/
│   │   ├── causal-graph-editor/
│   │   └── dashboard/
│   └── package.json
├── sdk/
│   ├── python/
│   └── javascript/
└── cli/

---

# 8. docs/

docs/
├── architecture/
├── api/
├── security/
├── cost/
├── sla/
├── runbooks/
└── incident-response/

---

# 9. Visual Separation of Planes

CONTROL PLANE
└── hackforge-control-plane/

COMPUTE PLANE
└── hackforge-compute-plane/

CONTRACT LAYER
└── hackforge-contracts/

INFRASTRUCTURE LAYER
└── hackforge-infra/

ML CORE ENGINE LAYER
└── hackforge-ml-engines/

CLIENT LAYER
└── hackforge-client/

---

This visual structure ensures strict architectural boundary enforcement, deterministic execution isolation, contract-first development, and infrastructure-level tenant separation as defined across the HackForge AI documentation set.
