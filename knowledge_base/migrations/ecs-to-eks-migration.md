# ECS to EKS Migration Plan & Execution Log

**Document type:** Migration Plan + Execution Log
**Status:** Complete
**Migration owner:** Sarah Chen (Platform Engineering Lead)
**Migration period:** 2025-11-03 to 2025-12-19
**Original planned completion:** 2025-12-05 (8 weeks from plan approval 2025-10-08)
**Actual completion:** 2025-12-19 (11 weeks; 3-week slip due to payment-service outage and replanning)
**Related ADR:** [ADR-001: Migrate to EKS](../adrs/ADR-001-migrate-to-eks.md)
**Postmortem:** [Payment Outage — Nov 2025](../incidents/payment-outage-postmortem.md)

---

## Migration Objectives

1. Move all three CloudShop production services (recommendation-service, user-service, payment-service) from Amazon ECS to Amazon EKS.
2. Maintain zero data loss throughout migration.
3. Maintain ≥99.9% availability for user-service and recommendation-service; ≥99.95% for payment-service.
4. Complete with no customer-observable degradation beyond planned maintenance windows.
5. Leave the ECS infrastructure in a decommissionable state by December 19, 2025.

---

## Migration Architecture

### Dual-Running Strategy

Each service will run simultaneously on both ECS and EKS during its migration window. An AWS Application Load Balancer (ALB) with weighted target groups routes a configurable percentage of traffic to each orchestrator.

```
                     ┌─────────────────────────────────────────┐
                     │         AWS Application Load Balancer     │
                     └────────────┬────────────────┬────────────┘
                                  │                │
                           Weight: X%        Weight: (100-X)%
                                  │                │
                     ┌────────────▼───┐    ┌───────▼────────────┐
                     │  ECS Target    │    │  EKS Target        │
                     │  Group         │    │  Group             │
                     │  (old)         │    │  (new)             │
                     └────────────────┘    └────────────────────┘
```

Traffic weight starts at ECS: 100%, EKS: 0%. As confidence grows, weight shifts toward EKS until ECS: 0%, EKS: 100%, at which point ECS tasks are drained and decommissioned.

**Rollback:** Shift ALB weight back to ECS 100% / EKS 0% at any time. No redeployment required. Maximum rollback time: ~2 minutes (ALB weight update + health check propagation).

### Migration Gate Criteria

Before increasing ALB traffic weight above 20%, each service must pass:

1. **Health checks pass** on all EKS pods for ≥10 minutes under load.
2. **p99 latency within 5%** of the ECS baseline measured over the same period.
3. **Error rate below 0.1%** on EKS-routed requests.
4. **Load test completed** at ≥80% of production peak RPS (gate added post-Nov-15 incident).
5. **PodDisruptionBudget configured** with `maxUnavailable: 1` (gate added post-Nov-15 incident).
6. **Resource limits validated** against production-representative load (gate added post-Nov-15 incident).

Before declaring a service fully migrated:

7. **72-hour observation window** at EKS 100% with no SLO breach.
8. **Runbook updated** to reflect EKS-specific procedures.
9. **Alerts confirmed firing** correctly in the new EKS namespace (not ECS).
10. **Security sign-off** for payment-service (PCI-DSS scope review).

---

## Service Migration Order

**Phase 1: recommendation-service** (lowest risk, no PCI-DSS scope, no synchronous dependency)
**Phase 2: user-service** (medium risk, upstream dependency of payment-service)
**Phase 3: payment-service** (highest risk, PCI-DSS scope, directly handles payment data)

---

## Phase 1: recommendation-service Migration

**Migration window:** 2025-11-03 to 2025-11-14
**Owner:** James Okonkwo (application), Alex Kim (infrastructure)

### Execution Log

| Date | Action | Result |
|---|---|---|
| 2025-11-03 | EKS Deployment created for recommendation-service in `recommendations` namespace | Pods healthy |
| 2025-11-03 | ALB weight: ECS 95% / EKS 5% | Latency and error rate nominal |
| 2025-11-05 | Load test at 200 RPS (production peak ~180 RPS) against EKS pods | p99: 420ms (ECS baseline: 410ms — within 5%) |
| 2025-11-05 | ALB weight: ECS 80% / EKS 20% | Nominal |
| 2025-11-07 | ALB weight: ECS 50% / EKS 50% | Nominal |
| 2025-11-10 | ALB weight: ECS 0% / EKS 100% | Nominal |
| 2025-11-10 | 72-hour observation window begins | No SLO breach |
| 2025-11-13 | 72-hour window complete | All gates passed |
| 2025-11-14 | ECS tasks for recommendation-service drained and stopped | Migration complete |

**Phase 1 result:** Complete. No incidents. Elapsed: 11 days (planned: 10 days).

---

## Phase 2: user-service Migration

**Migration window:** 2025-11-10 to 2025-11-14
**Owner:** Backend Platform team, Alex Kim (infrastructure)

### Execution Log

| Date | Action | Result |
|---|---|---|
| 2025-11-10 | EKS Deployment created for user-service in `users` namespace | Pods healthy |
| 2025-11-10 | ALB weight: ECS 95% / EKS 5% | Nominal |
| 2025-11-11 | Load test at 400 RPS (production peak ~350 RPS) | p99: 160ms (ECS baseline: 155ms — within 5%) |
| 2025-11-11 | ALB weight: ECS 80% / EKS 20% | Nominal |
| 2025-11-12 | ALB weight: ECS 50% / EKS 50% | Nominal |
| 2025-11-13 | ALB weight: ECS 0% / EKS 100% | Nominal |
| 2025-11-13 | 72-hour observation window begins | No SLO breach |
| 2025-11-14 (partial) | Redis JWT blacklist verified functioning correctly on EKS pods | Confirmed |
| 2025-11-16 | 72-hour window complete | All gates passed |
| 2025-11-17 | ECS tasks for user-service drained and stopped | Migration complete |

**Phase 2 result:** Complete. No incidents. Elapsed: 7 days (planned: 7 days).

---

## Phase 3: payment-service Migration

**Migration window (original):** 2025-11-15 to 2025-11-25
**Migration window (actual, post-incident):** 2025-11-29 to 2025-12-19
**Owner:** Priya Patel (application), Alex Kim (infrastructure), Sarah Chen (migration lead)

### Phase 3 Execution Log — Attempt 1 (2025-11-15, FAILED)

| Time (UTC) | Action | Result |
|---|---|---|
| 05:30 | EKS Deployment created (`payments` namespace) with `memory: 512Mi` limit (copied from recommendation-service template — DEFECT) | Pods scheduled |
| 05:38 | ALB weight: ECS 80% / EKS 20% | Nominal (low traffic hour) |
| 06:01 | Cluster Autoscaler scales down 2 ECS-adjacent nodes | 2 payment-service EKS pods evicted |
| 06:06 | Remaining EKS pods begin OOMKilling under increased per-pod load (512Mi limit exceeded) | Pods crashing |
| 06:13 | All EKS pods unhealthy; ALB removes from target group | ECS carrying 100% |
| 06:14 | ECS autoscaler simultaneously scales down 1 ECS node, draining 3 ECS tasks | **0 healthy payment-service instances. OUTAGE BEGINS.** |
| 06:24 | ECS drain completes; 5 ECS tasks healthy | Partial recovery |
| 07:01 | EKS memory limits corrected to `1Gi/1.5Gi`; new pods scheduled | Stable |
| 09:10 | Full traffic restored to EKS | — |
| 09:15 | Incident resolved. Migration attempt suspended. | — |

**Outcome:** Migration suspended. Payment-service returned to ECS 100%. Incident postmortem initiated. Migration paused for 2 weeks pending investigation and process remediation.

See full incident details: [incidents/payment-outage-postmortem.md](../incidents/payment-outage-postmortem.md)

### Post-Incident Process Changes (implemented before Attempt 2)

The following changes were made to the migration runbook and process before Phase 3 was retried:

1. **PodDisruptionBudget added** to payment-service Deployment: `minAvailable: 4` (out of 6 replicas). Prevents cluster autoscaler from evicting more than 2 pods simultaneously.

2. **Resource limits validated against production load:** Priya ran the payment-service under 180 RPS load in the staging EKS cluster and observed JVM memory usage peaking at 920 Mi. Resource limits set to `requests: memory: 1Gi, cpu: 500m`; `limits: memory: 1.5Gi, cpu: 2000m`.

3. **Autoscaler coordination:** ECS and EKS autoscaler scale-down schedules coordinated to not overlap during migration windows. ECS capacity provider scale-in protection enabled for payment-service tasks during migration.

4. **Migration window changed** to 09:00–17:00 UTC (peak traffic hours) so that autoscaler scale-up pressure prevents simultaneous scale-down events.

5. **Load test gate added:** migration runbook now requires a successful load test at 200 RPS (≥80% of peak) for ≥15 minutes before ALB weight exceeds 20%.

6. **Security review completed:** PCI-DSS scope review with security team confirmed EKS network isolation (NetworkPolicy, IRSA) is equivalent to ECS security group configuration.

### Phase 3 Execution Log — Attempt 2 (2025-11-29, SUCCESSFUL)

| Date | Action | Result |
|---|---|---|
| 2025-11-29 | PDB configured. Resource limits set to `1Gi / 1.5Gi`. Pre-flight checklist reviewed. | Ready |
| 2025-11-29 | EKS Deployment created. All 6 pods scheduled. | Healthy |
| 2025-11-29 | Load test: 200 RPS for 20 minutes against EKS pods. p99: 175ms. | Pass (ECS baseline: 180ms) |
| 2025-11-29 | ALB weight: ECS 80% / EKS 20% | Nominal |
| 2025-12-01 | ALB weight: ECS 60% / EKS 40% | Nominal |
| 2025-12-03 | ALB weight: ECS 40% / EKS 60% | Nominal |
| 2025-12-05 | ALB weight: ECS 20% / EKS 80% | Nominal |
| 2025-12-08 | ALB weight: ECS 0% / EKS 100% | Nominal |
| 2025-12-08 | 72-hour observation window begins. | — |
| 2025-12-11 | 72-hour window complete. p99: 175ms average, no SLO breach. | All gates passed |
| 2025-12-12 | Security sign-off received from CISO. | Confirmed |
| 2025-12-12 | Runbook updated for EKS procedures. | Done |
| 2025-12-12 | ECS tasks for payment-service drained and stopped. | Migration complete |

**Phase 3 result:** Complete. Migration successful on second attempt. Elapsed (attempt 2): 13 days. Total elapsed including pause: 27 days.

---

## ECS Decommission

| Date | Action |
|---|---|
| 2025-12-12 | ECS tasks stopped for all services |
| 2025-12-15 | ECS services deleted (Terraform destroy) |
| 2025-12-17 | ECS cluster deleted |
| 2025-12-19 | ECS-related IAM roles, task definitions, and CloudWatch log groups archived/deleted |
| 2025-12-19 | **Migration declared complete.** |

**Cost impact of decommission:** Estimated $3,800/month savings from ECS EC2 instance reduction, partially offset by EKS management fee ($72/month) and slightly higher EKS node costs. Net savings: ~$3,100/month.

---

## Lessons Learned

1. **Never copy resource limits from an unrelated service.** Resource limit baselines must be validated against production-representative load for each service, not inherited from templates.

2. **PodDisruptionBudgets are not optional for critical services.** They must be a blocking pre-flight check, not a best-practice recommendation.

3. **ADR review comments that represent requirements must become Jira tickets or runbook checklist items.** Comments in an ADR are not sufficient to ensure follow-through.

4. **Migration windows for critical services should occur during peak traffic.** Low-traffic windows cause autoscalers to make scale-down decisions that are destructive when traffic increases.

5. **The dual-running architecture worked as a rollback mechanism.** ECS was available as a recovery target throughout the migration. The 47-minute outage would have been hours longer without the rollback path.
