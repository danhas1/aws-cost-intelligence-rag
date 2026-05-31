# JIRA-101: payment-service p99 latency regression — investigation and resolution

**Ticket ID:** JIRA-101
**Project:** CloudShop Backend
**Type:** Bug / Reliability
**Priority:** P1 — Critical
**Status:** Closed (Resolved)
**Reporter:** Marcus Rivera
**Assignee:** Priya Patel
**Opened:** 2025-07-14
**Closed:** 2025-09-15
**Labels:** payment-service, reliability, slo-breach, performance, redis

---

## Summary

The payment-service p99 response latency increased from a historical baseline of ~180ms to ~620ms over the three weeks following the v2.3.1 release on 2025-07-07. The degradation was gradual rather than sudden, which initially obscured its relationship to the release. The breach of the p99 < 350ms SLO (see architecture/current_architecture.md) triggered an automated PagerDuty alert on 2025-07-14.

---

## Timeline of Discovery

| Date | Event |
|---|---|
| 2025-07-07 | payment-service v2.3.1 released to production |
| 2025-07-08 | No immediate latency signal; p99 ~185ms (within SLO) |
| 2025-07-10 | p99 climbs to ~280ms; within SLO but trending up; no alert triggered |
| 2025-07-12 | p99 reaches ~390ms; SLO breached; alert fires but auto-resolves before on-call reviews it |
| 2025-07-14 | p99 peaks at ~620ms during morning peak traffic; PagerDuty alert fires and is acknowledged by Marcus Rivera |
| 2025-07-14 | JIRA-101 opened; Priya Patel assigned as primary investigator |

---

## Investigation

### Initial hypotheses

**Hypothesis 1: Stripe API latency increase**
Checked Stripe status page and our internal Stripe latency dashboard. Stripe p99 was stable at ~95ms throughout the period. Ruled out.

**Hypothesis 2: PostgreSQL query plan regression**
Ran `EXPLAIN ANALYZE` against the payment-service's most frequent queries. Found that the query against `payment_idempotency_keys` was performing a sequential scan on a table with ~18 million rows. This was a strong signal.

**Hypothesis 3: JVM garbage collection**
Checked payment-service JVM metrics (heap usage, GC pause duration) in Grafana. GC pauses were within normal bounds (p99 GC pause < 12ms). Ruled out as a primary contributor.

**Hypothesis 4: Connection pool saturation**
Checked HikariCP connection pool metrics. Pool was fully utilized during peak periods, with threads waiting for connections. This was a downstream symptom of the slow query, not a root cause.

### Root cause

The root cause was a combination of two factors introduced in v2.3.1:

**Factor 1: TTL change in `payment_idempotency_keys`**

payment-service uses an idempotency key table to prevent duplicate payment submissions (a standard pattern for payment APIs). Prior to v2.3.1, idempotency keys expired after 1 hour (matching our internal payment timeout window). v2.3.1 extended the TTL to 24 hours to align with Stripe's documented idempotency window, which guarantees that Stripe will honor an idempotency key for 24 hours.

This change caused the `payment_idempotency_keys` table to grow from ~750,000 rows (1 hour of peak traffic) to ~18 million rows (24 hours of peak traffic) over three weeks.

**Factor 2: Missing compound index**

The query pattern introduced in v2.3.1 searched idempotency keys by `(merchant_id, idempotency_key_hash)`. A single-column index on `idempotency_key_hash` existed from prior versions, but the new query used a compound predicate that required a compound index. PostgreSQL was falling back to a sequential scan.

**Why the degradation was gradual:** The table grew linearly over 3 weeks. At 750k rows the sequential scan completed in ~5ms; at 18M rows it took ~300ms. The latency increased proportionally as the table grew.

### Fix applied (partial, 2025-07-18)

A compound index `(merchant_id, idempotency_key_hash)` was added to the `payment_idempotency_keys` table via a migration. This brought p99 down to ~290ms — within SLO but still elevated compared to the ~180ms baseline. 

The residual latency was attributed to the synchronous PostgreSQL round-trip itself under high write concurrency: even with an optimal index, 150+ concurrent payment submissions each waiting for a PostgreSQL round-trip created measurable contention on the connection pool.

### Root fix (2025-09-12)

The root fix required introducing a shared caching layer. See ADR-002 for the full decision context. In summary: idempotency keys are now written to and checked in Redis (`pmt:idem:<hash>` prefix, 24h TTL) before any PostgreSQL interaction. The database is used only for durable persistence (write-through) and as the source of truth for audit purposes.

After Redis-backed idempotency caching was deployed to production on 2025-09-12:
- Payment-service p99 fell to ~180ms within 48 hours.
- PostgreSQL `payment_idempotency_keys` query volume dropped by ~94% (Redis cache hit rate: 94%).
- HikariCP pool wait time eliminated.

---

## Impact

| Metric | Value |
|---|---|
| Duration of SLO breach | 2025-07-12 to 2025-09-12 (62 days) |
| Peak p99 latency observed | 620ms (vs. 350ms SLO target) |
| Estimated checkout abandonment uplift (A/B proxy) | +3.2% during peak hours |
| Estimated revenue impact | ~$140,000 over the 62-day breach period (attribution model, not exact) |
| Customer complaints (payment failures / slowness) | 47 support tickets tagged `payment-slow` |

---

## Contributing Factors

1. **Missing index on a new query pattern:** Code review did not catch the missing compound index. The query performed acceptably in staging (table size: ~5,000 rows) and was not load tested with a production-scale dataset.

2. **Gradual degradation pattern evaded alerting:** Our latency alert fired only when p99 exceeded the SLO threshold continuously for 5 minutes. The gradual rise meant that individual alert evaluations resolved before the 5-minute window elapsed. A trend-based alert (p99 increasing >15% week-over-week) would have caught this earlier.

3. **No pre-release performance regression test:** payment-service does not have a performance regression test suite. The v2.3.1 release was validated against functional tests only.

---

## Action Items

| # | Action | Owner | Status | Due |
|---|---|---|---|---|
| 1 | Add compound index `(merchant_id, idempotency_key_hash)` to production | Priya Patel | Done | 2025-07-18 |
| 2 | Deploy Redis idempotency key caching to production | Priya Patel | Done | 2025-09-12 |
| 3 | Add p99 trend-based alert (>15% WoW increase over 3 days) | Marcus Rivera | Done | 2025-08-01 |
| 4 | Add performance regression test (k6 load test at 200 RPS) to payment-service CI pipeline | Priya Patel | Done | 2025-10-01 |
| 5 | Update code review checklist: require EXPLAIN ANALYZE output for any query touching tables expected to exceed 1M rows | Sarah Chen | Done | 2025-08-15 |

---

## Related Documents

- [ADR-002: Adopt Redis](../adrs/ADR-002-adopt-redis.md) — formal decision to introduce Redis as shared cache
- [Slack thread: Redis discussion](../slack/redis-discussion.md) — team debate preceding ADR-002
- [payment-service runbook](../runbooks/payment-service-runbook.md) — includes Redis hit rate monitoring guidance
