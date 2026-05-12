# HackForge AI – Multi-Tenant Isolation Model (Expanded & Cross-Integrated)

This document formalizes the end-to-end multi-tenant isolation strategy for HackForge AI and tightly integrates with:

- PRD (deterministic reproducibility & tiered monetization) fileciteturn0file2
- Technical Architecture Blueprint (control plane vs compute plane separation) fileciteturn0file0
- Microservices API Contract (tenant-scoped headers, JWT enforcement, idempotency) fileciteturn0file1
- Internal Data Models (Tenant, Experiment, Artifact, AuditLog invariants) fileciteturn0file3
- Engineering Roadmap (Phase 6 – Production Hardening) fileciteturn0file5
- Mathematical Definitions (config_hash + seed determinism invariant) fileciteturn0file6
- Infrastructure Cost Estimation Model (quota & resource governance) fileciteturn0file7
- Security & Compliance Deep Dive (RLS, mTLS, artifact integrity) fileciteturn0file8
- PostgreSQL Schema & Indexing Model (partitioning, RLS policies) fileciteturn0file9

Multi-tenancy in HackForge AI is not merely logical scoping. It is a deterministic, cryptographically bound, compute-isolated, cost-governed boundary enforced across identity, metadata, storage, runtime execution, and event propagation layers.

------------------------------------------------------------

# 1. Isolation Objectives

The isolation model must guarantee:

1. No cross-tenant data visibility.
2. No cross-tenant compute resource interference beyond plan-defined limits.
3. No deterministic reproducibility leakage (seed + config protection).
4. No cross-tenant artifact collision.
5. Strict cost attribution and quota enforcement per tenant.
6. Isolation invariant preserved under horizontal scaling.

Isolation must remain valid even under:
- Concurrent experiment execution
- High-throughput event streaming
- Multi-region deployments

------------------------------------------------------------

# 2. Isolation Layers Overview

Isolation is implemented across seven layers:

1. Identity Isolation
2. API Boundary Isolation
3. Database Isolation
4. Storage Isolation
5. Compute Isolation
6. Event Bus Isolation
7. Cost & Quota Isolation

Each layer enforces tenant_id as a non-optional boundary condition.

------------------------------------------------------------

# 3. Identity Isolation (IAM-Level)

## 3.1 Tenant-Bound JWT Claims

All external requests must include:
- Authorization: Bearer <JWT>
- X-Tenant-ID header

JWT payload must contain:
- tenant_id
- role
- exp
- jti

Validation rule:

assert(request.tenant_id == jwt.tenant_id)

Failure results in immediate 403 rejection at API Gateway.

## 3.2 Role Constrained Actions

RBAC matrix enforced before business logic execution:

owner → billing + deletion
admin → experiment lifecycle
member → create & download
viewer → read-only

Isolation property:
Users cannot enumerate tenant IDs. Tenant scope is inferred strictly from JWT.

------------------------------------------------------------

# 4. API Boundary Isolation

Per API Contract:
- All endpoints require X-Tenant-ID.
- Idempotency-Key is tenant-scoped.
- Rate limiting uses token bucket per tenant.

Rate bucket definition:

bucket_key = hash(tenant_id)

Concurrency cap applied before scheduler allocation.

Prevents one tenant from saturating experiment-service queue.

------------------------------------------------------------

# 5. Database Isolation (Metadata Plane)

## 5.1 Row-Level Security (RLS)

All tenant-scoped tables implement:

USING (tenant_id = current_setting('app.current_tenant')::uuid)

Connection bootstrap step:

SET app.current_tenant = '<tenant_uuid>';

No service query permitted without tenant context.

## 5.2 Partitioning by tenant_id

Experiments table:
PARTITION BY HASH (tenant_id)

Properties:
- Query pruning at planner level
- Reduced index bloat
- Improved vacuum locality

## 5.3 Deterministic Reproducibility Constraint

UNIQUE (tenant_id, config_hash, seed, version)

This ensures:
Two tenants may share identical config_hash, but identity remains tenant-bound.

------------------------------------------------------------

# 6. Storage Isolation (Artifact Plane)

Artifact path structure:

/tenant_id/experiment_id/artifact_id

Checksum invariant:
UNIQUE(checksum_sha256)

Collision protection:
Even if two tenants generate identical dataset bytes, access path remains tenant-scoped.

Signed URL properties:
- Time-limited
- Tenant-bound validation before generation
- Optional IP binding (enterprise)

Artifact metadata includes:
- tenant_id
- experiment_id
- config_hash
- seed

------------------------------------------------------------

# 7. Compute Isolation (Execution Plane)

Isolation varies by plan tier.

## 7.1 Free / Pro Tier

- Logical isolation via run_id namespace
- ResourceQuota per tenant
- Scheduler ensures max concurrent runs

## 7.2 Enterprise Tier

- Kubernetes namespace per tenant
- NetworkPolicy blocking cross-namespace traffic
- Dedicated service account per tenant
- Optional node pool segregation

Compute namespace binding:

compute_namespace = hash(tenant_id || environment)

## 7.3 Deterministic RNG Segregation

Per mathematical definition:

R = SHA256(config_hash || seed)
RNG_i = PRNG(R ⊕ service_namespace_i ⊕ tenant_id)

Tenant_id inclusion prevents cross-tenant RNG state collision.

------------------------------------------------------------

# 8. Event Bus Isolation

Kafka partition key:

key = tenant_id

Topic example:
experiment.created

Message schema includes:
- experiment_id
- tenant_id
- version

Consumers must verify tenant_id before processing.

Prevents replay from another tenant context.

------------------------------------------------------------

# 9. Cost & Quota Isolation

Scheduler pre-run validation:

if estimated_cost > tenant.quota_config.limit:
    reject_request

ResourceAllocation stored with tenant_id.

Quota enforcement layers:
1. API rate limit
2. Scheduler admission control
3. Kubernetes ResourceQuota
4. Post-run reconciliation audit

Cross-tenant compute starvation prevention:
Weighted fair scheduling by tenant plan.

------------------------------------------------------------

# 10. Observability Isolation

Metrics labeled with:
- tenant_id
- experiment_id

Grafana dashboards enforce tenant filtering.

AuditLog table is tenant-scoped via RLS.

Log redaction policy ensures:
- Seed masked unless privileged
- Billing IDs restricted

------------------------------------------------------------

# 11. Failure & Incident Containment

If S2 (cross-tenant breach suspected):

1. Freeze affected tenant namespace
2. Rotate service certificates
3. Lock artifact downloads
4. Audit event replay
5. Verify RLS policy integrity

Isolation recovery includes checksum verification across artifact store.

------------------------------------------------------------

# 12. Multi-Region & DR Isolation

In multi-region mode:

/region/tenant_id/experiment_id/

RLS and namespace mapping replicated.

Reproducibility invariant:
(config_hash, seed, tenant_id) → identical artifact checksum

Isolation preserved even during failover.

------------------------------------------------------------

# 13. Cross-Layer Isolation Invariants

Invariant 1:
No query executes without tenant context.

Invariant 2:
All artifacts map to exactly one tenant.

Invariant 3:
Compute resources allocated under tenant quota.

Invariant 4:
Deterministic reproduction never crosses tenant boundary.

Invariant 5:
Cost attribution is bijective:
run_id → tenant_id → billing record

------------------------------------------------------------

# 14. Formal Isolation Guarantee Statement

For any two tenants T1 ≠ T2:

Let D(T, H, S) denote generated dataset.

Guarantees:

1. Metadata isolation:
∀ rows r ∈ DB: r.tenant_id ∈ {T1} is invisible to T2.

2. Storage isolation:
No storage_uri(T1) accessible to T2 without cryptographic violation.

3. Compute isolation:
CPU/GPU allocation(T1) cannot exceed quota(T1) nor consume quota(T2).

4. Deterministic isolation:
D(T1, H, S) = D(T2, H, S) only if both independently configure identical values;
identity and access remain tenant-scoped.

------------------------------------------------------------

# 15. Strategic Outcome

The multi-tenant isolation model converts tenant_id from a simple column attribute into a first-class architectural boundary embedded in:

- Identity tokens
- API enforcement
- PostgreSQL RLS policies
- Partitioning strategy
- Object storage paths
- RNG derivation
- Scheduler cost admission
- Kubernetes namespaces
- Event streaming keys
- Billing attribution

This ensures HackForge AI operates as a deterministic, statistically rigorous, causally consistent experimental simulation engine while maintaining strict, defense-in-depth tenant separation across all planes of execution.

The isolation model is therefore mathematically aligned, architecturally enforced, financially governed, and operationally hardened for enterprise-scale deployment.

