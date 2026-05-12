Hey, the document generated now contains only till.
# HackForge AI – PostgreSQL Schema & Indexing (Expanded & Cross-Integrated)

This document formalizes the PostgreSQL physical schema, indexing strategy, partitioning model, and performance governance for HackForge AI. It extends and tightly integrates with:

* PRD (deterministic reproducibility, difficulty scoring, statistical compliance) fileciteturn0file2
* Technical Architecture (control plane vs compute plane separation) fileciteturn0file0
* Microservices API Contract (tenant-scoped APIs, idempotency, async lifecycle) fileciteturn0file1
* Internal Data Models (canonical entity definitions) fileciteturn0file3
* Engineering Roadmap (multi-tenant production hardening) fileciteturn0file5
* Mathematical Definitions (deterministic seed + config_hash invariants) fileciteturn0file6
* Cost Estimation Model (ResourceAllocation & MetricSnapshot feedback loop) fileciteturn0file7
* Security & Compliance Deep Dive (RLS, audit immutability) fileciteturn0file8

PostgreSQL is used exclusively for metadata, lifecycle state, statistical reports, cost tracking, and governance. Generated datasets remain in object storage.

---

# 1. Database Design Principles

1. All entities are tenant-scoped.
2. ExperimentConfig is immutable.
3. Reproducibility invariant enforced at schema level.
4. JSONB used for flexible scientific configuration.
5. Heavy read paths are indexed for API latency SLO (<200ms per PRD SLO alignment).
6. Partitioning prepared for multi-tenant scale.

---

# 2. Core Schema Definitions (DDL-Level Specification)

## 2.1 Tenants

CREATE TABLE tenants (
tenant_id UUID PRIMARY KEY,
name TEXT NOT NULL,
slug TEXT NOT NULL UNIQUE,
plan_type TEXT NOT NULL CHECK (plan_type IN ('free','pro','enterprise')),
quota_config JSONB NOT NULL,
billing_account_id TEXT,
status TEXT NOT NULL DEFAULT 'active',
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Indexes:

* UNIQUE(slug)
* INDEX idx_tenants_plan_type(plan_type)

Rationale: Plan-based cost governance ties to Cost Estimation Model.

---

## 2.2 Users

CREATE TABLE users (
user_id UUID PRIMARY KEY,
tenant_id UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
email TEXT NOT NULL,
role TEXT NOT NULL CHECK (role IN ('owner','admin','member','viewer')),
auth_provider TEXT NOT NULL,
last_login_at TIMESTAMPTZ,
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
UNIQUE (tenant_id, email)
);

Indexes:

* INDEX idx_users_tenant(tenant_id)

---

## 2.3 Experiments (Partitioned)

CREATE TABLE experiments (
experiment_id UUID PRIMARY KEY,
tenant_id UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE CASCADE,
name TEXT NOT NULL,
description TEXT,
status TEXT NOT NULL CHECK (status IN ('draft','validated','queued','running','completed','failed','cancelled')),
difficulty_level TEXT,
seed BIGINT NOT NULL,
config_hash TEXT NOT NULL,
reproducibility_token TEXT,
version INT NOT NULL DEFAULT 1,
created_by UUID REFERENCES users(user_id),
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY HASH (tenant_id);

Partition Strategy:
CREATE TABLE experiments_p0 PARTITION OF experiments FOR VALUES WITH (MODULUS 8, REMAINDER 0);
-- additional partitions up to remainder 7

Critical Indexes:

* UNIQUE(experiment_id)
* INDEX idx_experiments_tenant_status(tenant_id, status)
* INDEX idx_experiments_created_at(created_at DESC)
* INDEX idx_experiments_config_hash(config_hash)

Reproducibility Constraint:
UNIQUE (tenant_id, config_hash, seed, version)

This enforces deterministic identity defined mathematically in the algorithmic specification.

---

## 2.4 ExperimentConfig (Immutable Snapshot)

CREATE TABLE experiment_configs (
experiment_id UUID PRIMARY KEY REFERENCES experiments(experiment_id) ON DELETE CASCADE,
schema_config JSONB NOT NULL,
distribution_config JSONB NOT NULL,
causal_graph JSONB,
failure_config JSONB,
difficulty_config JSONB,
output_config JSONB,
row_count BIGINT NOT NULL,
feature_count INT NOT NULL,
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Immutability Enforcement:
REVOKE UPDATE, DELETE ON experiment_configs FROM PUBLIC;

Indexes:

* GIN idx_schema_config ON experiment_configs USING GIN (schema_config);
* GIN idx_distribution_config ON experiment_configs USING GIN (distribution_config);
* INDEX idx_row_feature_count(row_count, feature_count);

GIN indexes support complex JSON queries from validation-service.

---

## 2.5 ExperimentRun

CREATE TABLE experiment_runs (
run_id UUID PRIMARY KEY,
experiment_id UUID NOT NULL REFERENCES experiments(experiment_id) ON DELETE CASCADE,
tenant_id UUID NOT NULL,
status TEXT NOT NULL,
progress_percentage INT,
compute_namespace TEXT,
started_at TIMESTAMPTZ,
completed_at TIMESTAMPTZ
);

Indexes:

* INDEX idx_runs_experiment(experiment_id)
* INDEX idx_runs_tenant_status(tenant_id, status)

Hot-path optimization: status filtered frequently by API polling.

---

## 2.6 ResourceAllocation

CREATE TABLE resource_allocations (
allocation_id UUID PRIMARY KEY,
run_id UUID NOT NULL REFERENCES experiment_runs(run_id) ON DELETE CASCADE,
cpu_cores INT,
memory_gb FLOAT,
gpu_count INT,
estimated_cost NUMERIC,
actual_cost NUMERIC,
billing_reference TEXT,
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Indexes:

* INDEX idx_allocation_run(run_id)
* INDEX idx_allocation_billing(billing_reference)

Ties directly to scheduler-service EstimateResources contract.

---

## 2.7 Statistical Reports (Partitioned by run_id)

CREATE TABLE distribution_reports (
report_id UUID PRIMARY KEY,
run_id UUID NOT NULL REFERENCES experiment_runs(run_id) ON DELETE CASCADE,
feature_id UUID,
ks_statistic FLOAT,
p_value FLOAT,
distribution_match_score FLOAT,
passed BOOLEAN,
created_at TIMESTAMPTZ DEFAULT NOW()
);

Indexes:

* INDEX idx_distribution_run_feature(run_id, feature_id)
* INDEX idx_distribution_passed(run_id, passed)

Compliance score computation depends on fast aggregation of passed.

---

## 2.8 DatasetArtifact

CREATE TABLE dataset_artifacts (
artifact_id UUID PRIMARY KEY,
run_id UUID NOT NULL REFERENCES experiment_runs(run_id),
experiment_id UUID NOT NULL,
tenant_id UUID NOT NULL,
storage_uri TEXT NOT NULL,
checksum_sha256 TEXT NOT NULL,
size_bytes BIGINT,
format TEXT,
signed_url_expiry TIMESTAMPTZ,
metadata_uri TEXT,
created_at TIMESTAMPTZ DEFAULT NOW(),
UNIQUE (checksum_sha256)
);

Indexes:

* INDEX idx_artifacts_tenant(tenant_id)
* INDEX idx_artifacts_experiment(experiment_id)

Checksum uniqueness enforces content-addressable storage invariant.

---

# 3. Row-Level Security (RLS)

ALTER TABLE experiments ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_experiments
ON experiments
USING (tenant_id = current_setting('app.current_tenant')::uuid);

All tenant-scoped tables follow same policy pattern.

This implements isolation model defined in Security & Compliance deep dive.

---

# 4. Query Optimization for Core Workloads

## 4.1 Experiment Status Polling

Query Pattern:
SELECT status, progress_percentage
FROM experiment_runs
WHERE experiment_id = $1;

Covered by idx_runs_experiment.

## 4.2 Compliance Score Computation

SELECT COUNT(*) FILTER (WHERE passed)
FROM distribution_reports
WHERE run_id = $1;

Index idx_distribution_run_feature ensures index-only scan potential.

## 4.3 Cost Reconciliation

SELECT estimated_cost, actual_cost
FROM resource_allocations
WHERE run_id = $1;

Optimized via idx_allocation_run.

---

# 5. Partitioning & Scale Strategy

Phase 1: Hash partition by tenant_id (8–32 partitions).
Phase 2: For enterprise scale, sub-partition experiment_runs by created_at (monthly range partitioning).
Phase 3: Archive completed experiment_runs > retention window to cold storage.

---

# 6. JSONB Strategy & Trade-offs

Used for:

* schema_config
* distribution_config
* causal_graph
* failure_config

Advantages:

* Scientific flexibility
* Backward-compatible schema evolution

Mitigation:

* Validate at application layer
* Create expression indexes for common fields, e.g.:

CREATE INDEX idx_config_difficulty
ON experiment_configs ((difficulty_config->>'target_index'));

---

# 7. Vacuum, Autovacuum & Bloat Control

High-churn tables:

* experiment_runs
* audit_logs

Configuration:

* Lower autovacuum_vacuum_scale_factor
* Periodic REINDEX on GIN indexes

Ensures API SLO alignment.

---

# 8. Transactional Boundaries

Control Plane writes (short transactions):

* experiment creation
* status transitions
* report persistence

Compute Plane writes batched inserts (COPY-based for reports).

No long-running transactions allowed on partitioned tables.

---

# 9. Read Replicas & HA

Primary:

* Write-heavy control plane operations

Read Replica:

* Analytics dashboards
* Cost aggregation
* Monitoring queries

Synchronous replication for enterprise SLA.

---

# 10. Deterministic Integrity Constraints

Invariant 1:
(config_hash, seed, version) uniquely identify dataset semantics.

Invariant 2:
checksum_sha256 uniquely identifies dataset artifact.

Invariant 3:
ExperimentConfig immutable after validation.

Database-level constraints reinforce mathematical determinism.

---

# 11. Observability & Index Monitoring

Use:

* pg_stat_statements
* auto_explain
* index_usage metrics

Quarterly index review:

* Drop unused indexes
* Rebalance partition counts

---

# 12. Strategic Outcome

This PostgreSQL schema and indexing model ensures:

* Deterministic reproducibility enforced at constraint level
* Sub-200ms experiment lifecycle queries
* Multi-tenant isolation via RLS + partitioning
* Cost tracking aligned with scheduler-service estimates
* Immutable statistical compliance persistence
* Enterprise-ready scale-out path

The database becomes a deterministic metadata control plane backing a distributed experimental simulation engine, fully aligned with architectural, mathematical, security, and cost governance layers defined across the project documentation.


