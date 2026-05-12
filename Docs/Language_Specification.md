# HackForge AI – Language & Technology Architecture Specification (Maximum Optimization Edition)

This document formally specifies the programming languages and technology stack to be used in each repository and folder of HackForge AI.

The objective is maximum optimization across:
- Deterministic reproducibility
- Compute performance
- Concurrency scalability
- Memory efficiency
- Infrastructure control
- ML performance
- Multi-tenant isolation
- Operational reliability

This stack assumes enterprise-grade deployment and high-scale dataset generation.

---------------------------------------------------------------------

# 1. Top-Level Repository Language Mapping

hackforge-ai/
├── hackforge-contracts/      → YAML + Protobuf + Avro
├── hackforge-control-plane/  → Go
├── hackforge-compute-plane/  → Go (orchestration) + Python (execution)
├── hackforge-ml-engines/     → Python (NumPy/SciPy/PyTorch)
├── hackforge-infra/          → Terraform + Helm + YAML
├── hackforge-client/         → TypeScript (React) + Python SDK
├── docs/                     → Markdown

---------------------------------------------------------------------

# 2. hackforge-contracts/

Purpose:
Authoritative API, gRPC, and event schema definitions.

Languages:
- OpenAPI → YAML
- Protobuf → proto3
- Kafka schemas → Avro (preferred) or JSON Schema
- Shared enums/docs → Markdown

Why:
- Language-agnostic
- Enables strict contract-first development
- Allows automatic stub generation

No runtime logic permitted here.

---------------------------------------------------------------------

# 3. hackforge-control-plane/

Primary Language: Go (1.21+)

Why Go:
- High-performance HTTP server
- Efficient goroutine-based concurrency
- Strong gRPC and Kafka ecosystem
- Small memory footprint
- Fast container startup
- Easy static binary builds

Subfolder Mapping:

cmd/
  api-gateway/            → Go
  experiment-service/     → Go
  validation-service/     → Go
  scheduler-service/      → Go
  seed-service/           → Go
  artifact-service/       → Go

internal/
  domain/                 → Go (pure domain models)
  application/            → Go (use cases, lifecycle orchestration)
  infrastructure/         → Go (DB, Kafka, Redis, security)
  config/                 → YAML config files

migrations/
  SQL (PostgreSQL DDL)

Technologies:
- PostgreSQL (metadata store)
- Redis (rate limiting, idempotency)
- Kafka client for Go
- gRPC (generated from Protobuf)

Optimization Rationale:
- Go handles concurrency-heavy lifecycle orchestration
- Efficient scheduler implementation
- Low latency (<200ms API SLO)

---------------------------------------------------------------------

# 4. hackforge-ml-engines/

Primary Language: Python 3.11+

Why Python:
- NumPy (vectorized operations)
- SciPy (statistical distributions)
- PyTorch (GAN/VAE)
- scikit-learn (probe models)
- NetworkX (DAG operations)

Folder Mapping:

statistical/             → Python (NumPy/SciPy)
causal/                  → Python (NumPy + NetworkX)
difficulty/              → Python (NumPy + sklearn)
failure_injection/       → Python
multimodal/
  timeseries/            → Python
  gan/                   → Python + PyTorch
  vae/                   → Python + PyTorch
  nlp/                   → Python
  graph/                 → Python

Rules:
- Must be deterministic
- No network access
- No database writes
- Pure computation layer

Optimization Enhancements:
- Vectorized operations only
- Avoid Python loops for large arrays
- Use torch GPU acceleration when required
- Optional Numba acceleration for heavy loops

---------------------------------------------------------------------

# 5. hackforge-compute-plane/

Hybrid Model for Maximum Optimization

Language Split:
- Orchestration Layer → Go
- Execution Layer → Python

Structure:

jobs/
  tabular-generator/      → Python execution
  causal-engine/          → Python execution
  difficulty-engine/      → Python execution
  failure-engine/         → Python execution
  multimodal-engine/      → Python execution

entrypoints/
  job-runner/             → Go (Kafka consumer + orchestration)

shared/
  rng/                    → Go + Python deterministic wrapper
  artifact-writer/        → Go
  compliance-checker/     → Go

Why Hybrid:
- Go manages job lifecycle and Kafka reliability
- Python performs heavy matrix and ML operations
- Clear separation of orchestration vs compute

Optimization Rationale:
- Prevents Python from handling high-throughput messaging
- Keeps ML engine pure and fast
- Minimizes container overhead

---------------------------------------------------------------------

# 6. hackforge-infra/

Languages:
- Terraform (HCL)
- Helm (YAML templates)
- Kubernetes manifests (YAML)
- Bash (automation scripts)

Terraform modules:
- network/
- kubernetes/
- kafka/
- postgres/
- redis/
- object-storage/

Monitoring:
- Prometheus (YAML)
- Grafana dashboards (JSON)

Optimization Rationale:
- Declarative infrastructure
- Autoscaling policies
- Node pool segregation (CPU vs GPU)

---------------------------------------------------------------------

# 7. hackforge-client/

Web Client:
- TypeScript
- React
- WebSocket API

SDK:
- Python (for data scientists)
- JavaScript (optional)

CLI:
- Go (recommended for consistency)
  OR
- Python (if targeting data science users)

Optimization Rationale:
- Type safety via TypeScript
- OpenAPI-generated clients
- Async progress monitoring

---------------------------------------------------------------------

# 8. Testing Stack

Control Plane:
- Go testing framework
- Testcontainers (Postgres + Kafka)

ML Engines:
- pytest
- Determinism regression tests
- Statistical compliance tests

Load Testing:
- k6 (JavaScript)

Security Testing:
- gosec
- Bandit (Python)

---------------------------------------------------------------------

# 9. Cross-Language Determinism Strategy

To ensure deterministic reproducibility across Go and Python:

1. Seed derived in Go
2. config_hash + seed combined
3. Passed to Python as explicit parameter
4. Python RNG initialized explicitly
5. No implicit randomness

All randomness must flow from a single deterministic root.

---------------------------------------------------------------------

# 10. Performance Optimization Guidelines

Go:
- Use connection pooling
- Avoid reflection-heavy libraries
- Use context propagation

Python:
- Use NumPy vectorization
- Avoid Pandas for core generation
- Preallocate arrays
- Use GPU only when modal_type requires

PostgreSQL:
- Hash partition by tenant_id
- GIN indexes on JSONB

Kafka:
- Partition by tenant_id
- Idempotent producers

Kubernetes:
- HPA for control plane
- Separate CPU and GPU node pools

---------------------------------------------------------------------

# 11. Final Language Summary Table

Contracts Layer → YAML + Protobuf + Avro
Control Plane → Go
Compute Orchestration → Go
ML Engines → Python
Infrastructure → Terraform + YAML
Database → PostgreSQL (SQL)
Cache → Redis
Event Bus → Kafka
Client → TypeScript (React) + Python SDK
Monitoring → PromQL + YAML

---------------------------------------------------------------------

# 12. Strategic Outcome

This language distribution ensures:

- Maximum runtime efficiency
- Clean architectural separation
- Deterministic reproducibility
- High concurrency control
- Scalable GPU-accelerated ML execution
- Enterprise-grade observability

The system remains modular, performant, and maintainable while preserving strict control-plane / compute-plane boundaries.

