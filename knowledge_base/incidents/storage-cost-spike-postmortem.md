# Postmortem: S3 Storage Cost Spike — Q1 2026

**Incident ID:** INC-2026-003
**Severity:** SEV-3 (cost/business impact, no availability impact)
**Status:** Resolved — Action items complete
**Date of discovery:** 2026-01-09
**Duration of undetected growth:** ~90 days (October 2025 – January 2026)
**Services affected:** recommendation-service (ML training pipeline)
**Incident commander:** Sarah Chen
**Lead engineer:** James Okonkwo
**Postmortem author:** Alex Kim
**Postmortem review date:** 2026-03-15
**Postmortem status:** Accepted

---

## Incident Summary

Between October 2025 and January 2026, CloudShop's monthly AWS S3 bill increased from $2,100 to $8,650, an increase of $6,550/month (+312%). The growth went undetected for approximately 90 days because no bucket-level cost alerts existed and the month-over-month growth rate (~45%) stayed below the account-level budget alert threshold (120% of prior month's total).

The cause was the recommendation-service nightly model training pipeline writing full model snapshots (~6 GB per nightly run) to the `cloudshop-ml-models` S3 bucket with no lifecycle policy, no object expiration, and S3 versioning enabled but version expiration disabled. Over 90 days, the bucket accumulated 2.4 TB of unreferenced training artifacts.

No customer-facing service was degraded. The impact was purely financial: an estimated $12,000 in overspend between October and December 2025.

---

## Discovery

**2026-01-09, 10:30 AM:** Alex Kim is conducting a routine monthly AWS Cost Explorer review. The December 2025 AWS bill was $42,100, up from $31,200 in September 2025 — a 35% increase the team had partially attributed to EKS migration overhead (additional EC2 instances, NAT gateway traffic, EKS management fee).

On breaking down the bill by service and resource, Alex notices that S3 alone accounts for $8,650 in December, up from $2,100 three months earlier. Drilling into cost by bucket, `cloudshop-ml-models` shows $6,370 — 74% of total S3 spend and nearly zero in September.

Alex opens JIRA-102 and pings James Okonkwo in #eng-data.

**2026-01-09, 11:05 AM:** James reviews the bucket contents via AWS CLI. Finds 312 objects totaling 2.4 TB. Oldest objects date to October 2025 (when the nightly training pipeline was introduced). All objects are in S3 Standard storage class. S3 versioning is enabled; there are an additional 87 non-current object versions from manual re-run sessions.

---

## Root Cause Analysis

### Primary cause: No lifecycle policy on `cloudshop-ml-models` bucket

The `cloudshop-ml-models` bucket was created in September 2025 by Alex Kim to support the recommendation-service ML training pipeline. The bucket was created with S3 versioning enabled (to prevent accidental deletion) but without a lifecycle policy to expire old versions or transition objects to cheaper storage tiers.

The nightly training CronJob (introduced in early October 2025) writes three artifact types per run:
- `models/collaborative_filter_v{date}.pkl` — ~4.2 GB
- `data/interactions_{date}.npz` — ~1.8 GB
- `config/feature_config_{date}.json` — ~12 MB

At ~6 GB/night over 90 days: ~540 GB from scheduled runs alone. Additional manual re-runs during a debugging session in mid-November (when James was tuning the collaborative filtering hyperparameters) created an additional ~120 GB of non-current object versions.

**Why no lifecycle policy:** When James and Alex set up the bucket, the conversation was: "We might want to roll back to an older model if a new one performs badly. Let's keep versioning on and not expire anything until we have a proper model evaluation framework in place." This was a reasonable short-term position that was never revisited.

### Contributing cause: No cost owner for the bucket

The `cloudshop-ml-models` bucket was created by the Platform Engineering team (Alex Kim) at the request of the recommendation-service team (James Okonkwo). Neither team had explicit cost ownership. The Terraform resource had no `CostOwner` tag and was not included in any team's cost review process.

CloudShop's cost review process at the time reviewed AWS spend by service (EC2, RDS, ElastiCache, etc.) at the account level. There was no per-bucket S3 breakdown in the standard monthly review.

### Contributing cause: Gradual growth evaded alerting

The account-level AWS Budget alert was configured to trigger when monthly spend exceeded 120% of the prior month. Month-over-month S3 growth was ~45% per month:
- October: $2,800 (up from $2,100 — +33%, below threshold)
- November: $4,100 (up from $2,800 — +46%, below threshold)
- December: $8,650 (up from $4,100 — +111%, below threshold)

No single monthly jump crossed the 120% threshold, so no alert fired across the entire 90-day period.

### Contributing cause: S3 versioning enabled without version expiration

S3 versioning was enabled as a safeguard. The team understood that versioning would create multiple object versions, but underestimated the volume of non-current versions generated by manual re-runs. Non-current versions are not visible in the default S3 console view (only the current version is shown), making the accumulation invisible during routine bucket inspection.

---

## Impact

| Metric | Value |
|---|---|
| Duration of undetected cost growth | ~90 days |
| Peak monthly S3 overspend | ~$6,370 (December 2025 vs. $2,100 baseline) |
| Total estimated overspend (Oct–Dec 2025) | ~$12,000 |
| Storage accumulated | 2.4 TB |
| Models that were actually needed | Last 3 (as confirmed with recommendation-service team) |
| Customer-facing impact | None |
| Engineering time to remediate | ~12 hours (investigation + cleanup + policy work) |

---

## What Went Well

1. **Discovery was made during a routine review, not by an external party or a customer complaint.** The monthly Cost Explorer review, even though imperfect, caught the issue.

2. **Cleanup was straightforward.** Once the root cause was clear, cleanup was a single AWS CLI command sequence. No complex rollback or data recovery was needed.

3. **The recommendation-service team was transparent and collaborative.** James Okonkwo immediately took ownership, audited the bucket contents, identified which models were needed, and proposed the remediation plan without needing to be pushed.

4. **No compliance implications.** The artifacts stored in `cloudshop-ml-models` are ML training data and model files, not customer PII or payment data. There were no GDPR or PCI-DSS implications.

---

## What Went Wrong

1. **A bucket was created without a lifecycle policy.** The Platform team's Terraform modules for S3 did not require a lifecycle policy at bucket creation time.

2. **Cost ownership was implicit, not explicit.** "The team that created the bucket" and "the team that uses the bucket" were different teams. Neither felt responsible.

3. **The alerting strategy was too coarse-grained.** Account-level budget alerts cannot detect slow leaks within a single resource.

4. **ML pipeline was treated as "just a cron job."** No one applied MLOps discipline to artifact management at the time the pipeline was designed. Model artifacts were treated as disposable outputs without thinking about their long-term accumulation.

5. **S3 versioning was enabled without version lifecycle rules.** This is a known anti-pattern that CloudShop had not established guidance on.

---

## Action Items

| # | Action | Owner | Status | Due Date |
|---|---|---|---|---|
| 1 | Delete unnecessary model artifacts; retain last 3 versions | James Okonkwo | **Done** (2026-01-20) | 2026-01-20 |
| 2 | Add S3 lifecycle policy to `cloudshop-ml-models` (30-day Intelligent-Tiering, 90-day expiry, 14-day version expiry) | Alex Kim | **Done** (2026-01-25) | 2026-01-25 |
| 3 | Refactor training pipeline for explicit model versioning (keep last 3 production-promoted versions) | James Okonkwo | **Done** (2026-03-01) | 2026-03-01 |
| 4 | Add per-bucket S3 cost anomaly alert in AWS Cost Explorer | Alex Kim | **Done** (2026-01-25) | 2026-01-25 |
| 5 | Audit all S3 buckets for missing lifecycle policies; remediate gaps | Alex Kim | **Done** (2026-02-15) | 2026-02-15 |
| 6 | Write and socialize cost ownership policy: every S3 bucket requires `CostOwner` tag and lifecycle policy at creation | Sarah Chen | **Done** (2026-03-01) | 2026-03-01 |
| 7 | Enforce lifecycle policy and `CostOwner` tag in Terraform S3 module via policy-as-code (OPA or Sentinel) | Alex Kim | **Done** (2026-04-02) | 2026-04-02 |
| 8 | Add per-team AWS cost review to monthly engineering all-hands; include S3 per-bucket breakdown | Emily Torres | **Done** (2026-02-01) | 2026-02-01 |

---

## Systemic Recommendations

### Recommendation 1: Treat ML pipelines as first-class infrastructure from day one

A model training pipeline is not functionally different from a database write path — it produces persistent artifacts at volume. Artifact retention, storage tier selection, and cost attribution must be part of the initial design, not an afterthought. Going forward, any new ML pipeline requires a data lifecycle plan reviewed by Platform Engineering before merging to production.

### Recommendation 2: Cost ownership must be explicit and documented

CloudShop's growth from a monolith to microservices created many shared resources (S3 buckets, ElastiCache clusters, RDS instances) owned by cross-cutting platform teams but used by product service teams. The organizational structure does not naturally produce a single cost owner. We need a formal policy.

The accepted policy: every AWS resource must have a `CostOwner` tag naming the responsible team. Resources without this tag are flagged in the monthly cost review. New resources without the tag are blocked at Terraform apply time.

### Recommendation 3: S3 versioning and lifecycle rules must always be configured together

Any S3 bucket with versioning enabled must also have a lifecycle rule that expires non-current versions within a defined window. This should be enforced in the Terraform S3 module rather than relying on engineers to remember it.

---

## Related Documents

- [JIRA-102: S3 storage growth investigation](../jira/JIRA-102-storage-growth.md)
- [architecture/current_architecture.md](../architecture/current_architecture.md) — updated to reflect ML artifact lifecycle policy
