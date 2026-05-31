# Archived Slack Thread: #eng-platform — Redis Caching Discussion

**Channel:** #eng-platform
**Thread start date:** 2025-08-14
**Archived by:** Sarah Chen (2025-09-04, for ADR-002 decision record)
**Context:** This thread preceded the formal ADR-002 decision. It captures the unfiltered team debate, including concerns that were later addressed in the ADR. This is the kind of reasoning that normally disappears when a thread scrolls off.

---

## Thread

---

**Priya Patel** — 2025-08-14 10:07 AM

Hey @platform-eng — I'm still investigating JIRA-101 (payment-service latency). Quick update: I found the problem. It's the idempotency key table. We changed the TTL to 24h in v2.3.1 and the table blew up to 18M rows. We added the missing compound index last month and it helped, but p99 is still sitting at ~290ms. The root issue is that every single payment submission is doing a synchronous round-trip to Postgres to check the idempotency key, and at 150 RPS we're just burning through connections.

I'm thinking we need a cache. Can we talk about options?

---

**Sarah Chen** — 2025-08-14 10:23 AM

Yeah, let's talk about this properly. I've actually been meaning to bring this up — James mentioned last week that the recommendation-service team wants to cache recommendation results, and I know user-service has that gross in-memory JWT blacklist that's been inconsistent across replicas since forever. This might be a "do this once properly" moment.

---

**Marcus Rivera** — 2025-08-14 10:31 AM

Agreed, let's not solve this three separate times. What are we evaluating? Redis? Memcached? Something else?

---

**James Okonkwo** — 2025-08-14 10:45 AM

For recommendation-service I specifically need Redis. Memcached won't work for me — I want to use sorted sets for ranking popular items by a recency-weighted score so I can serve a ranked fallback when personalized recommendations aren't available. That's a native Redis data structure. Memcached is just string key/value.

Also, have you all thought about ElastiCache vs. running Redis ourselves? I'm leaning strongly toward managed. Last thing I want is to be paged at 3am because Redis ran out of memory.

---

**Priya Patel** — 2025-08-14 10:52 AM

100% managed. For payment-service this is going to touch PCI-DSS scope — we need encryption in transit and at rest, and we need to be able to demonstrate that to auditors without maintaining our own certificate rotation logic.

For my use case, Redis is exactly right. Idempotency key lookup, TTL of 24h to match Stripe's window. It's literally a `SET NX EX` operation. Very simple.

---

**Marcus Rivera** — 2025-08-14 11:04 AM

I'm going to push back a little. Not on Redis — Redis is fine — but on the "one shared cluster" approach. If we put payment-service, user-service, and recommendation-service all on the same Redis cluster, we're creating a shared blast radius. What happens if recommendation-service starts writing huge payloads and OOMs the cluster? Do payment-service idempotency checks start failing?

We should either size this very conservatively with aggressive alerting, or talk about separate clusters per service.

---

**Sarah Chen** — 2025-08-14 11:19 AM

That's a fair concern. I've been thinking about this too. Here's my take:

Option A (shared cluster): One ElastiCache cluster, cluster mode enabled, 3 shards. Each service gets a key prefix namespace. We configure `allkeys-lru` eviction so that if memory fills up, Redis starts evicting keys rather than returning errors. The worst-case for payment-service is a cache miss — it falls back to Postgres, which is slower but correct. We alert at 70% memory utilization.

Option B (separate clusters): One cluster per service. Full isolation. But we're paying for 3x the overhead: 3 sets of primary + replica nodes, 3 sets of monitoring, 3 Terraform modules to maintain. At current scale I think this is premature.

My recommendation is Option A with Marcus's safeguards: aggressive memory alerting, mandatory fallback behavior in every service, and we revisit at 60 days.

---

**Marcus Rivera** — 2025-08-14 11:28 AM

OK, I can live with that. The `allkeys-lru` eviction policy is key — it means a memory pressure event degrades performance rather than causing failures. The fallback requirement is non-negotiable for me though. Every service has to be written to handle a Redis cache miss gracefully. This cannot be "Redis goes down and payments stop working."

---

**Priya Patel** — 2025-08-14 11:33 AM

Agreed. I'm actually already writing it that way. The idempotency check code path is:

1. Try Redis `GET pmt:idem:<hash>`
2. If found: return duplicate payment error (fast path)
3. If not found (miss OR Redis error): fall through to Postgres idempotency check
4. After Postgres confirms new key: write-through to Redis with 24h TTL

So Redis is an optimization, not a requirement for correctness. Postgres is still the source of truth.

---

**James Okonkwo** — 2025-08-14 11:41 AM

Same for recommendation-service. Redis stores results as an optimization. On miss or error, we generate recommendations synchronously from DynamoDB. It's slower but it works.

I do want to flag one more thing: recommendation-service recommendation payloads can be large. A user with extensive browsing history might have a recommendation result that's 80–120 KB as JSON. If we're caching `(user_id, context_hash)` → `List[product_id, score]` for 4M users with 10-minute TTL, at any given time we might have ~500k active entries × ~100KB = ~50 GB of recommendation cache. That doesn't fit on a single `r7g.large` shard.

We either need to: (a) store only product IDs in Redis and re-fetch scores from DynamoDB on serve (smaller payloads), or (b) plan for significantly larger cluster sizing for recommendation use cases.

---

**Sarah Chen** — 2025-08-14 12:03 PM

Good catch. Let's go with (a) — store only the ranked product ID list, not the full product data. The recommendation serve path can re-fetch product details from DynamoDB by ID anyway (that lookup is fast and DynamoDB DAX would apply there). Redis payload should be a list of product IDs + a score float, which is ~200 bytes per recommendation set, not 100KB.

At 500k active entries × 200 bytes = 100 MB for recommendations. Very manageable.

---

**James Okonkwo** — 2025-08-14 12:09 PM

That works. I'll refactor the cache write path to store `List[{product_id: str, score: float}]` instead of the full recommendation object.

---

**Alex Kim** — 2025-08-14 12:15 PM

Coming in late here. One operational question: who owns the cluster after it's stood up? Like, who owns memory capacity planning, patching, backup testing?

---

**Sarah Chen** — 2025-08-14 12:22 PM

Platform Engineering owns the cluster. Service teams own their key namespace and TTL policies — if payment-service wants to change their TTL from 24h to 48h, that's Priya's call and she documents it in the payment-service runbook. But cluster-level stuff (instance sizing, patching schedule, replication config, backup policy) is on us.

---

**Alex Kim** — 2025-08-14 12:26 PM

Perfect. I'll provision the cluster in Terraform and set up the CloudWatch alarms. What's the sizing we're going with?

---

**Sarah Chen** — 2025-08-14 12:30 PM

Let's start with `cache.r7g.large` per shard (6.38 GB per shard), 3 shards, 1 replica each. That's ~19 GB total usable memory across the cluster. Conservative, but we can scale up shard count or instance type without downtime (ElastiCache online resharding). We alert at 70% (~13 GB used). Marcus, does that seem right to you?

---

**Marcus Rivera** — 2025-08-14 12:35 PM

Yeah, 70% alert is right. I'd also add a per-prefix memory alert if ElastiCache supports it — actually, it doesn't natively. We'll need a custom Redis keyspace scan script. I'll write something that runs every 5 minutes and reports per-prefix key counts and estimated memory to CloudWatch. That way we can see if one service's namespace is crowding out others before it becomes a problem.

---

**Priya Patel** — 2025-08-14 12:40 PM

Can we just make a decision and write the ADR? This is clearly the right call. We've addressed the blast radius concern (LRU eviction + fallback), the sizing concern (conservative start with alerting), the ownership concern (Platform owns cluster, service teams own namespaces), and the data structure concern (James's sorted sets work natively).

---

**Sarah Chen** — 2025-08-14 12:45 PM

Agreed. I'll write ADR-002 today. Expect a PR by end of day for review. Going with: Redis on ElastiCache, single shared cluster, cluster mode, 3 shards, `r7g.large`, `allkeys-lru`, mandatory fallback behavior per service, 70% memory alert, Platform Engineering owns cluster operations.

---

**Marcus Rivera** — 2025-08-14 12:47 PM

:+1: One last ask: can we make the "no Redis write without a TTL" rule a hard requirement? A service accidentally writing a key with no expiry could fill up a shard silently. I'll add a linting rule to the payment-service codebase to catch `RedisTemplate.opsForValue().set()` calls without a TTL argument.

---

**Priya Patel** — 2025-08-14 12:50 PM

:+1: I'll put it in the runbook and add it to our service-level code review checklist.

---

**James Okonkwo** — 2025-08-14 12:52 PM

Same for recommendation-service. All Redis writes will have explicit TTLs. Will document in the service runbook.

---

**Sarah Chen** — 2025-08-14 12:53 PM

Great. ADR-002 PR incoming. Let's move to #adrs for the formal review.

---

*Thread archived for organizational memory. 14 participants viewed. 0 messages deleted.*
