# HackForge AI – Expanded Infrastructure Cost Estimation Model

This document extends the Infrastructure Cost Estimation Model and formally integrates with:
- PRD differentiation pillars (statistical guarantees, causal modeling, difficulty calibration)
- Technical Architecture (control plane vs compute plane separation)
- Microservices API Contract (scheduler-service → EstimateResources)
- Internal Data Models (ResourceAllocation, ExperimentRun, MetricSnapshot)
- Engineering Roadmap (Phase 2 Scheduler, Phase 6 Cost Tracking)
- Mathematical Definitions (resource complexity model)

The goal is to transform cost estimation from a heuristic into a deterministic, model-driven subsystem aligned with reproducibility, tenant isolation, and difficulty calibration.

---------------------------------------------------------------------

# 1. Cost Model Design Principles

1. Cost must be predictable before execution (pre-run estimate).
2. Cost must be measured precisely after execution (post-run reconciliation).
3. Estimation must map to ExperimentConfig complexity parameters.
4. Cost must be tenant-aware and plan-aware.
5. GPU vs CPU decision must be deterministic and explainable.

All estimates are generated inside scheduler-service and persisted via ResourceAllocation and MetricSnapshot entities.

---------------------------------------------------------------------

# 2. Cost Components (Full Decomposition)

Total Experiment Cost C_total is decomposed as:

C_total = C_compute + C_storage + C_egress + C_control_plane + C_observability + C_overhead

Where:

C_compute = (CPU_hours × r_cpu) + (GPU_hours × r_gpu)
C_storage = (GB_month × r_storage)
C_egress = (GB_out × r_egress)
C_control_plane = orchestration_cost
C_observability = logging + metrics + tracing
C_overhead = safety margin buffer

Each component maps to architectural layers defined in the Technical Architecture.

---------------------------------------------------------------------

# 3. Compute Cost Estimation (Core Model)

Let:

n = row_count
f = feature_count

Base computational volume:

V_base = n × f

Define complexity factor c as:

c = c_dist + c_causal + c_difficulty + c_failure + c_modal

Where:

c_dist = distribution enforcement multiplier
c_causal = DAG depth × nonlinear factor
c_difficulty = probe model training overhead
c_failure = injection layers multiplier
c_modal = multi-modal compute multiplier

Total effective volume:

V_eff = V_base × c

CPU_hours ≈ V_eff / κ_cpu
GPU_hours ≈ V_gpu / κ_gpu

Where κ_cpu and κ_gpu are empirically calibrated constants stored in scheduler-service configuration.

This directly maps to the mathematical definition:
CPU_hours ≈ (n × f × c) / κ

---------------------------------------------------------------------

# 4. Complexity Factor Formalization

## 4.1 Distribution Complexity (c_dist)

c_dist = 1 + α1 × custom_distribution_count
        + α2 × auto_correction_iterations

## 4.2 Causal Graph Complexity (c_causal)

Let d = maximum DAG depth
Let e = number of edges

c_causal = 1 + β1 × d + β2 × (e / f)

Nonlinear SEM adds multiplier γ_nl.

## 4.3 Difficulty Calibration Overhead (c_difficulty)

Let k = number of adaptive regeneration loops

c_difficulty = 1 + δ1 × k

Probe model cost:
O(n × f × epochs)

## 4.4 Failure Injection Complexity (c_failure)

c_failure = 1 + η1 × missingness_layers
               + η2 × adversarial_steps
               + η3 × drift_windows

## 4.5 Multi-Modal Multiplier (c_modal)

Tabular only: 1
Time-series: 1.2
Image (GAN/VAE): 3–8 (GPU bound)
NLP corpus: 2–5
Graph simulation: 1.5–3

---------------------------------------------------------------------

# 5. GPU Scheduling Decision Function

Define GPU requirement G as:

G = 1 if
    (modal_type ∈ {image, multimodal})
    OR (n × f > threshold_large)
    OR (probe_model_size > threshold_model)
Else G = 0

This feeds directly into scheduler-service → ResourceEstimate.gpu_required.

---------------------------------------------------------------------

# 6. Storage Cost Model

Artifact size approximation:

S_bytes ≈ n × f × avg_bytes_per_feature

For numeric tabular:
avg_bytes_per_feature ≈ 8–16 bytes

S_GB = S_bytes / (1024³)

C_storage = S_GB × retention_months × r_storage

Retention policy derived from tenant plan (free/pro/enterprise).

---------------------------------------------------------------------

# 7. Egress Cost Model

If dataset downloaded k times:

C_egress = S_GB × k × r_egress

Enterprise plans may bundle fixed egress quotas.

---------------------------------------------------------------------

# 8. Control Plane Cost Allocation

Control plane cost per experiment:

C_control_plane = T_orch × r_orch_unit

Where T_orch includes:
- validation-service runtime
- experiment-service coordination
- event streaming overhead

This ensures separation between compute plane and orchestration plane per Technical Architecture.

---------------------------------------------------------------------

# 9. Observability Cost Model

Metrics:
- Prometheus time-series volume
- Log ingestion size
- Trace storage

Approximation:

C_observability = (log_GB + metric_GB + trace_GB) × r_observe

Logs scale roughly with:
O(stage_count × feature_count)

---------------------------------------------------------------------

# 10. Pre-Run Estimation Algorithm (Scheduler-Service)

Input: ExperimentConfig

Step 1: Extract (n, f)
Step 2: Compute complexity components
Step 3: Estimate CPU_hours, GPU_hours
Step 4: Estimate storage size
Step 5: Compute cost components
Step 6: Apply tenant plan constraints
Step 7: Return ResourceEstimate

Output (matches API contract):
{
  cpu_cores,
  memory_gb,
  gpu_required,
  estimated_cost_usd,
  estimated_runtime_seconds
}

---------------------------------------------------------------------

# 11. Post-Run Reconciliation Model

Actual metrics retrieved from MetricSnapshot:

- cpu_hours_actual
- gpu_hours_actual
- storage_gb_actual
- compute_time_seconds

Actual Cost:

C_actual = cpu_hours_actual·r_cpu + gpu_hours_actual·r_gpu
         + storage_gb_actual·r_storage
         + egress_gb_actual·r_egress

Variance:

ΔC = C_actual − C_estimated

Used for:
- Model recalibration
- κ_cpu, κ_gpu refinement
- Anomaly detection

---------------------------------------------------------------------

# 12. Tenant Plan Cost Governance

Free Tier:
- Hard row cap
- No GPU usage
- Storage TTL enforced

Pro Tier:
- Soft quota with throttling
- Limited GPU window

Enterprise:
- Dedicated namespace
- Custom compute pricing
- Reserved cluster option

Quota enforcement integrates with Tenant.quota_config and ResourceAllocation.

---------------------------------------------------------------------

# 13. Cost-Aware Adaptive Generation

If estimated_cost > tenant_quota:

Options:
1. Reduce row_count
2. Reduce feature_count
3. Disable adaptive loops
4. Lower difficulty target

This integrates cost awareness into generation logic rather than post-failure rejection.

---------------------------------------------------------------------

# 14. Regional & Multi-Cluster Considerations

Total cost across regions r:

C_total = Σ C_region_i

Where region-specific rates differ:

r_cpu_i, r_gpu_i, r_storage_i

Scheduler selects region minimizing:

min_i (C_compute_i + C_storage_i + latency_penalty_i)

---------------------------------------------------------------------

# 15. Deterministic Cost Reproducibility

Given identical ExperimentConfig and seed, cost estimate must satisfy:

C_estimate(H, S) = deterministic

Even though actual runtime may vary slightly, estimation function is purely config-driven.

---------------------------------------------------------------------

# 16. SLA-Linked Cost Buffering

Safety margin buffer:

C_overhead = λ × C_compute

λ depends on SLA tier:
- Free: 5%
- Pro: 10%
- Enterprise: 15%

Ensures cost predictability under burst conditions.

---------------------------------------------------------------------

# 17. Financial Projection Layer (Business Model Integration)

Expected Monthly Revenue:

R = Σ (tenant_i_subscription + usage_i_overage)

Compute Margin:

Margin = (R − Σ C_actual) / R

Target margin threshold configurable.

---------------------------------------------------------------------

# 18. Integration Mapping to System Documents

PRD: Cost aligns with deterministic simulation strategy and monetization tiers.
Technical Architecture: Control-plane vs compute-plane cost separation.
API Contract: Scheduler EstimateResources contract preserved.
Internal Data Models: ResourceAllocation, MetricSnapshot persistence.
Engineering Roadmap: Phase 2 (estimation), Phase 6 (cost tracking).
Mathematical Definitions: κ-based compute scaling formalized.

---------------------------------------------------------------------

# 19. Strategic Outcome

This expanded cost model converts infrastructure estimation from a flat linear formula into a multi-factor, architecture-aligned, deterministic financial model.

It ensures:
- Predictive cost governance
- Tenant-aware scaling
- Reproducible estimation
- GPU rationalization
- Feedback-driven calibration

The result is a financially sustainable experimental simulation engine that scales consistently with its mathematical and architectural guarantees.

