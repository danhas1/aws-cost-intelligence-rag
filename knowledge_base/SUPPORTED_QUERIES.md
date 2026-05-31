# Supported Queries — AI Engineering Organizational Memory Assistant

The following are 20 realistic questions that the RAG assistant should be able to answer using the documents in this knowledge base. Each question reflects the kind of institutional knowledge that is typically lost as teams grow and engineers change roles or leave the company.

These questions are organized by category. They are intended both as acceptance criteria for the RAG system and as demonstrations of value to stakeholders.

---

## Architecture & Design Decisions

**1. Why did CloudShop migrate from ECS to EKS?**
> Expected sources: `adrs/ADR-001-migrate-to-eks.md`, `migrations/ecs-to-eks-migration.md`
> The answer should cover the technical limitations of ECS (lack of native service mesh, coarse-grained scheduling, limited RBAC), the growth in service count that made ECS operationally difficult, and the strategic alignment with the broader AWS/CNCF ecosystem.

**2. What alternatives to Kubernetes were considered before adopting EKS?**
> Expected sources: `adrs/ADR-001-migrate-to-eks.md`
> The answer should describe that HashiCorp Nomad and managed ECS with App Mesh were evaluated, and explain why each was rejected.

**3. Why was Redis chosen as the caching layer instead of Memcached or DynamoDB DAX?**
> Expected sources: `adrs/ADR-002-adopt-redis.md`, `slack/redis-discussion.md`
> The answer should explain that Memcached lacks native support for data structures needed by the recommendation-service, and that DAX is DynamoDB-specific, whereas Redis gave a single unified cache tier across all three services.

**4. What problem was Redis specifically adopted to solve in the payment service?**
> Expected sources: `adrs/ADR-002-adopt-redis.md`, `jira/JIRA-101-payment-latency.md`, `slack/redis-discussion.md`
> The answer should trace the payment-service p99 latency regression in July 2025, the root cause being repeated PostgreSQL queries for idempotency key lookups, and how Redis idempotency key caching resolved it.

**5. What is the current architecture of CloudShop's core services?**
> Expected sources: `architecture/current_architecture.md`
> The answer should describe the EKS cluster layout, service communication patterns (internal gRPC, external REST), datastores per service, and the shared Redis cluster.

---

## Incidents & Postmortems

**6. What caused the payment service outage in November 2025?**
> Expected sources: `incidents/payment-outage-postmortem.md`, `migrations/ecs-to-eks-migration.md`
> The answer should describe the misconfigured resource limits on the payment-service Deployment during the EKS migration that caused OOMKilled pods, combined with a missing PodDisruptionBudget that allowed all replicas to be evicted simultaneously.

**7. What was the customer impact of the November 2025 payment outage?**
> Expected sources: `incidents/payment-outage-postmortem.md`
> The answer should include the duration (47 minutes of full outage, 2 hours of degraded state), estimated failed transactions, and revenue impact.

**8. What action items came out of the November 2025 payment outage postmortem?**
> Expected sources: `incidents/payment-outage-postmortem.md`
> The answer should list the specific action items: PodDisruptionBudgets for all critical services, resource limit standardization via Helm chart defaults, enhanced synthetic monitoring, and a migration runbook checklist gate.

**9. Why did S3 storage costs spike in Q1 2026, and what was done about it?**
> Expected sources: `incidents/storage-cost-spike-postmortem.md`, `jira/JIRA-102-storage-growth.md`
> The answer should explain that the recommendation-service began writing full model snapshots to S3 on every training run without a lifecycle policy, and that the fix included an S3 Intelligent-Tiering policy, a 90-day lifecycle rule, and a model versioning strategy.

**10. Which team was responsible for the storage cost spike, and how was ownership resolved?**
> Expected sources: `incidents/storage-cost-spike-postmortem.md`, `jira/JIRA-102-storage-growth.md`
> The answer should note that the recommendation-service team owned the S3 write path but that no one had established a cost ownership policy for ML artifacts; the postmortem resulted in a formal cost ownership assignment.

---

## Engineering Process & Migrations

**11. How long did the ECS to EKS migration take, and was it on schedule?**
> Expected sources: `migrations/ecs-to-eks-migration.md`
> The answer should explain the planned 8-week timeline, the actual 11-week completion, and that the payment-service outage in Week 5 caused a 2-week pause and replanning phase.

**12. What was the rollback strategy during the ECS to EKS migration?**
> Expected sources: `migrations/ecs-to-eks-migration.md`
> The answer should describe the dual-running period where both ECS and EKS served traffic behind a weighted ALB target group, enabling a fast rollback by shifting traffic weights.

**13. What validation steps were required before a service was declared fully migrated to EKS?**
> Expected sources: `migrations/ecs-to-eks-migration.md`
> The answer should list the migration gate criteria: 72-hour observation window, p99 latency within 5% of ECS baseline, error rate below 0.1%, runbook updated, and alerts confirmed firing in the new namespace.

---

## Operational Knowledge

**14. How should an on-call engineer respond to a payment-service high-latency alert?**
> Expected sources: `runbooks/payment-service-runbook.md`
> The answer should walk through the triage steps: check Redis hit rate, check PostgreSQL connection pool saturation, verify downstream fraud-scoring API latency, and escalate to Priya Patel or the payment-service on-call rotation.

**15. What are the SLOs for the payment service, and where are they defined?**
> Expected sources: `runbooks/payment-service-runbook.md`, `architecture/current_architecture.md`
> The answer should state the SLOs: 99.95% availability, p99 latency < 350ms, and error rate < 0.05% measured over a 30-day rolling window.

**16. Who are the current owners of the payment service and Redis configuration?**
> Expected sources: `runbooks/payment-service-runbook.md`, `adrs/ADR-002-adopt-redis.md`
> The answer should name Priya Patel as the payment-service lead and the Platform Engineering team (Sarah Chen) as the Redis cluster owner, with service-level TTL configuration owned per service.

---

## Historical Context & Reasoning

**17. What were the key arguments against adopting Redis made during the internal debate?**
> Expected sources: `slack/redis-discussion.md`, `adrs/ADR-002-adopt-redis.md`
> The answer should surface James Okonkwo's concern about operational complexity and Marcus Rivera's concern about memory sizing, and how those concerns were addressed (managed ElastiCache, conservative initial sizing with autoscaling).

**18. Was there any opposition to the EKS migration, and what were the concerns?**
> Expected sources: `adrs/ADR-001-migrate-to-eks.md`, `slack/redis-discussion.md`
> The answer should describe the concern about Kubernetes operational overhead, the risk of a long migration disrupting feature velocity, and how the team mitigated these with a phased approach and a Platform Engineering investment in Helm chart standards.

**19. What was the state of the payment service's latency before and after Redis was introduced?**
> Expected sources: `jira/JIRA-101-payment-latency.md`, `adrs/ADR-002-adopt-redis.md`, `incidents/payment-outage-postmortem.md`
> The answer should cite the pre-Redis p99 of ~620ms, the target of <300ms, and the post-Redis p99 of ~180ms achieved within two weeks of rollout.

**20. What systemic issues did the November 2025 outage reveal about CloudShop's migration practices?**
> Expected sources: `incidents/payment-outage-postmortem.md`, `migrations/ecs-to-eks-migration.md`
> The answer should connect the outage to broader process gaps: no standardized resource limit baselines, no mandatory PodDisruptionBudget policy, insufficient pre-production load testing in the EKS environment, and an incomplete migration runbook that lacked a critical services checklist.
