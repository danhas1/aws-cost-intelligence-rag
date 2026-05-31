# Postmortem: Payment Service Outage — November 15, 2025

**Incident ID:** INC-2025-047
**Severity:** SEV-1
**Status:** Resolved — Action items in progress
**Date of incident:** 2025-11-15
**Duration:** 47 minutes full outage (06:14–07:01 UTC); 2 hours degraded state (07:01–09:10 UTC)
**Services affected:** payment-service (primary), user-service (secondary, elevated error rate)
**Incident commander:** Marcus Rivera
**Lead engineer:** Priya Patel
**Scribe:** Alex Kim
**Postmortem author:** Marcus Rivera
**Postmortem review date:** 2025-11-22
**Postmortem status:** Accepted by Engineering Leadership

---

## Incident Summary

On November 15, 2025 at 06:14 UTC, the payment-service became completely unavailable for 47 minutes, followed by 2 hours of degraded availability. All payment submissions during the full outage window returned HTTP 503. Approximately 11,400 payment attempts failed during the full outage window and an estimated 4,200 additional transactions experienced timeouts or elevated errors during the degraded period.

The incident occurred during Week 5 of the ECS-to-EKS migration. The payment-service migration had begun at 05:30 UTC that morning, earlier than usual to avoid peak traffic hours. A combination of misconfigured resource limits and a missing PodDisruptionBudget caused all payment-service pods to be simultaneously evicted from the EKS cluster within an approximately 90-second window.

**Estimated revenue impact:** $640,000 (based on average order value × failed transaction count during full outage).

---

## Timeline (all times UTC)

| Time | Event |
|---|---|
| 05:30 | Payment-service EKS migration begins. Alex Kim applies Helm chart to `payments` namespace. |
| 05:35 | payment-service Deployment created in EKS. 6 pods scheduled across 3 nodes. |
| 05:38 | ALB target group weight updated: ECS 80%, EKS 20%. Initial traffic split begins. |
| 05:45 | EKS pods pass initial health checks. p99 latency on EKS pods: ~195ms (acceptable). |
| 05:52 | Alex Kim increases ALB weight: ECS 50%, EKS 50%. |
| 06:01 | Node autoscaler scales down `general` node group from 9 nodes to 7 nodes (scheduled scale-down based on prior-night low-traffic window; autoscaler did not account for new payment-service pods). |
| 06:03 | 2 payment-service pods evicted from the scaled-down nodes. 4 pods remain on EKS, 2 entering `Pending` state waiting for node capacity. |
| 06:06 | Payment-service pods on EKS begin reporting elevated memory usage. Investigation later reveals the Helm chart resource limit was set to `memory: 512Mi`, copied from the recommendation-service Helm chart template. Actual payment-service memory usage at 150 RPS: ~900Mi. |
| 06:09 | First payment-service pod on EKS OOMKilled. Kubernetes attempts restart. |
| 06:11 | Second EKS pod OOMKilled. Now 2 of 4 remaining EKS pods are in CrashLoopBackoff. |
| 06:13 | ALB health checks fail for all EKS pods (2 OOMKilled, 2 pending on unschedulable nodes). ALB automatically removes all EKS pods from target group. ECS now carrying 100% of traffic. |
| 06:14 | Cluster Autoscaler begins scaling up new nodes. ECS payment-service pods are now the only targets. |
| 06:14 | **Simultaneously:** Cluster Autoscaler scales down 1 additional underutilized ECS node (a separate node group), triggering ECS task draining for 3 payment-service ECS tasks. |
| 06:14 | ECS drain + EKS failure = 0 healthy payment-service instances. **FULL OUTAGE BEGINS.** |
| 06:15 | PagerDuty alert fires (payment-service error rate 100%). Marcus Rivera acknowledges. |
| 06:18 | Marcus Rivera attempts to roll back ALB weights to ECS 100% / EKS 0%. Finds that ECS tasks are still draining. |
| 06:22 | Priya Patel joins incident bridge. |
| 06:24 | ECS drain completes. 5 of 6 original ECS payment-service tasks are healthy. ALB routes 100% to ECS. |
| 06:25 | Error rate falls from 100% to ~35% as ECS absorbs traffic. Some tasks still warming up. |
| 06:30 | All 6 ECS tasks healthy. Error rate falls to <1%. **Degraded state: EKS pods still OOMKilling in background.** |
| 07:01 | EKS memory limits corrected to `memory: 1Gi` (request) / `memory: 1.5Gi` (limit). New EKS pods scheduled. EKS begins passing health checks. |
| 07:15 | ALB weight: ECS 90%, EKS 10% (cautious re-introduction). |
| 08:00 | ALB weight: ECS 50%, EKS 50%. All EKS pods stable. |
| 09:10 | ALB weight: ECS 0%, EKS 100%. Migration declared complete. |
| 09:15 | **Incident resolved.** |
| 09:20 | Post-incident review meeting scheduled for 2025-11-22. |

---

## Root Causes

### Root Cause 1: Incorrect memory resource limits in Helm chart

The payment-service Helm chart was bootstrapped by copying the recommendation-service chart template. The recommendation-service pods use `memory: 512Mi` because they are lightweight API servers (the heavy model inference is offloaded). The payment-service JVM process, under 150 RPS load with Redis connection pool, Stripe client thread pool, and Spring Boot overhead, requires approximately 900 MB of heap and non-heap memory.

The 512Mi limit was never tested under production load. In staging (load test at 30 RPS), memory usage was ~320Mi — well within limits. The discrepancy between staging load and production load was not caught during migration planning.

When the node autoscaler eviction reduced EKS pod count from 6 to 4 and traffic per pod increased, memory usage crossed the 512Mi limit and pods were OOMKilled.

**Why this wasn't caught:** No resource limit validation was part of the migration runbook. The checklist gate required health checks to pass under light traffic, but did not require a load test against production-representative traffic levels.

### Root Cause 2: Missing PodDisruptionBudget

No PodDisruptionBudget (PDB) was configured for the payment-service Deployment. PDBs guarantee a minimum number of pods remain available during voluntary disruptions (node drains, autoscaler scale-downs). Without a PDB, the Cluster Autoscaler was free to evict all payment-service pods simultaneously during a node scale-down.

Marcus Rivera had explicitly raised the PDB requirement during the ADR-001 review (see ADR-001 review notes). The ADR was accepted with a note that PDBs should be enforced before the migration is declared complete. However, this requirement was not reflected as a blocking checklist item in the migration runbook. The migration proceeded to the payment-service step without PDBs in place.

**Systemic cause:** The migration runbook was written before ADR-001 was finalized. ADR-001 review comments were not backpropagated to the runbook as binding requirements.

### Root Cause 3: Simultaneous autoscaler action on ECS and EKS node groups

The Cluster Autoscaler operates across all node groups in the EKS cluster. During the migration, CloudShop had both EKS and ECS instances in the same VPC, with a separate autoscaling group for ECS. The Cluster Autoscaler for EKS and the ECS capacity provider's autoscaling acted independently and simultaneously, producing a worst-case scenario where both orchestrators scaled down capacity at the same time.

This was a transient configuration state that existed only during the migration window, but it was not anticipated in the migration plan.

---

## Contributing Factors

1. **Migration started at 05:30 UTC (pre-peak, but node autoscaler hadn't completed its nightly scale-down).** If the migration had started after 08:00 UTC when traffic was live and the autoscaler had finished scaling up, the node eviction would not have occurred.

2. **No synthetic transaction monitoring for payment-service on EKS.** Our synthetic monitoring (Datadog Synthetics checkout flow) was configured to target the production endpoint (which routes through ALB). It did not independently test the EKS pods before they received production traffic.

3. **Memory limits copied from an unrelated service.** Platform Engineering did not have a standardized memory limit baseline or sizing guide. Each service team was expected to determine appropriate resource requests and limits independently, without tooling or process support.

---

## Impact

| Metric | Value |
|---|---|
| Full outage duration | 47 minutes (06:14–07:01 UTC) |
| Degraded state duration | 2 hours (07:01–09:10 UTC) |
| Failed payment transactions | ~11,400 |
| Timed-out / degraded transactions | ~4,200 |
| Estimated revenue impact | $640,000 |
| Affected customers (unique) | ~8,900 |
| Customer support tickets opened | 312 (payment failure, order not placed) |
| SLO impact | 99.95% availability SLO breached for November 2025 |

---

## What Went Well

1. **Rollback path worked.** The dual-running ECS/EKS architecture (per the migration plan) allowed recovery by shifting traffic back to ECS without redeploying. Once the ECS drain completed, recovery was fast.

2. **On-call response was fast.** PagerDuty alert fired within 1 minute of full outage. Incident commander acknowledged within 2 minutes. All required engineers on the bridge within 8 minutes.

3. **Communication was clear.** Status page was updated within 5 minutes of the outage start. Internal stakeholders (Support, Customer Success, Finance) were notified via the incident Slack channel within 10 minutes.

4. **No data loss.** Redis idempotency key caching and the PostgreSQL write-through pattern ensured that no payments were double-charged. All failed transactions received clean 503 responses; no partial payments were recorded.

---

## Action Items

| # | Action | Owner | Status | Due Date |
|---|---|---|---|---|
| 1 | Mandate PodDisruptionBudgets (`maxUnavailable: 1`) for all services in `payments`, `users`, and `recommendations` namespaces | Sarah Chen | **Done** (2025-11-20) | 2025-11-20 |
| 2 | Add PDB existence check as a blocking gate in migration runbook | Marcus Rivera | **Done** (2025-11-25) | 2025-11-25 |
| 3 | Create standardized resource request/limit baselines by service tier (CPU-bound, memory-bound, JVM-based) | Sarah Chen | **Done** (2025-12-05) | 2025-12-05 |
| 4 | Add migration runbook gate: load test at ≥80% production RPS before increasing ALB weight above 20% | Marcus Rivera | **Done** (2025-11-25) | 2025-11-25 |
| 5 | Add synthetic monitoring for each service's EKS pods (pre-production traffic validation) | Alex Kim | **Done** (2025-12-01) | 2025-12-01 |
| 6 | Coordinate ECS/EKS autoscaler schedules during migration windows to prevent simultaneous scale-downs | Alex Kim | **Done** (2025-11-25) | 2025-11-25 |
| 7 | Add ADR review comment backpropagation process: ADR comments flagged as requirements must create Jira tickets | Emily Torres | **In Progress** | 2026-01-15 |
| 8 | Conduct internal training: "Kubernetes Resource Management for Service Teams" | Sarah Chen | **Done** (2026-01-10) | 2026-01-10 |

---

## Systemic Observations

This incident was not caused by a single mistake. It was caused by the interaction of three independent gaps, none of which was sufficient to cause an outage on its own:

- A resource limit error that would have been caught by a load test.
- A missing PodDisruptionBudget that was called out in a code review but not enforced.
- A migration plan that did not account for simultaneous autoscaler actions across two orchestrators.

The deeper issue is that CloudShop's migration process for critical, PCI-DSS-scoped services was not sufficiently rigorous for the payment-service's risk profile. The same process that worked acceptably for recommendation-service and user-service was not appropriate for payment-service without additional gates.

Going forward, services touching payment data will require a dedicated pre-production load test, a security sign-off, and an explicit resource limit review before any infrastructure change that increases their blast radius.

---

## Related Documents

- [ADR-001: Migrate to EKS](../adrs/ADR-001-migrate-to-eks.md) — includes Marcus Rivera's PDB requirement in review notes
- [migrations/ecs-to-eks-migration.md](../migrations/ecs-to-eks-migration.md) — full migration plan; updated post-incident with additional gates
- [runbooks/payment-service-runbook.md](../runbooks/payment-service-runbook.md) — updated with EKS-specific triage steps
