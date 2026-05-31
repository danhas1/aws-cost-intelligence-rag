# CloudShop Current Architecture

**Document status:** Living document
**Last updated:** 2026-05-20
**Owner:** Platform Engineering (Sarah Chen)
**Review cadence:** Quarterly or after any major architectural change

---

## Overview

CloudShop operates a microservices architecture on AWS, running all workloads on Amazon EKS (Elastic Kubernetes Service) following the completion of the ECS-to-EKS migration in December 2025. All three core backend services are deployed to a single multi-tenant EKS cluster segmented by Kubernetes namespace and NetworkPolicy.

The platform serves approximately 4 million monthly active users, processes an average of 85,000 orders per day, and has a peak throughput of ~3,200 requests/second across all services during major sale events.

---

## Core Services

### payment-service

| Property | Value |
|---|---|
| Language | Java 17, Spring Boot 3.2 |
| Deployment | EKS, `payments` namespace |
| Replicas (steady state) | 6 |
| Replicas (peak) | 20 (HPA, CPU + custom RPS metric) |
| Primary datastore | Amazon RDS PostgreSQL 15 (Multi-AZ, `payments-db`) |
| Cache | Amazon ElastiCache for Redis 7 (shared cluster, `cloudshop-redis`) |
| External dependencies | Stripe API (payment processing), Sift (fraud scoring) |
| Internal dependencies | `user-service` (gRPC, user identity validation) |
| Ingress | AWS ALB → Kubernetes Ingress → ClusterIP Service |
| SLO: Availability | 99.95% (30-day rolling) |
| SLO: Latency | p99 < 350ms |
| SLO: Error rate | < 0.05% (5xx responses) |
| On-call owner | Priya Patel (primary), payments-oncall PagerDuty rotation |

**Key Redis usage:**
- Idempotency key cache (TTL: 24h) — prevents duplicate payment submissions
- Session-scoped payment intent cache (TTL: 30m) — avoids redundant PostgreSQL round-trips during multi-step checkout flows
- Rate-limiting counters for fraud heuristics (TTL: 1m rolling window)

**Key architectural note:** The idempotency key caching pattern was introduced in September 2025 (ADR-002) after JIRA-101 identified repeated PostgreSQL queries for idempotency lookups as the primary driver of p99 latency regression. Prior to Redis adoption, the service performed a synchronous PostgreSQL SELECT on every payment submission to check key uniqueness, which caused contention under high write load.

---

### user-service

| Property | Value |
|---|---|
| Language | Go 1.22 |
| Deployment | EKS, `users` namespace |
| Replicas (steady state) | 4 |
| Replicas (peak) | 12 (HPA, CPU) |
| Primary datastore | Amazon RDS PostgreSQL 15 (Multi-AZ, `users-db`) |
| Cache | Amazon ElastiCache for Redis 7 (shared cluster, `cloudshop-redis`) |
| External dependencies | Auth0 (social login federation) |
| Internal dependencies | None (upstream dependency for other services) |
| Ingress | AWS ALB → Kubernetes Ingress → ClusterIP Service |
| SLO: Availability | 99.9% (30-day rolling) |
| SLO: Latency | p99 < 200ms |
| SLO: Error rate | < 0.1% |
| On-call owner | Backend Platform rotation |

**Key Redis usage:**
- JWT session token blacklist (TTL: token expiry, used for logout/revocation)
- User preference cache (TTL: 5m) — avoids DB reads on every recommendation request
- Login rate-limiting (TTL: 15m sliding window)

---

### recommendation-service

| Property | Value |
|---|---|
| Language | Python 3.12, FastAPI |
| Deployment | EKS, `recommendations` namespace |
| Replicas (steady state) | 3 |
| Replicas (peak) | 10 (HPA, CPU + queue depth) |
| Primary datastore | Amazon DynamoDB (on-demand, `recommendations-table`) |
| Model artifact storage | Amazon S3 (`cloudshop-ml-models` bucket) |
| Cache | Amazon ElastiCache for Redis 7 (shared cluster, `cloudshop-redis`) |
| External dependencies | None |
| Internal dependencies | `user-service` (gRPC, user preference data) |
| Ingress | Internal only (no public ALB), consumed by API Gateway |
| SLO: Availability | 99.5% (30-day rolling) |
| SLO: Latency | p99 < 500ms |
| SLO: Error rate | < 0.5% |
| On-call owner | James Okonkwo (primary), data-platform rotation |

**Key Redis usage:**
- Recommendation result cache keyed by `(user_id, context_hash)` (TTL: 10m)
- Popular-items fallback cache (TTL: 1h) — served when personalized recommendations are unavailable

**Key architectural note:** Model training runs are triggered nightly by an EKS CronJob. As of April 2026 (following the S3 storage cost spike postmortem), model artifacts are versioned with a 3-version retention policy, and an S3 Intelligent-Tiering lifecycle rule transitions artifacts older than 30 days automatically.

---

## Infrastructure Layer

### EKS Cluster

| Property | Value |
|---|---|
| Cluster name | `cloudshop-prod` |
| Kubernetes version | 1.30 |
| Node groups | `general` (m6i.2xlarge), `memory-optimized` (r6i.2xlarge, for recommendation-service) |
| Autoscaling | Cluster Autoscaler + Karpenter (introduced Q1 2026) |
| Networking | AWS VPC CNI |
| Service mesh | None (evaluated Istio; deferred to H2 2026 — see ADR-001) |
| Ingress controller | AWS Load Balancer Controller |
| Secrets management | AWS Secrets Manager, synced via External Secrets Operator |
| Certificate management | cert-manager, Let's Encrypt for internal, ACM for public |

**Namespaces:**

| Namespace | Contents | NetworkPolicy |
|---|---|---|
| `payments` | payment-service | Ingress from `ingress-nginx`, `users`; no egress to `recommendations` |
| `users` | user-service | Ingress from all internal namespaces |
| `recommendations` | recommendation-service | Ingress from API Gateway pod |
| `platform` | External Secrets Operator, cert-manager, Cluster Autoscaler | N/A |
| `monitoring` | Prometheus, Grafana, Alertmanager | N/A |

### Redis (ElastiCache)

| Property | Value |
|---|---|
| Cluster name | `cloudshop-redis` |
| Engine | Redis 7.2 |
| Mode | Cluster mode enabled, 3 shards, 1 replica per shard |
| Instance type | cache.r7g.large |
| Encryption | In-transit (TLS) and at-rest (AWS KMS) |
| Multi-AZ | Yes (replica in secondary AZ) |
| Owner | Platform Engineering |
| TTL governance | Each service team owns their key namespace and TTL policy |

**Key prefixes by service:**

| Prefix | Service | Description |
|---|---|---|
| `pmt:idem:` | payment-service | Idempotency keys |
| `pmt:intent:` | payment-service | Payment intent cache |
| `pmt:rl:` | payment-service | Rate-limiting counters |
| `usr:session:` | user-service | JWT blacklist |
| `usr:pref:` | user-service | User preference cache |
| `usr:rl:` | user-service | Login rate-limiting |
| `rec:result:` | recommendation-service | Recommendation results |
| `rec:popular:` | recommendation-service | Popular items fallback |

### Networking & Security

- All inter-service communication within the cluster uses gRPC over TLS with mTLS certificates managed by cert-manager.
- Public API traffic terminates at AWS ALB, which performs TLS offload before forwarding to the Ingress controller.
- All services run with least-privilege IAM roles via IRSA (IAM Roles for Service Accounts).
- PodDisruptionBudgets are enforced for all services with `maxUnavailable: 1` (introduced as a mandatory policy following the November 2025 payment outage postmortem).

### Observability

| Component | Tool |
|---|---|
| Metrics | Prometheus + Grafana |
| Logs | Fluent Bit → CloudWatch Logs → S3 (90-day retention) |
| Traces | AWS X-Ray (payment-service, user-service), OpenTelemetry collector |
| Alerting | Alertmanager → PagerDuty |
| Uptime | Synthetic checks via Datadog Synthetics (payment checkout flow, user login) |
| SLO tracking | Grafana SLO dashboards, weekly SLO review in #eng-reliability |

---

## Historical Context

| Date | Change |
|---|---|
| Q3 2023 | Monolith decomposed into payment-service, user-service, recommendation-service |
| Q1 2024 | All services containerized and deployed to Amazon ECS |
| Q3 2025 | Redis (ElastiCache) adopted as shared cache layer (ADR-002) |
| Q4 2025 – Q4 2025 | ECS-to-EKS migration (ADR-001); completed December 2025 |
| Nov 2025 | Payment service outage during EKS migration; PodDisruptionBudget policy introduced |
| Q1 2026 | S3 storage cost spike; ML artifact lifecycle policy introduced |
| Q1 2026 | Karpenter adopted for node autoscaling |
| Q2 2026 | Istio evaluation in progress (non-production) |
