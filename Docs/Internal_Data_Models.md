# HackForge AI – Internal Data Models (Expanded Specification)

This document defines the canonical internal domain model for HackForge AI. The design follows strict immutability for experiment configuration, deterministic reproducibility guarantees, and full tenant isolation.

---

# 1. Core Domain Model Principles

1. All mutable runtime state is separated from immutable configuration snapshots.
2. Every experiment is reproducible via (config_hash + seed).
3. All entities are tenant-scoped.
4. Artifacts are content-addressable.
5. Statistical validation outputs are first-class entities.

---

# 2. Identity & Tenant Model

## 2.1 Tenant

Represents an isolated organizational boundary.

Fields:
- tenant_id (UUID, PK)
- name (TEXT)
- slug (TEXT, unique)
- plan_type (ENUM: free, pro, enterprise)
- status (ENUM: active, suspended)
- quota_config (JSONB)
- billing_account_id (TEXT)
- created_at (TIMESTAMP)
- updated_at (TIMESTAMP)

Derived Constraints:
- Unique slug
- Plan-based quota enforcement

---

## 2.2 User

Fields:
- user_id (UUID, PK)
- tenant_id (UUID, FK)
- email (TEXT, unique within tenant)
- role (ENUM: owner, admin, member, viewer)
- auth_provider (ENUM: local, google, github, oidc)
- last_login_at
- created_at

---

# 3. Experiment Aggregate

Experiment is the central aggregate root.

## 3.1 Experiment

Fields:
- experiment_id (UUID, PK)
- tenant_id (UUID, FK)
- name
- description
- status (ENUM: draft, validated, queued, running, completed, failed, cancelled)
- difficulty_level (ENUM: beginner, intermediate, advanced, kaggle)
- seed (BIGINT)
- config_hash (SHA256 TEXT)
- reproducibility_token (TEXT)
- version (INT)
- created_by (UUID FK user)
- created_at
- updated_at

Invariant:
config_hash = SHA256(serialized immutable config)

---

## 3.2 ExperimentConfig (Immutable Snapshot)

Fields:
- experiment_id (PK, FK)
- schema_config (JSONB)
- distribution_config (JSONB)
- causal_graph (JSONB)
- failure_config (JSONB)
- difficulty_config (JSONB)
- output_config (JSONB)
- row_count (BIGINT)
- feature_count (INT)
- created_at

Immutability Rule:
No updates allowed after validation.
Version increments create new config snapshot.

---

# 4. Schema Modeling Layer

## 4.1 FeatureDefinition

Defines atomic feature specification.

Fields:
- feature_id (UUID)
- experiment_id (FK)
- name (TEXT)
- type (ENUM: numeric, categorical, boolean, datetime, text, tensor)
- distribution_type (ENUM: normal, poisson, lognormal, pareto, uniform, custom)
- distribution_params (JSONB)
- nullable (BOOLEAN)
- cardinality (INT nullable)
- parent_features (UUID[])
- structural_equation (TEXT)
- created_at

Constraints:
- Unique (experiment_id, name)

---

## 4.2 CausalEdge

Fields:
- edge_id (UUID)
- experiment_id
- parent_feature_id
- child_feature_id
- weight (FLOAT)
- relationship_type (linear, nonlinear, logistic)

Constraint:
Graph must remain acyclic (validated externally).

---

# 5. Runtime Execution Model

## 5.1 ExperimentRun

Represents a single execution instance.

Fields:
- run_id (UUID, PK)
- experiment_id (FK)
- tenant_id
- status (ENUM: initializing, generating, validating, injecting, packaging, completed, failed)
- progress_percentage
- started_at
- completed_at
- compute_namespace

---

## 5.2 ResourceAllocation

Fields:
- allocation_id (UUID)
- run_id
- cpu_cores
- memory_gb
- gpu_count
- node_type
- estimated_cost
- actual_cost
- billing_reference
- created_at

---

# 6. Statistical Validation Models

## 6.1 DistributionReport

Fields:
- report_id (UUID)
- run_id
- feature_id
- ks_statistic
- p_value
- distribution_match_score
- target_params (JSONB)
- actual_params (JSONB)
- passed (BOOLEAN)
- created_at

---

## 6.2 CorrelationReport

Fields:
- report_id
- run_id
- correlation_matrix (JSONB)
- mutual_information_matrix (JSONB)
- feature_entropy_map (JSONB)
- created_at

---

# 7. Difficulty & Failure Injection Models

## 7.1 DifficultyReport

Fields:
- report_id
- run_id
- linear_separability_score
- noise_ratio
- entropy_score
- overfitting_risk_score
- class_imbalance_index
- final_difficulty_index
- created_at

---

## 7.2 FailureInjectionReport

Fields:
- injection_id
- run_id
- missingness_type (MCAR, MAR, MNAR)
- missing_ratio
- adversarial_noise_level
- drift_type
- covariate_shift_strength
- leakage_enabled
- created_at

---

# 8. Artifact Model

## 8.1 DatasetArtifact

Fields:
- artifact_id (UUID)
- run_id
- experiment_id
- tenant_id
- storage_uri
- checksum_sha256
- size_bytes
- format (csv, parquet, json, torch, tfrecord)
- signed_url_expiry
- metadata_uri
- created_at

Content Addressable Rule:
checksum_sha256 uniquely identifies artifact content.

---

# 9. Observability Model

## 9.1 AuditLog

Fields:
- audit_id
- tenant_id
- user_id
- action_type
- entity_type
- entity_id
- metadata (JSONB)
- created_at

---

## 9.2 MetricSnapshot

Fields:
- metric_id
- run_id
- compute_time_seconds
- cpu_hours
- gpu_hours
- storage_gb
- compliance_score
- created_at

---

# 10. Multi-Modal Extension Model

## 10.1 TimeSeriesConfig

Fields:
- experiment_id
- frequency
- seasonality_params (JSONB)
- trend_function

## 10.2 ImageGenerationConfig

Fields:
- experiment_id
- model_type (gan, vae)
- latent_dim
- resolution

## 10.3 TextGenerationConfig

Fields:
- experiment_id
- vocabulary_size
- language_model_type
- sequence_length

---

# 11. Indexing Strategy

Primary indexes:
- experiment_id
- tenant_id
- run_id

Composite indexes:
- (tenant_id, status)
- (experiment_id, version)
- (run_id, feature_id)

JSONB indexes:
- GIN index on schema_config
- GIN index on distribution_config

---

# 12. Data Retention Rules

- Experiment metadata retained indefinitely.
- Artifacts retained per plan (configurable TTL).
- Audit logs retained minimum 1 year.

---

End of Expanded Internal Data Model Specification

