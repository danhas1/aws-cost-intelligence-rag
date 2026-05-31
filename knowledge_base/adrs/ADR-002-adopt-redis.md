# ADR-002: Adopt Redis (ElastiCache) as Shared Caching Layer

**Status:** Accepted
**Date:** 2025-09-03
**Deciders:** Sarah Chen (Platform Engineering Lead), Priya Patel (payment-service), James Okonkwo (recommendation-service), Marcus Rivera (Senior SRE)
**Supersedes:** N/A
**Superseded by:** N/A
**Related tickets:** JIRA-101 (payment-service latency regression)
**Related ADRs:** ADR-001 (EKS migration; Redis on ElastiCache fits the AWS-native strategy)
**Related Slack:** #eng-platform, thread starting 2025-08-14 (archived in slack/redis-discussion.md)

---

## Context

### The immediate problem: payment-service latency regression

In July 2025, the payment-service p99 latency rose from a historical baseline of ~180ms to ~620ms over a three-week period following a routine release. JIRA-101 tracked the investigation. After ruling out Stripe API latency, PostgreSQL query plan regressions, and GC pressure in the JVM, the root cause was identified as a query pattern introduced in payment-service v2.3.1.

Every payment submission performs an idempotency key check: a SELECT against the `payment_idempotency_keys` table to verify that the submitted key has not been processed before. Prior to v2.3.1, idempotency keys had a short TTL and the table was small. v2.3.1 changed the TTL to 24 hours to align with Stripe's idempotency window, which grew the table to ~18 million rows. A missing compound index was part of the cause, but even after the index was added, the synchronous PostgreSQL round-trip under high write concurrency (150+ payment submissions/second at peak) created measurable latency.

The fix options evaluated for JIRA-101 were:
1. Add the missing index and tune the PostgreSQL query plan. *(Partial fix; doesn't eliminate the synchronous DB call under load.)*
2. Cache idempotency keys in application memory (in-JVM cache). *(Rejected: doesn't survive pod restarts; payment-service runs 6 replicas with independent memory.)*
3. Introduce a shared external cache. *(Selected.)*

### The broader context: multiple services independently discovering the same need

Concurrently, the recommendation-service team was exploring caching recommendation results to reduce DynamoDB read costs. Personalized recommendation generation takes 150–400ms per request; caching results for 10 minutes for returning users would reduce DynamoDB reads by an estimated 70% and cut latency for repeat pageloads to <50ms.

The user-service had an existing in-memory JWT blacklist (used for immediate logout/token revocation) that was inconsistent across replicas — a logged-in-on-pod-A token could still be valid on pod-B for up to 5 minutes after logout. This was a known security gap.

All three teams were solving the same category of problem (shared mutable state across service replicas) with different improvised solutions. A unified caching infrastructure would eliminate the duplicated operational surface and allow the Platform Engineering team to standardize on a single Redis cluster with consistent monitoring, alerting, and backup policies.

---

## Decision

**We will adopt Amazon ElastiCache for Redis 7 as the shared caching layer for all three core services.**

Key implementation decisions:

1. **Managed service (ElastiCache), not self-hosted Redis.** Operating Redis on ECS/EKS ourselves would require the Platform team to own patching, backup, failover, and capacity planning for a stateful system. ElastiCache handles these operationally, with automatic failover and Multi-AZ replication.

2. **Cluster mode enabled, 3 shards, 1 replica per shard.** This provides horizontal write scalability and reads from replicas for non-critical paths. Starting with `cache.r7g.large` instances (6.38 GB per node), which is conservative but allows memory headroom before autoscaling or resharding is required.

3. **Single shared cluster, namespaced by key prefix.** Each service team owns their key namespace (`pmt:`, `usr:`, `rec:`) and is responsible for defining TTL policies within that namespace. Platform Engineering owns cluster-level concerns (memory policy, eviction policy, TLS configuration, IAM). The cluster uses `allkeys-lru` eviction to handle memory pressure gracefully without application-level eviction logic.

4. **Encryption in-transit (TLS) and at-rest (AWS KMS).** Required by PCI-DSS for the payment-service key namespace. Applied uniformly across the cluster for operational simplicity.

5. **Service teams own TTL policy.** TTLs are not standardized globally. Each service team documents the TTL rationale in their service runbook. Payment-service idempotency keys: 24h (matching Stripe's idempotency window). Recommendation results: 10m. User preference cache: 5m. JWT blacklist: token expiry time.

---

## Alternatives Considered

### Option A: Amazon DynamoDB as shared cache (DAX)

**Rejected.** DAX is DynamoDB Accelerator — it only accelerates DynamoDB reads. The payment-service and user-service use PostgreSQL as their primary datastore, so DAX provides no value there. We would need two separate accelerator technologies (DAX for recommendation-service, something else for payment-service and user-service), which defeats the goal of a unified platform.

### Option B: Memcached (ElastiCache for Memcached)

**Rejected.** Memcached lacks native support for data structures beyond simple key-value pairs. The recommendation-service wants to use Redis Sorted Sets for ranking popular items by recency-weighted score. The payment-service fraud rate-limiting pattern requires atomic increment operations with EXPIRE, which is available in Redis but requires more complex workarounds in Memcached. Memcached also does not support replication — every node is independent, which means cache loss on a node failure.

### Option C: Per-service caches (each service manages its own)

**Rejected.** The recommendation-service team had begun evaluating a self-hosted Redis-on-ECS approach. The payment-service team was considering a lightweight Hazelcast embedded cluster. Allowing each service team to choose and operate its own caching technology would produce three different caching substrates with different operational patterns, monitoring integrations, and failure modes. Platform Engineering would need to support all three. A unified platform is significantly lower total cost of ownership.

### Option D: Application-level in-memory caching (Caffeine for Java, groupcache for Go)

**Rejected as the sole solution.** In-memory caching per-replica cannot serve the shared-state use cases that are driving this decision: idempotency key deduplication across payment-service replicas, JWT blacklisting across user-service replicas. An in-memory cache is still appropriate as a first-level cache in front of Redis for certain high-frequency reads (e.g., popular items in recommendation-service), but it cannot replace a shared external cache for these use cases.

---

## Consequences

### Positive
- Resolves JIRA-101 payment-service latency regression: post-Redis deployment, payment-service p99 fell from ~620ms to ~180ms within two weeks.
- Eliminates the user-service JWT blacklist inconsistency across replicas.
- Reduces DynamoDB read costs for recommendation-service by ~68% (observed over the first 30 days post-deployment).
- Single operational surface for caching infrastructure: one ElastiCache cluster, one Terraform module, one Grafana dashboard.
- PCI-DSS compliance: encryption requirements satisfied uniformly.

### Negative / Risks
- **New operational dependency:** A Redis cluster failure could degrade all three services simultaneously if application-level fallback paths are not implemented. Each service team is required to implement graceful degradation: fall through to the primary datastore on Redis cache miss or connection error, rather than propagating a Redis error to the caller.
- **Memory sizing risk:** Initial sizing of `cache.r7g.large` (6.38 GB usable per shard) is conservative. If the recommendation-service caches large result payloads or the payment-service key table grows faster than expected, we may need to reshard or upgrade instance types mid-quarter. Marcus Rivera flagged this risk during the Slack discussion; we will monitor Redis memory utilization and set an alert at 70% used.
- **Cross-team TTL governance:** With all services sharing a cluster, a misconfigured TTL (e.g., payment-service writing keys with no TTL) could fill the cache and trigger aggressive LRU eviction of other services' keys. Mitigation: `allkeys-lru` eviction prevents OOM, and Platform Engineering will monitor per-prefix memory utilization via Redis keyspace analysis.
- **Security boundary:** The Redis cluster is accessible to all three services within the VPC. Although TLS is enforced, there is no per-service authentication within the cluster. A compromised payment-service pod could theoretically read `rec:` or `usr:` prefixed keys. This is an accepted risk at current scale; Redis ACLs with per-service usernames will be evaluated in H2 2026.

---

## Implementation Timeline

| Date | Milestone |
|---|---|
| 2025-09-03 | ADR accepted |
| 2025-09-05 | ElastiCache cluster provisioned (Terraform) |
| 2025-09-08 | payment-service idempotency key caching deployed to staging |
| 2025-09-12 | payment-service Redis caching deployed to production |
| 2025-09-15 | p99 latency confirmed at ~180ms (JIRA-101 closed) |
| 2025-09-20 | user-service JWT blacklist migrated to Redis |
| 2025-10-01 | recommendation-service result caching deployed |
| 2025-10-15 | All services using Redis; platform monitoring finalized |

---

## Review Notes

*2025-09-03 — James Okonkwo:* Approved, but I want to make sure we document the fallback behavior requirement clearly. If Redis goes down, recommendation-service should serve from DynamoDB, not return an error. I'll add this to the service runbook.

*2025-09-03 — Marcus Rivera:* Approved. I'm still nervous about memory sizing — we should set a 70% threshold alert and revisit sizing after 60 days of production data. Also, we need to make sure no service writes to Redis without a TTL. I'll add a linting rule to the payment-service codebase.

*2025-09-03 — Priya Patel:* Approved. The idempotency key caching pattern is exactly what we need. I'll take responsibility for deploying payment-service first since JIRA-101 is blocking our Q3 reliability OKR.
