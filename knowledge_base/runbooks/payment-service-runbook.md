# payment-service Operational Runbook

**Service:** payment-service
**Version:** 3.1 (updated for EKS, December 2025)
**Owner:** Priya Patel (primary), payments-oncall rotation
**Cluster:** `cloudshop-prod`, namespace: `payments`
**PagerDuty service:** `payment-service-prod`
**Last updated:** 2026-02-10
**Next review:** 2026-08-10

---

## Quick Reference

| Resource | Location |
|---|---|
| Grafana dashboard | [payment-service overview](https://grafana.internal/d/payment-service) |
| PagerDuty escalation policy | Payments On-Call → Priya Patel → Emily Torres |
| Kubernetes namespace | `payments` |
| Deployment name | `payment-service` |
| Helm chart | `charts/payment-service` (internal Helm repo) |
| Redis key prefix | `pmt:` |
| PostgreSQL instance | `payments-db.prod.cloudshop.internal` (RDS, us-east-1) |
| Stripe dashboard | [stripe.com/dashboard](https://stripe.com/dashboard) (requires Stripe MFA) |
| Sift dashboard | [sift.com/console](https://sift.com/console) |
| Runbook changelog | See bottom of this document |

---

## SLOs

| Metric | Target | Measurement Window |
|---|---|---|
| Availability | 99.95% | 30-day rolling |
| p99 latency | < 350ms | 30-day rolling |
| Error rate (5xx) | < 0.05% | 30-day rolling |

SLO burn rate alerts are configured in Alertmanager. At 2x burn rate (fast burn), PagerDuty fires immediately. At 1x burn rate over 6 hours (slow burn), a Slack alert fires in #eng-reliability.

---

## Architecture Overview

```
Internet → AWS ALB → Kubernetes Ingress (payments namespace)
                           │
                   payment-service Pods (6 replicas, HPA: 6–20)
                      /              \
              PostgreSQL          Redis (ElastiCache)
              (payments-db)       (cloudshop-redis, pmt: prefix)
                                       │
                              External APIs:
                              - Stripe API (payment processing)
                              - Sift API (fraud scoring, async)
                              - user-service (gRPC, identity)
```

**Critical dependency graph:**
- **Stripe API:** Synchronous, in the payment submission critical path. Stripe latency directly impacts our p99. Target Stripe p99: <120ms.
- **Redis:** Idempotency key cache, payment intent cache, rate-limiting counters. Loss of Redis degrades performance but does not cause correctness failures (fallback to PostgreSQL is implemented for all Redis reads). See [Redis failure mode](#redis-failure-mode) below.
- **PostgreSQL (payments-db):** Primary datastore. Loss of PostgreSQL causes payment-service to be fully unavailable. Multi-AZ RDS with automatic failover (~30-45s failover time).
- **user-service (gRPC):** Called synchronously during payment submission to validate user identity. Loss of user-service causes payment submissions to fail with 400 (identity unverifiable). Timeouts are set to 2s with a fast failure.
- **Sift (fraud scoring):** Called asynchronously. Fraud score is computed after payment authorization and does not block the critical path. Loss of Sift causes payments to be processed without fraud scoring (they are flagged for manual review in this state).

---

## Common Alert Responses

### Alert: `payment-service-high-latency` (p99 > 350ms for 5 minutes)

**Step 1: Check Redis hit rate**

```bash
kubectl exec -n payments deployment/payment-service -- \
  curl -s localhost:8080/actuator/metrics/cache.gets | jq '.measurements'
```

Or check the "Redis cache hit rate" panel in the Grafana dashboard. Target hit rate: ≥90%.

- **Hit rate < 80%:** Redis is missing keys that should be cached. Possible causes: Redis restart (keys evicted), new pod deployment with cold cache, TTL too short. If Redis was recently restarted, this is transient — allow 10–15 minutes for cache warm-up. p99 will recover as the cache fills.
- **Hit rate nominal but latency high:** Redis is not the cause. Proceed to Step 2.

**Step 2: Check Stripe API latency**

Open the Stripe dashboard → Developers → API logs. Check p99 latency on `POST /v1/payment_intents` over the last 30 minutes. Also check [status.stripe.com](https://status.stripe.com).

- **Stripe latency > 200ms:** Stripe is contributing to our latency. If Stripe is having an incident, open a P2 incident and notify the business. Do not attempt to mitigate on our side — we cannot control Stripe API performance.
- **Stripe latency nominal:** Proceed to Step 3.

**Step 3: Check PostgreSQL connection pool saturation**

```bash
kubectl exec -n payments deployment/payment-service -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq '.measurements'
```

HikariCP pending connections > 0 indicates pool saturation. This means payment-service threads are waiting for a database connection.

- **Pool saturated:** Check `payment_idempotency_keys` query performance. Connect to `payments-db` (read replica is acceptable) and run:
  ```sql
  SELECT count(*) FROM payment_idempotency_keys WHERE created_at > NOW() - INTERVAL '24 hours';
  ```
  If this count is much larger than expected, there may be a Redis failure causing all idempotency checks to fall back to PostgreSQL. Check Redis connectivity (see [Redis failure mode](#redis-failure-mode)).

- **Pool not saturated:** Proceed to Step 4.

**Step 4: Check user-service gRPC latency**

```bash
kubectl logs -n payments deployment/payment-service --since=10m | \
  grep "user-service" | grep "latency_ms" | tail -20
```

If user-service gRPC calls are taking >500ms, user-service may be degraded. Check the user-service Grafana dashboard and escalate to the Backend Platform rotation if user-service is the root cause.

**Step 5: Escalate**

If the above steps do not identify the root cause within 15 minutes, escalate to Priya Patel (primary) or the payments-oncall rotation. Include:
- Current p99 (screenshot or metric value)
- Redis hit rate
- Stripe API latency
- HikariCP pending connection count
- Any relevant log excerpts

---

### Alert: `payment-service-error-rate-high` (5xx error rate > 0.05% for 5 minutes)

**Step 1: Identify error type**

```bash
kubectl logs -n payments deployment/payment-service --since=5m | \
  grep '"level":"ERROR"' | jq '{timestamp: .timestamp, message: .message, exception: .exception}' | head -30
```

Common error categories:

| Error pattern | Likely cause | Action |
|---|---|---|
| `StripeException: Request failed with status code 503` | Stripe outage | Check status.stripe.com, open incident |
| `Connection to payments-db refused` | PostgreSQL failover in progress | Wait 45s for RDS failover; if not recovered, page DBA |
| `RedisConnectionException` | Redis cluster issue | See Redis failure mode section; payments still process via Postgres fallback |
| `UNAVAILABLE: user-service` | user-service is down | Check user-service health; escalate to Backend Platform |
| `Idempotency key conflict` | Duplicate payment submission (correct behavior) | Not an error; verify client is not misconfigured |
| `OutOfMemoryError` | JVM memory limit exceeded | See OOMKill response below |

**Step 2: If errors are retryable (Stripe 503, transient network)**

Payment-service has retry logic for Stripe 503 responses with exponential backoff (max 3 retries, max 8 seconds total). If the Stripe incident resolves quickly, errors will self-heal. Monitor the error rate over the next 5 minutes.

**Step 3: If errors are non-retryable**

Open a SEV-1 incident immediately. Notify #incidents channel. Engage Priya Patel.

---

### Alert: `payment-service-pod-oom` (OOMKilled pods detected)

**Immediate action:**

Check the number of healthy pods:

```bash
kubectl get pods -n payments -l app=payment-service
```

If fewer than 4 healthy pods are running, take immediate action to prevent full outage.

**Step 1: Cordon the affected node to prevent further scheduling**

```bash
# Get the node the OOMKilled pod was on
kubectl describe pod <pod-name> -n payments | grep "Node:"

# Cordon the node
kubectl cordon <node-name>
```

**Step 2: Check current resource usage vs. limits**

```bash
kubectl top pods -n payments
```

If pods are consuming near or at their memory limit (1.5Gi), the JVM may be under abnormal memory pressure. Check for:
- Unusually large payment batch (high volume of concurrent submissions)
- Memory leak (compare current heap usage to yesterday's peak)

**Step 3: Increase replica count temporarily if under-replicated**

```bash
kubectl scale deployment payment-service -n payments --replicas=10
```

This will increase replica count above HPA minimum while the situation is investigated.

**Step 4: Check JVM heap dump if memory leak suspected**

Coordinate with Priya Patel for JVM heap dump analysis. Do not attempt this during an active incident without guidance.

---

### Redis Failure Mode

payment-service is designed to degrade gracefully when Redis is unavailable. The fallback behavior is:

1. **Idempotency key cache miss:** Fall through to PostgreSQL `payment_idempotency_keys` table check. This is slower (~40–80ms additional latency per request) but correct.
2. **Payment intent cache miss:** Re-fetch from PostgreSQL. Same latency impact.
3. **Rate-limiting counter unavailable:** Fail open (allow the request). This is an accepted risk — fraud heuristics will be less effective during a Redis outage but payments will continue processing.

**Expected impact of Redis outage on payment-service:**
- p99 latency: increases from ~180ms to ~250–290ms (still within SLO)
- Error rate: unchanged (no payment submissions fail due to Redis)
- HikariCP pool: will see increased connection demand; monitor pool saturation

To check Redis connectivity from a payment-service pod:

```bash
kubectl exec -n payments deployment/payment-service -- \
  redis-cli -h $REDIS_HOST -p 6379 --tls PING
```

If Redis is confirmed unavailable, open a P2 incident and page the Platform Engineering on-call (Sarah Chen primary).

---

## Deployment Procedures

### Standard deployment (Helm)

```bash
# Update values in charts/payment-service/values.yaml
# Then deploy:
helm upgrade payment-service charts/payment-service \
  --namespace payments \
  --set image.tag=<new-tag> \
  --wait \
  --timeout 10m
```

Deployments use a RollingUpdate strategy with `maxSurge: 2, maxUnavailable: 0`. No payment requests should fail during a rolling deployment.

**Verify deployment health:**

```bash
kubectl rollout status deployment/payment-service -n payments
kubectl get pods -n payments -l app=payment-service
```

### Rollback a deployment

```bash
kubectl rollout undo deployment/payment-service -n payments

# Or roll back to a specific revision:
kubectl rollout history deployment/payment-service -n payments
kubectl rollout undo deployment/payment-service -n payments --to-revision=<N>
```

Rollback completes in ~2 minutes (RollingUpdate in reverse).

### Emergency: scale to zero (last resort)

Only as a last resort if pods are causing cascading failures in other services. Scaling to zero will cause a full payment outage.

```bash
# REQUIRES approval from Priya Patel or Emily Torres
kubectl scale deployment payment-service -n payments --replicas=0
```

---

## Configuration Reference

### Key environment variables

| Variable | Description | Source |
|---|---|---|
| `REDIS_HOST` | ElastiCache primary endpoint | AWS Secrets Manager |
| `REDIS_PORT` | Redis port (6380 for TLS) | Helm values |
| `REDIS_IDEMPOTENCY_TTL_SECONDS` | Idempotency key TTL (default: 86400 = 24h) | Helm values |
| `REDIS_INTENT_TTL_SECONDS` | Payment intent cache TTL (default: 1800 = 30m) | Helm values |
| `STRIPE_SECRET_KEY` | Stripe API key (prod) | AWS Secrets Manager |
| `SIFT_API_KEY` | Sift fraud scoring API key | AWS Secrets Manager |
| `DB_URL` | PostgreSQL JDBC URL | AWS Secrets Manager |
| `DB_POOL_SIZE` | HikariCP pool size (default: 30) | Helm values |
| `USER_SERVICE_GRPC_HOST` | user-service internal hostname | Helm values |
| `USER_SERVICE_GRPC_TIMEOUT_MS` | gRPC call timeout (default: 2000) | Helm values |

### Redis TTL Policy

All Redis writes from payment-service must include a TTL. No key should ever be written without expiration. This is enforced via a code linting rule (see payment-service repo: `checkstyle/redis-ttl-rule.xml`).

| Key prefix | TTL | Rationale |
|---|---|---|
| `pmt:idem:<hash>` | 86400s (24h) | Matches Stripe's idempotency window |
| `pmt:intent:<id>` | 1800s (30m) | Matches payment session timeout |
| `pmt:rl:<merchant_id>` | 60s (1m rolling) | Fraud rate-limiting window |

---

## Escalation Policy

| Condition | First contact | Second contact | Third contact |
|---|---|---|---|
| p99 > 350ms, self-resolving within 15m | Monitor | — | — |
| p99 > 350ms, not self-resolving | Priya Patel | Marcus Rivera | Emily Torres |
| Error rate > 0.05% | Priya Patel | Emily Torres | — |
| Full payment outage | Priya Patel immediately | Emily Torres | CTO (Slack DM) |
| Redis cluster unavailable | Platform Engineering (Sarah Chen) | Marcus Rivera | — |
| Stripe API incident | Priya Patel notifies #incidents | Emily Torres notifies business stakeholders | — |
| Suspected fraud pattern | Priya Patel | Head of Trust & Safety | — |

---

## Runbook Changelog

| Date | Change | Author |
|---|---|---|
| 2026-02-10 | Added Redis TTL policy table; clarified rate-limiting fail-open behavior | Priya Patel |
| 2025-12-15 | Updated all commands for EKS (`kubectl` replacing `ecs-cli`); added OOMKill response section | Priya Patel |
| 2025-12-12 | Initial EKS runbook created post-migration | Priya Patel, Alex Kim |
| 2025-11-22 | Added note on PodDisruptionBudget and autoscaler behavior (post-incident learnings) | Marcus Rivera |
| 2025-09-15 | Added Redis failure mode section; updated latency triage to include Redis hit rate as Step 1 | Priya Patel |
