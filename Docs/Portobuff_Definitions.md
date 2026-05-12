# HackForge AI – Protobuf Definitions (Expanded & Cross-Integrated)

This document formalizes the complete Protobuf (proto3) contract layer for all internal gRPC communication across HackForge AI microservices. It integrates directly with:

* PRD (deterministic reproducibility, statistical compliance ≥95%) fileciteturn0file2
* Technical Architecture Blueprint (control plane vs compute plane separation) fileciteturn0file0
* Microservices API Contract (service boundaries & async lifecycle) fileciteturn0file1
* Internal Data Models (canonical entity schema) fileciteturn0file3
* Engineering Roadmap (phase-aligned execution model) fileciteturn0file5
* Mathematical Definitions (config_hash + seed invariants) fileciteturn0file6
* Cost Estimation Model (scheduler EstimateResources contract) fileciteturn0file7
* Security & Compliance (tenant isolation, mTLS, immutability) fileciteturn0file8
* PostgreSQL Schema & Indexing (immutability & uniqueness constraints) fileciteturn0file9
* Multi-Tenant Isolation Model (tenant-scoped invariants) fileciteturn0file10
* SLA/SLO & Rate Limiting (quota enforcement & admission control) fileciteturn0file11
* Kafka Event Contracts (event-driven state machine alignment) fileciteturn0file12

Protobuf definitions are the strongly typed internal contract between control plane and compute plane services. They enforce deterministic semantics, tenant scoping, and idempotent execution across the distributed system.

---

# 1. Global Conventions

syntax = "proto3";

package hackforge.internal.v1;

option go_package = "github.com/hackforge/internal/v1";
option java_multiple_files = true;
option java_package = "ai.hackforge.internal.v1";

All requests MUST include:

* tenant_id
* correlation_id
* idempotency_key (where applicable)

All services must operate under mTLS as defined in Security & Compliance.

---

# 2. Common Message Types

message TenantContext {
string tenant_id = 1;          // UUID
string correlation_id = 2;     // Lifecycle trace
string request_id = 3;         // Per-hop ID
}

message ExperimentIdentity {
string experiment_id = 1;
string config_hash = 2;
int64 seed = 3;
int32 version = 4;
}

message RunIdentity {
string run_id = 1;
string experiment_id = 2;
string tenant_id = 3;
}

message Pagination {
int32 page_size = 1;
string page_token = 2;
}

---

# 3. Experiment Service (Control Plane)

service ExperimentService {
rpc CreateExperiment(CreateExperimentRequest)
returns (CreateExperimentResponse);

rpc GetExperimentStatus(GetExperimentStatusRequest)
returns (GetExperimentStatusResponse);

rpc CancelExperiment(CancelExperimentRequest)
returns (CancelExperimentResponse);
}

message CreateExperimentRequest {
TenantContext context = 1;
string name = 2;
string description = 3;
bytes schema_config = 4;        // JSON serialized
bytes distribution_config = 5;
bytes causal_graph = 6;
bytes failure_config = 7;
bytes difficulty_config = 8;
bytes output_config = 9;
int64 row_count = 10;
int32 feature_count = 11;
string idempotency_key = 12;
}

message CreateExperimentResponse {
string experiment_id = 1;
string status = 2; // draft | validated | queued
}

message GetExperimentStatusRequest {
TenantContext context = 1;
string experiment_id = 2;
}

message GetExperimentStatusResponse {
string status = 1;
int32 progress_percentage = 2;
int64 estimated_time_remaining_seconds = 3;
}

message CancelExperimentRequest {
TenantContext context = 1;
string experiment_id = 2;
}

message CancelExperimentResponse {
bool cancelled = 1;
}

---

# 4. Validation Service

service ValidationService {
rpc ValidateExperiment(ValidateExperimentRequest)
returns (ValidationResult);
}

message ValidateExperimentRequest {
TenantContext context = 1;
ExperimentIdentity experiment = 2;
bytes schema_config = 3;
bytes distribution_config = 4;
bytes causal_graph = 5;
}

message ValidationResult {
bool valid = 1;
repeated string errors = 2;
repeated string warnings = 3;
}

---

# 5. Scheduler Service (Cost & Admission Control)

service SchedulerService {
rpc EstimateResources(ResourceEstimateRequest)
returns (ResourceEstimate);

rpc AllocateCompute(AllocateComputeRequest)
returns (AllocateComputeResponse);

rpc ReleaseCompute(ReleaseComputeRequest)
returns (ReleaseComputeResponse);
}

message ResourceEstimateRequest {
TenantContext context = 1;
ExperimentIdentity experiment = 2;
int64 row_count = 3;
int32 feature_count = 4;
string modal_type = 5; // tabular | image | nlp | graph
}

message ResourceEstimate {
int32 cpu_cores = 1;
double memory_gb = 2;
bool gpu_required = 3;
double estimated_cost_usd = 4;
int64 estimated_runtime_seconds = 5;
}

message AllocateComputeRequest {
TenantContext context = 1;
RunIdentity run = 2;
}

message AllocateComputeResponse {
string compute_namespace = 1;
bool admitted = 2;
string rejection_reason = 3;
}

message ReleaseComputeRequest {
TenantContext context = 1;
RunIdentity run = 2;
}

message ReleaseComputeResponse {
bool released = 1;
}

---

# 6. Statistical Engine (Distribution Enforcement)

service StatisticalEngine {
rpc GenerateIndependentFeatures(GenerateFeaturesRequest)
returns (FeatureBatch);

rpc ValidateDistribution(ValidateDistributionRequest)
returns (DistributionReport);
}

message GenerateFeaturesRequest {
TenantContext context = 1;
RunIdentity run = 2;
bytes feature_spec = 3;
}

message FeatureBatch {
bytes encoded_matrix = 1; // columnar binary
int64 row_count = 2;
int32 feature_count = 3;
}

message ValidateDistributionRequest {
TenantContext context = 1;
RunIdentity run = 2;
string feature_id = 3;
bytes generated_values = 4;
}

message DistributionReport {
string report_id = 1;
string feature_id = 2;
double ks_statistic = 3;
double p_value = 4;
double distribution_match_score = 5;
bool passed = 6;
}

---

# 7. Causal Engine

service CausalEngine {
rpc BuildGraph(BuildGraphRequest) returns (BuildGraphResponse);
rpc SimulateDependencies(SimulateDependenciesRequest)
returns (FeatureBatch);
rpc RunIntervention(InterventionRequest)
returns (FeatureBatch);
}

message BuildGraphRequest {
TenantContext context = 1;
ExperimentIdentity experiment = 2;
bytes causal_graph = 3;
}

message BuildGraphResponse {
string graph_id = 1;
}

message SimulateDependenciesRequest {
TenantContext context = 1;
string graph_id = 2;
bytes independent_features = 3;
}

message InterventionRequest {
TenantContext context = 1;
string graph_id = 2;
string target_feature = 3;
double forced_value = 4;
}

---

# 8. Difficulty Engine

service DifficultyEngine {
rpc ComputeDifficulty(DifficultyRequest)
returns (DifficultyReport);
}

message DifficultyRequest {
TenantContext context = 1;
RunIdentity run = 2;
bytes dataset_ref = 3;
}

message DifficultyReport {
double linear_separability_score = 1;
double noise_ratio = 2;
double entropy_score = 3;
double overfitting_risk_score = 4;
double class_imbalance_index = 5;
double final_difficulty_index = 6;
}

---

# 9. Failure Injection Engine

service FailureEngine {
rpc InjectFailures(FailureInjectionRequest)
returns (FailureInjectionReport);
}

message FailureInjectionRequest {
TenantContext context = 1;
RunIdentity run = 2;
bytes failure_config = 3;
}

message FailureInjectionReport {
string injection_id = 1;
string missingness_type = 2;
double missing_ratio = 3;
double adversarial_noise_level = 4;
double drift_strength = 5;
bool leakage_enabled = 6;
}

---

# 10. Artifact Service

service ArtifactService {
rpc FinalizeArtifact(FinalizeArtifactRequest)
returns (FinalizeArtifactResponse);

rpc GenerateSignedUrl(SignedUrlRequest)
returns (SignedUrlResponse);
}

message FinalizeArtifactRequest {
TenantContext context = 1;
RunIdentity run = 2;
bytes dataset_binary = 3;
string format = 4;
}

message FinalizeArtifactResponse {
string artifact_id = 1;
string checksum_sha256 = 2;
int64 size_bytes = 3;
}

message SignedUrlRequest {
TenantContext context = 1;
string artifact_id = 2;
}

message SignedUrlResponse {
string signed_url = 1;
int64 expiry_epoch_seconds = 2;
}

---

# 11. Observability & Metrics Service

service MonitoringService {
rpc RecordMetric(MetricRequest) returns (MetricAck);
}

message MetricRequest {
TenantContext context = 1;
string run_id = 2;
double cpu_hours = 3;
double gpu_hours = 4;
double storage_gb = 5;
double compliance_score = 6;
}

message MetricAck {
bool recorded = 1;
}

---

# 12. Deterministic Integrity Constraints (Cross-Service)

Constraint 1:
ExperimentIdentity (config_hash + seed + version) must uniquely define dataset semantics.

Constraint 2:
Scheduler EstimateResources must be pure function of ExperimentConfig.

Constraint 3:
DistributionReport.passed must be immutable once persisted.

Constraint 4:
Artifact checksum_sha256 must equal SHA256(dataset_binary).

Constraint 5:
All requests must propagate TenantContext.

---

# 13. Versioning & Backward Compatibility

* All messages are additive-only.
* Field numbers never reused.
* Deprecated fields marked but retained.
* Major breaking changes require package v2.

---

# 14. Strategic Outcome

These Protobuf definitions establish a strongly typed, deterministic, tenant-isolated internal contract across the HackForge AI microservices architecture.

They ensure:

* Strict control plane ↔ compute plane separation
* Deterministic reproducibility invariants preserved at transport layer
* Cost-aware admission and scheduling
* Immutable statistical validation persistence
* SLA-aligned lifecycle orchestration
* Replay-safe event-driven integration

The Protobuf layer therefore functions as the structural backbone of the distributed experimental simulation engine, binding architecture, mathematics, cost governance, and multi-tenant security into a coherent internal execution contract.
