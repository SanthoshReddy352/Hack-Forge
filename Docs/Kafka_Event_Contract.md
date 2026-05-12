# HackForge AI – Kafka Event Contracts (Expanded & Cross-Integrated)

This document formalizes the complete Kafka event contract layer for HackForge AI and tightly integrates with:

- PRD (deterministic reproducibility, statistical compliance guarantees) fileciteturn0file2  
- Technical Architecture Blueprint (event-driven control plane ↔ compute plane separation) fileciteturn0file0  
- Microservices API Contract (service boundaries, async lifecycle) fileciteturn0file1  
- Internal Data Models (Experiment, ExperimentRun, DistributionReport, DatasetArtifact) fileciteturn0file3  
- Engineering Roadmap (Phase 2 event bus integration) fileciteturn0file5  
- Mathematical Definitions (config_hash + seed determinism invariant) fileciteturn0file6  
- Cost Estimation Model (scheduler admission + reconciliation loop) fileciteturn0file7  
- Security & Compliance (tenant isolation, RLS, mTLS) fileciteturn0file8  
- PostgreSQL Schema & Indexing (metadata persistence) fileciteturn0file9  
- Multi-Tenant Isolation Model (tenant-scoped partitioning & keys) fileciteturn0file10  
- SLA/SLO & Rate Limiting (backpressure & fairness guarantees) fileciteturn0file11  

Kafka is the backbone of the asynchronous experiment lifecycle. All cross-service state transitions occur via immutable, versioned, tenant-scoped events.

---------------------------------------------------------------------

# 1. Event Architecture Principles

1. All events are immutable.
2. All events are tenant-scoped.
3. Event schemas are versioned and registry-enforced.
4. Event ordering is guaranteed per tenant_id key.
5. No service performs synchronous cross-service state mutation.
6. Events are idempotent and replay-safe.

Kafka partitions use:

partition_key = tenant_id

This ensures ordering guarantees within tenant boundaries while preserving horizontal scalability.

---------------------------------------------------------------------

# 2. Topic Taxonomy

Topics are grouped by lifecycle domain.

## 2.1 Experiment Lifecycle Topics

- experiment.created
- experiment.validated
- experiment.rejected
- experiment.scheduled
- experiment.started
- experiment.progress.updated
- experiment.failed
- experiment.completed
- experiment.cancelled

## 2.2 Statistical & Validation Topics

- distribution.validation.completed
- correlation.analysis.completed
- difficulty.calculated
- compliance.evaluated

## 2.3 Failure Injection Topics

- failure.injection.applied
- drift.simulation.completed

## 2.4 Artifact & Storage Topics

- artifact.packaging.started
- artifact.ready
- artifact.download.requested

## 2.5 Cost & Resource Topics

- resource.estimated
- resource.allocated
- resource.released
- cost.reconciled

## 2.6 Observability & Governance Topics

- audit.logged
- sla.breach.detected
- quota.limit.reached

---------------------------------------------------------------------

# 3. Canonical Event Envelope

All events follow a uniform envelope structure.

{
  "event_id": "UUID",
  "event_type": "string",
  "event_version": "v1",
  "tenant_id": "UUID",
  "correlation_id": "UUID",
  "causation_id": "UUID",
  "timestamp": "ISO8601",
  "producer_service": "string",
  "payload": { }
}

Field Definitions:

- event_id: unique per event (idempotency anchor)
- correlation_id: traces entire experiment lifecycle
- causation_id: previous triggering event
- event_version: additive-only schema evolution

Schema Registry:
- Backward compatible changes only
- Additive fields permitted
- Breaking changes require new topic version

---------------------------------------------------------------------

# 4. Core Lifecycle Event Definitions

## 4.1 experiment.created

Producer: experiment-service  
Consumer: validation-service

Payload:

{
  "experiment_id": "UUID",
  "tenant_id": "UUID",
  "config_hash": "SHA256",
  "seed": "BIGINT",
  "version": 1,
  "row_count": "BIGINT",
  "feature_count": "INT"
}

Invariant:
config_hash must match ExperimentConfig snapshot.

---------------------------------------------------------------------

## 4.2 experiment.validated

Producer: validation-service  
Consumer: scheduler-service

Payload:

{
  "experiment_id": "UUID",
  "validation_passed": true,
  "warnings": [],
  "validated_at": "timestamp"
}

If validation_passed = false → experiment.rejected

---------------------------------------------------------------------

## 4.3 resource.estimated

Producer: scheduler-service

{
  "experiment_id": "UUID",
  "cpu_cores": 8,
  "memory_gb": 32,
  "gpu_required": false,
  "estimated_runtime_seconds": 240,
  "estimated_cost_usd": 3.75
}

Deterministic requirement:
Estimate must be pure function of ExperimentConfig.

---------------------------------------------------------------------

## 4.4 experiment.started

Producer: scheduler-service

{
  "run_id": "UUID",
  "experiment_id": "UUID",
  "compute_namespace": "string",
  "started_at": "timestamp"
}

---------------------------------------------------------------------

## 4.5 experiment.progress.updated

Producer: compute-plane services

{
  "run_id": "UUID",
  "stage": "generation | validation | injection | packaging",
  "progress_percentage": 65
}

---------------------------------------------------------------------

## 4.6 distribution.validation.completed

Producer: statistical-engine

{
  "run_id": "UUID",
  "feature_id": "UUID",
  "ks_statistic": 0.03,
  "p_value": 0.12,
  "passed": true
}

Used to compute compliance_score.

---------------------------------------------------------------------

## 4.7 difficulty.calculated

Producer: difficulty-engine

{
  "run_id": "UUID",
  "linear_separability_score": 0.72,
  "noise_ratio": 0.31,
  "final_difficulty_index": 0.64
}

---------------------------------------------------------------------

## 4.8 experiment.completed

Producer: orchestrator

{
  "run_id": "UUID",
  "experiment_id": "UUID",
  "compliance_score": 0.97,
  "artifact_checksum": "SHA256"
}

Artifact exposure allowed only if compliance_score ≥ 0.95.

---------------------------------------------------------------------

## 4.9 artifact.ready

Producer: artifact-service

{
  "artifact_id": "UUID",
  "run_id": "UUID",
  "storage_uri": "s3://...",
  "checksum_sha256": "SHA256",
  "size_bytes": 1024000
}

Checksum must match DatasetArtifact constraint.

---------------------------------------------------------------------

# 5. Idempotency & Replay Semantics

Consumer Rules:

1. If event_id already processed → ignore.
2. State transitions must be monotonic.
3. Out-of-order events rejected if causation_id invalid.

Replay Safety:

System must tolerate full topic replay without:
- Duplicate artifacts
- Duplicate billing entries
- Seed mutation

Idempotency anchors:
- experiment_id
- run_id
- artifact_checksum

---------------------------------------------------------------------

# 6. Event Ordering Guarantees

Ordering scope:
- Guaranteed per tenant_id partition
- Not guaranteed globally

Implication:
Cross-tenant lifecycle ordering is irrelevant by design.

---------------------------------------------------------------------

# 7. Security Controls on Events

1. mTLS between producers and brokers.
2. ACL rules per service identity.
3. Event payload must include tenant_id.
4. Consumers verify tenant_id before processing.
5. Sensitive fields (seed) masked in audit events.

---------------------------------------------------------------------

# 8. Backpressure & SLA Integration

If consumer lag exceeds threshold:
- Reduce experiment intake rate
- Emit quota.limit.reached
- Trigger autoscaling of consumers

Protects API latency SLO and compute fairness guarantees.

---------------------------------------------------------------------

# 9. Dead Letter Queues (DLQ)

Each topic has corresponding DLQ:

experiment.created.dlq
experiment.started.dlq
artifact.ready.dlq

DLQ retention:
- 7–30 days configurable

DLQ events require manual reconciliation.

---------------------------------------------------------------------

# 10. Cross-Region & DR Replication

Enterprise mode:
- MirrorMaker replication
- Region-prefixed topic namespace

Replay invariant:
(config_hash, seed, tenant_id) → identical artifact checksum

---------------------------------------------------------------------

# 11. Event Lifecycle State Machine (Formal Model)

Let E be experiment.

E_created → E_validated → E_scheduled → E_started →
{E_failed | E_completed}

Transitions allowed only via event emission.

No direct DB mutation allowed outside consumer logic.

---------------------------------------------------------------------

# 12. Strategic Outcome

The Kafka event contract transforms HackForge AI into a fully asynchronous, tenant-isolated, replay-safe experimental simulation system.

It ensures:
- Deterministic lifecycle orchestration
- Stateless microservice boundaries
- Cost-aware scheduling feedback
- SLA-compliant backpressure
- Cryptographically verifiable artifact emission
- Strict cross-tenant ordering guarantees

The event bus is therefore not a transport layer but the authoritative state transition backbone of the distributed experimental engine.

