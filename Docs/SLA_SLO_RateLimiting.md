# HackForge AI – SLA, SLO & Rate Limiting Model (Expanded & Cross-Integrated)

This document formalizes the Service Level Agreements (SLA), Service Level Objectives (SLO), error budgets, rate limiting controls, and workload governance mechanisms for HackForge AI. It integrates directly with:

- PRD (deterministic reproducibility, statistical compliance >95%) fileciteturn0file2
- Technical Architecture Blueprint (control plane vs compute plane separation) fileciteturn0file0
- Microservices API Contract (async lifecycle, idempotency, tenant headers) fileciteturn0file1
- Internal Data Models (ExperimentRun, ResourceAllocation, MetricSnapshot) fileciteturn0file3
- Engineering Roadmap (Phase 6 – Production Hardening) fileciteturn0file5
- Mathematical Definitions (deterministic seed + config_hash invariants) fileciteturn0file6
- Cost Estimation Model (scheduler-based admission control) fileciteturn0file7
- Security & Compliance (JWT, RLS, namespace isolation) fileciteturn0file8
- PostgreSQL Schema & Indexing (sub-200ms metadata queries) fileciteturn0file9
- Multi-Tenant Isolation Model (quota, namespace, partition isolation) fileciteturn0file10

SLA/SLO definitions are not generic uptime metrics. They are mathematically and architecturally bound to deterministic experiment execution, statistical compliance guarantees, and tenant-scoped compute governance.

---------------------------------------------------------------------

# 1. SLA Framework

SLAs are contractual guarantees tied to tenant plan tier.

## 1.1 Availability SLA

Availability is defined as:

Availability = 1 − (Downtime_minutes / Total_minutes)

Downtime includes:
- API Gateway unavailable
- Control plane failure (experiment creation blocked)
- Artifact download failure (signed URL generation unavailable)

Excludes:
- Scheduled maintenance (pre-announced)
- Tenant quota exhaustion
- Invalid configuration rejections

Plan-based SLA:

Free Tier:
- Best effort (no contractual SLA)

Pro Tier:
- 99.5% monthly uptime

Enterprise Tier:
- 99.9% monthly uptime
- Optional multi-region active-passive configuration

SLA credits governed by billing_account_id in Tenant entity.

---------------------------------------------------------------------

# 2. SLO Definitions (System-Level Objectives)

SLOs define internal performance targets that protect SLA compliance.

## 2.1 API Latency SLOs (Control Plane)

Based on PostgreSQL indexing and async orchestration design:

- 99% of GET /experiments/{id} < 200ms
- 99% of GET /experiments/{id}/status < 150ms
- 99% of POST /experiments validation phase < 500ms (excluding compute)

These rely on:
- idx_experiments_tenant_status
- idx_runs_experiment
- short-lived transactions

## 2.2 Experiment Execution SLOs (Compute Plane)

Tabular jobs:
- 95% complete within predicted_runtime ±15%

Causal + difficulty calibration jobs:
- 90% complete within predicted_runtime ±20%

Multi-modal GPU jobs:
- 85% complete within predicted_runtime ±25%

Predicted runtime derived from scheduler EstimateResources model.

## 2.3 Statistical Compliance SLO

For any completed run:

Compliance_score = (features_passed / total_features)

SLO:
Compliance_score ≥ 95%

Failure to meet threshold prevents artifact exposure.

## 2.4 Reproducibility SLO

Given identical (config_hash, seed, tenant_id):

checksum_run1 == checksum_run2

Target failure rate:
< 0.1%

Monitored via reproducibility_failure counter.

---------------------------------------------------------------------

# 3. Error Budgets

Error budget = 1 − Availability_SLO

Enterprise (99.9%):
0.1% monthly error budget

If budget exhausted:
- Feature deployments paused
- Non-critical multi-modal jobs throttled
- Priority given to control-plane stability

This ensures architectural stability over feature velocity.

---------------------------------------------------------------------

# 4. Rate Limiting Architecture

Rate limiting operates at four layers:

1. API request rate
2. Concurrent experiment runs
3. Compute resource consumption
4. Event ingestion throughput

All limits are tenant-scoped.

---------------------------------------------------------------------

# 5. API Rate Limiting (Token Bucket Model)

Token bucket per tenant:

bucket_key = hash(tenant_id)

Parameters per plan:

Free:
- 10 requests/sec
- Burst 20

Pro:
- 50 requests/sec
- Burst 100

Enterprise:
- 200+ configurable

Refill rate = plan_config.refill_rate
Bucket capacity = plan_config.capacity

If tokens exhausted:
HTTP 429 returned
Retry-After header set

This prevents API flooding and protects metadata SLO.

---------------------------------------------------------------------

# 6. Concurrent Experiment Limits

Enforced before scheduler allocation.

Free:
- 2 concurrent runs

Pro:
- 10 concurrent runs

Enterprise:
- Configurable (default 50)

Constraint enforced via:

SELECT COUNT(*) FROM experiment_runs
WHERE tenant_id = $1 AND status IN ('initializing','generating','validating');

If limit exceeded:
Reject with EXP_429_CONCURRENCY_LIMIT.

---------------------------------------------------------------------

# 7. Compute Rate Governance

Scheduler admission control integrates cost model:

If estimated_cost + active_estimated_cost > quota_limit:
Reject request

GPU throttle:
- Free: GPU disabled
- Pro: max 1 concurrent GPU job
- Enterprise: namespace-level GPU quota

Weighted fair scheduling:
weight = plan_priority

Enterprise tenants receive higher queue priority during cluster contention.

---------------------------------------------------------------------

# 8. Adaptive Backpressure Mechanisms

If cluster CPU utilization > 85%:
- Reduce token refill rate dynamically
- Delay non-critical probe training
- Postpone adaptive regeneration loops

If GPU pool saturation > 90%:
- Queue image/NLP jobs
- Maintain tabular job priority

Backpressure preserves SLA for control-plane operations.

---------------------------------------------------------------------

# 9. Event Bus Rate Controls

Kafka partition key = tenant_id

Consumer lag monitoring:
If lag > threshold:
- Scale consumer replicas
- Temporarily reduce experiment intake

Prevents event replay storm affecting other tenants.

---------------------------------------------------------------------

# 10. Storage & Artifact Access Rate Limits

Signed URL issuance:
- Max N downloads/minute per artifact

Prevents egress abuse.

Large artifact streaming:
- Chunked transfer
- Per-connection bandwidth throttling (enterprise configurable)

---------------------------------------------------------------------

# 11. SLA Monitoring & Alerting

Metrics (Prometheus):
- api_latency_p99
- experiment_runtime_variance
- compliance_score_avg
- reproducibility_failures
- token_bucket_rejections
- concurrency_limit_hits

Alert thresholds:
- API p99 > SLO for 5 consecutive minutes
- Availability drop below SLA target
- Compliance score average < 95%

Alerts routed to on-call rotation.

---------------------------------------------------------------------

# 12. SLA Exclusions & Edge Cases

Not counted as SLA violation:
- Tenant misconfiguration
- Statistical validation failure
- Quota rejection
- Explicit experiment cancellation

Counted as SLA violation:
- Scheduler deadlock
- Cross-tenant RLS failure
- Artifact retrieval outage

---------------------------------------------------------------------

# 13. Multi-Region SLA Extension (Enterprise)

Enterprise optional configuration:

Active-passive failover:
RTO < 15 minutes
RPO < 5 minutes

Active-active (premium tier):
Regional latency-based routing

Isolation invariants preserved via:
- Region-scoped namespaces
- Replicated RLS policies
- Deterministic seed replay capability

---------------------------------------------------------------------

# 14. SLA & Cost Model Alignment

Safety margin buffer:
C_overhead = λ × C_compute

λ depends on plan tier.

Higher SLA tier → larger safety buffer → more reserved capacity.

Cost governance and SLA stability are mathematically linked via scheduler estimation.

---------------------------------------------------------------------

# 15. Formal Reliability Guarantees

For any tenant T:

1. API Isolation Guarantee:
No other tenant’s traffic can reduce T’s API throughput below plan-defined rate limit.

2. Compute Fairness Guarantee:
Allocated compute(T) ≤ quota(T)
AND
Compute(T1) does not consume quota(T2).

3. Deterministic Integrity Guarantee:
SLA compliance never alters dataset semantics.

4. Compliance Guarantee:
Artifact exposed only if compliance_score ≥ 95%.

---------------------------------------------------------------------

# 16. Strategic Outcome

The SLA/SLO and rate limiting model is architecturally integrated rather than layered on top.

It ensures:
- Deterministic experiment execution under load
- Tenant-fair compute allocation
- Predictable cost governance
- Protection of statistical integrity targets
- Enterprise-grade availability guarantees

SLA enforcement, rate limiting, and compute governance collectively transform HackForge AI into a reliability-bounded experimental simulation platform rather than a best-effort dataset generator.

