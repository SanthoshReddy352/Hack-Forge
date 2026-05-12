# 1. Internal Data Models and Schema Definitions

## 1.1 Core Domain Aggregates

### Tenant
- tenant_id (UUID, PK)
- name
- plan_type (free, pro, enterprise)
- quota_config (JSONB)
- created_at
- updated_at

### User
- user_id (UUID, PK)
- tenant_id (FK)
- email
- role (admin, member, viewer)
- auth_provider
- created_at

### Experiment
- experiment_id (UUID, PK)
- tenant_id (FK)
- name
- description
- status (queued, running, completed, failed, cancelled)
- difficulty_level
- seed
- config_hash
- created_at
- updated_at

### ExperimentConfig (Immutable Snapshot)
- experiment_id (PK, FK)
- schema_config (JSONB)
- distribution_config (JSONB)
- causal_graph (JSONB)
- failure_config (JSONB)
- output_format
- version

### ResourceAllocation
- experiment_id (FK)
- cpu_cores
- memory_gb
- gpu_count
- estimated_cost
- actual_cost
- started_at
- finished_at

### DatasetArtifact
- artifact_id (UUID, PK)
- experiment_id (FK)
- storage_uri
- checksum_sha256
- size_bytes
- metadata_json
- signed
- created_at

---

# 2. Phased Engineering Execution Roadmap (0 → Production)

Phase 0 – Mathematical Core
- Implement deterministic RNG wrapper
- Implement distribution generators
- KS-test validation
- Seed registry

Phase 1 – Tabular Engine MVP
- Schema DSL
- Independent feature generation
- Dependent variable simulation
- CSV export
- Single-tenant deployment

Phase 2 – Control Plane
- Experiment lifecycle orchestration
- Async execution
- Resource estimation heuristics
- Kubernetes job execution

Phase 3 – Causal Engine
- DAG builder
- Topological sorting
- SEM executor
- Intervention simulation

Phase 4 – Difficulty & Failure Injection
- Linear separability metric
- Noise injection
- Missingness injection
- Drift simulation

Phase 5 – Multi-Modal
- Time-series generator
- Image GAN/VAE cluster
- NLP synthetic corpus engine

Phase 6 – Production Hardening
- Multi-tenant isolation
- Rate limiting
- Observability stack
- Autoscaling
- Cost tracking
- DR strategy

---

# 3. Core Statistical & Causal Algorithms (Mathematical Definitions)

## 3.1 Distribution Enforcement

Given target distribution D with parameters θ,
Generate X ~ D(θ).

Post-generation validation:

Kolmogorov-Smirnov statistic:
D_n = sup_x |F_n(x) − F(x)|

Accept if p-value > α (default α = 0.05).

Auto-correction loop:
Minimize:
L(θ') = KS(X_generated, D(θ'))

---

## 3.2 Structural Equation Modeling

For DAG G(V, E):

For each node v:

v = f(parents(v)) + ε_v

Where ε_v ~ N(0, σ²_v)

Generation follows topological order.

---

## 3.3 Difficulty Index

Linear Separability Score:
S_ls = 1 − (min hinge loss over linear classifier)

Noise Ratio:
S_noise = Var(ε) / Var(signal)

Entropy Score:
H(X) = − Σ p(x) log p(x)

Final Difficulty Index:
D_final = w1 S_ls + w2 S_noise + w3 H(X) + w4 overfit_risk

---

# 4. Infrastructure Cost Estimation Model

Total Cost C:

C = (CPU_hours × CPU_rate) + (GPU_hours × GPU_rate)
    + (Storage_GB × Storage_rate)
    + (Egress_GB × Egress_rate)
    + Orchestration_overhead

Predict CPU_hours via:

CPU_hours ≈ (rows × features × complexity_factor) / compute_constant

Complexity factor derived from:
- presence of DAG
- multi-modal mode
- failure injection layers

---

# 5. Security & Compliance Deep-Dive

Security Layers:

1. Identity & Access
- OAuth2
- JWT validation
- RBAC per tenant

2. Transport
- TLS 1.3
- mTLS internal communication

3. Storage
- AES-256 encryption at rest
- KMS-managed keys

4. Tenant Isolation
- Row-level security in PostgreSQL
- Per-tenant object storage prefix

5. Compliance Targets
- SOC2-ready logging
- GDPR data deletion pipeline
- Audit trail immutable logs

---

# 6. Database Schema Design (PostgreSQL)

Tables:

CREATE TABLE tenants (
  tenant_id UUID PRIMARY KEY,
  name TEXT,
  plan_type TEXT,
  quota_config JSONB,
  created_at TIMESTAMP
);

CREATE INDEX idx_tenants_plan ON tenants(plan_type);

CREATE TABLE experiments (
  experiment_id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants,
  status TEXT,
  difficulty_level TEXT,
  seed BIGINT,
  created_at TIMESTAMP
);

CREATE INDEX idx_experiments_tenant ON experiments(tenant_id);
CREATE INDEX idx_experiments_status ON experiments(status);

Partitioning Strategy:
- experiments partitioned by tenant_id hash

---

# 7. Multi-Tenant Isolation Model

Isolation Levels:

1. Logical Isolation
- tenant_id mandatory in all queries
- PostgreSQL Row-Level Security policy

2. Storage Isolation
- S3 path: /tenant_id/experiment_id/

3. Compute Isolation
- Namespace-per-tenant in Kubernetes (enterprise tier)

---

# 8. SLA, SLO & Rate Limiting

SLA:
- 99.5% uptime (Pro)
- 99.9% uptime (Enterprise)

SLOs:
- 95% of tabular jobs < 3 min
- 99% experiment status API < 200ms

Rate Limits:
- Free: 5 concurrent experiments
- Pro: 20 concurrent
- Enterprise: configurable

Token bucket algorithm per tenant.

---

# 9. Event Contract Specification (Kafka)

Topic: experiment.created

Key: experiment_id
Value:
{
  "experiment_id": "UUID",
  "tenant_id": "UUID",
  "timestamp": "ISO8601",
  "config_version": "v1"
}

Versioning:
- Backward compatible schema evolution
- Additive fields only
- Schema registry enforced

---

# 10. Protobuf Definitions (gRPC)

syntax = "proto3";

message ExperimentConfig {
  string experiment_id = 1;
  string tenant_id = 2;
  string difficulty_level = 3;
  bytes schema_config = 4;
}

service SchedulerService {
  rpc EstimateResources(ExperimentConfig) returns (ResourceEstimate);
}

message ResourceEstimate {
  int32 cpu_cores = 1;
  double memory_gb = 2;
  bool gpu_required = 3;
  double estimated_cost_usd = 4;
}

---

# 11. OpenAPI (Swagger) Schema Outline

openapi: 3.0.0

paths:
  /experiments:
    post:
      summary: Create experiment
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ExperimentCreateRequest'

components:
  schemas:
    ExperimentCreateRequest:
      type: object
      properties:
        name:
          type: string
        schema:
          type: object
        difficulty_level:
          type: string
        failure_injection:
          type: object

---

# 12. Infrastructure Summary

- Kubernetes cluster with node pools (CPU/GPU)
- PostgreSQL HA cluster
- Redis cluster
- Kafka cluster
- Object storage (S3-compatible)
- Prometheus + Grafana
- ELK stack

---

End of Advanced System Documents

