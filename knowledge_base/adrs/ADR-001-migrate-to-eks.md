# ADR-001: Migrate Container Orchestration from Amazon ECS to Amazon EKS

**Status:** Accepted
**Date:** 2025-10-08
**Deciders:** Sarah Chen (Platform Engineering Lead), Emily Torres (Engineering Manager), Marcus Rivera (Senior SRE), Priya Patel (payment-service), James Okonkwo (recommendation-service)
**Supersedes:** N/A
**Superseded by:** N/A
**Related tickets:** JIRA-98 (Platform scalability), JIRA-101 (payment-service latency)
**Related ADRs:** ADR-002 (Redis adoption, driven by same scaling pressures)

---

## Context

CloudShop has operated on Amazon ECS since Q1 2024, when the monolith was decomposed into three microservices. At the time, ECS was a reasonable choice: it required minimal operational overhead, integrated tightly with the existing AWS toolchain, and the team had no prior Kubernetes experience.

By mid-2025, ECS was showing meaningful operational friction across several dimensions:

### 1. Scheduling inflexibility under heterogeneous workloads

The recommendation-service requires GPU-adjacent, memory-heavy nodes for model serving (r6i instance family), while the payment-service and user-service run efficiently on general-purpose compute (m6i). ECS capacity providers do not support fine-grained node affinity rules without significant custom tooling. We worked around this with separate ECS clusters, which fragmented our operational surface area: two separate sets of capacity providers, two sets of CloudWatch log groups, two sets of ECS service IAM roles, and duplicated Terraform modules.

### 2. Network policy and security isolation

ECS does not have a native equivalent to Kubernetes NetworkPolicy. We were enforcing service-to-service network isolation exclusively via Security Groups, which operate at the ENI level and become unwieldy to reason about as the number of services grows. A planned expansion to 6 services in 2026 would make Security Group sprawl a significant audit and compliance risk, particularly under PCI-DSS requirements for the payment-service network segment.

### 3. RBAC limitations

ECS's IAM-based access model does not offer the granularity of Kubernetes RBAC for controlling who can deploy, exec into, or read logs from specific services. Several near-miss incidents in 2025 involved engineers running ECS exec sessions on payment-service tasks in production, which was not detectable via standard CloudTrail without custom alerting.

### 4. Ecosystem and tooling

The broader container orchestration ecosystem (Helm, Argo CD, KEDA, Karpenter, cert-manager, External Secrets Operator, OpenTelemetry Operator) is built primarily for Kubernetes. Running ECS meant maintaining custom glue code or forgoing these tools entirely. The Platform Engineering team was maintaining ~2,400 lines of custom ECS deployment tooling that would be unnecessary on EKS.

### 5. Growing operational expertise in the market

Hiring in 2025 consistently revealed that candidates with container orchestration experience skewed heavily toward Kubernetes. ECS-specific knowledge is narrower and less transferable. Three of our four most recent DevOps/SRE hires had production EKS experience; none had ECS experience beyond entry-level AWS certifications.

---

## Decision

**We will migrate all CloudShop workloads from Amazon ECS to Amazon EKS.**

The migration will proceed service by service in order of risk (lowest-risk first): recommendation-service → user-service → payment-service.

A dual-running period will be maintained for each service during its migration window, with traffic split via weighted ALB target groups. This allows fast rollback by shifting weights without a redeployment.

EKS will be managed via Terraform and configured to use:
- AWS VPC CNI for pod networking (compatible with existing VPC layout)
- Kubernetes NetworkPolicy for service-to-service isolation
- IRSA for per-pod IAM roles
- Helm charts (maintained by Platform Engineering) for all service deployments
- External Secrets Operator for secrets management (replacing ECS Secrets Manager integration)
- Cluster Autoscaler initially, with Karpenter evaluated post-migration

We explicitly defer adoption of a service mesh (Istio, Linkerd) to a post-migration phase. The operational complexity of a service mesh is not justified by current traffic patterns and would increase migration risk.

---

## Alternatives Considered

### Option A: Remain on ECS and invest in tooling

**Rejected.** The fundamental constraints — networking model, scheduling rigidity, RBAC granularity — are architectural, not tooling gaps. Investing engineering time in ECS tooling would produce a bespoke platform that becomes harder to staff and maintain over time, while the ecosystem continues to move toward Kubernetes.

### Option B: Migrate to HashiCorp Nomad

**Rejected.** Nomad is a strong orchestrator with a simpler operational model than Kubernetes, but CloudShop's primary integration surface is AWS. Nomad's AWS integrations (IAM, ALB, CloudWatch) require third-party drivers or significant custom work, whereas EKS provides native AWS integrations out of the box. Additionally, the hiring pool for Nomad expertise is materially smaller than for Kubernetes.

### Option C: ECS with AWS App Mesh (service mesh)

**Rejected.** This was proposed as a way to address the networking and observability gaps without migrating orchestrators. App Mesh is based on Envoy and provides mTLS, traffic shifting, and observability. However, AWS has signaled reduced investment in App Mesh in favor of VPC Lattice, which itself does not have the maturity to serve as our primary service mesh. Choosing App Mesh would require a second migration within 18–24 months, which the team was not willing to accept.

### Option D: EKS on AWS Fargate (no managed nodes)

**Considered but rejected as the primary deployment target.** Fargate eliminates node management overhead but imposes meaningful constraints: no DaemonSets (which we need for Fluent Bit log collection and the Node Exporter), slower pod startup times, and higher per-vCPU costs at steady-state. We will use Fargate selectively for batch workloads (recommendation-service training jobs) where fast pod startup is not required.

---

## Consequences

### Positive
- Unified compute platform simplifies Platform Engineering operational burden.
- Kubernetes NetworkPolicy enables granular service-to-service network isolation (PCI-DSS alignment).
- Access to the full CNCF tooling ecosystem without custom glue code.
- Kubernetes RBAC allows auditable, fine-grained access control per namespace.
- Improved hiring and onboarding: industry-standard toolchain.

### Negative / Risks
- **Migration risk:** Moving the payment-service carries PCI-DSS compliance implications. Any misconfiguration could create a compliance gap or availability incident. Mitigation: payment-service migrates last, with a dedicated security review and a 72-hour parallel-run period.
- **Kubernetes operational complexity:** EKS requires understanding of node lifecycle management, etcd, kube-apiserver availability, and resource limits. Mitigation: Platform Engineering owns the cluster; service teams interact only via Helm chart values.
- **Short-term velocity impact:** The migration will consume approximately 60% of Platform Engineering capacity for 8 weeks, reducing support bandwidth for product teams.
- **Cost:** EKS cluster management fee ($0.10/hr/cluster) and slightly higher EC2 overhead during the dual-running migration window. Estimated incremental cost: ~$1,200/month during migration, falling to ~$200/month post-migration (cluster management fee only) vs. ECS.

---

## Implementation

See [migrations/ecs-to-eks-migration.md](../migrations/ecs-to-eks-migration.md) for the full migration plan and execution log.

---

## Review Notes

*2025-10-08 — Emily Torres:* Approved. PCI-DSS compliance review has been scheduled with the security team before payment-service migration begins.

*2025-10-08 — Marcus Rivera:* Approved with the condition that we enforce PodDisruptionBudgets for all critical services before declaring the migration complete. ECS handled draining gracefully; we need to be explicit about this in Kubernetes.

*2025-11-20 — Sarah Chen:* Adding a note post-incident: the November 2025 payment-service outage (see postmortem) was partly caused by the PodDisruptionBudget gap that Marcus flagged. PDBs are now a mandatory checklist item for all service migrations.
