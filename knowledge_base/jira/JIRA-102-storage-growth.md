# JIRA-102: S3 storage cost growth — recommendation-service ML artifacts

**Ticket ID:** JIRA-102
**Project:** CloudShop Infrastructure / Cost
**Type:** Cost Optimization / Incident
**Priority:** P2 — High
**Status:** Closed (Resolved)
**Reporter:** Alex Kim (DevOps Engineer)
**Assignee:** James Okonkwo (recommendation-service lead)
**Opened:** 2026-01-09
**Closed:** 2026-04-02
**Labels:** recommendation-service, s3, cost, ml-ops, storage

---

## Summary

CloudShop's monthly AWS S3 bill increased by 312% between October 2025 and January 2026, rising from $2,100/month to $8,650/month. The increase was traced entirely to the `cloudshop-ml-models` S3 bucket, which grew from 180 GB to 2.4 TB over the same period without triggering any cost alerts.

---

## Discovery

On 2026-01-09, Alex Kim noticed the anomaly during a routine monthly AWS Cost Explorer review. The `cloudshop-ml-models` bucket accounted for 74% of total S3 spend in December 2025.

```
Bucket                     | Dec 2025 Size | Dec 2025 Cost
---------------------------|---------------|---------------
cloudshop-ml-models        | 2.1 TB        | $6,370
cloudshop-logs             | 340 GB        | $1,030
cloudshop-static-assets    | 28 GB         | $85
cloudshop-backups          | 95 GB         | $287
```

No cost anomaly alert existed for S3 at the bucket level. CloudShop had a monthly budget alert at the AWS account level set at 120% of the prior month's total bill, but the growth was gradual enough that no single month triggered the alert threshold — costs grew 40–60% each month rather than all at once.

---

## Investigation

### Root cause

The recommendation-service training pipeline, introduced in Q3 2025 as part of a project to move from weekly batch recommendations to nightly-refreshed personalization models, was configured to write a full model snapshot to S3 after every training run.

The nightly CronJob (`recommendation-model-trainer`) produced:
- A full Scikit-learn collaborative filtering model (`model.pkl`, ~4.2 GB per run)
- A preprocessed user-item interaction matrix (`interactions.npz`, ~1.8 GB per run)
- Feature metadata (`feature_config.json`, ~12 MB per run)

With nightly runs, this produced approximately 6 GB of new data per day. Over 90 days (October–January), this accumulated ~540 GB from training artifacts alone.

The S3 bucket had **no lifecycle policy**, meaning every artifact was retained indefinitely in S3 Standard storage class (the most expensive tier). Additionally, S3 versioning was enabled on the bucket but object versions were also never expired, doubling effective storage in several cases where the training job was re-run manually during debugging sessions.

### Why no one noticed earlier

1. **No cost owner assigned to the `cloudshop-ml-models` bucket.** When the bucket was created in Q3 2025, no engineer was explicitly assigned ownership of its cost. The recommendation-service team owned the application code; the Platform Engineering team owned the bucket creation. Neither team felt responsible for ongoing cost monitoring.

2. **No ML artifact retention policy.** CloudShop had a documented data retention policy for customer data but no equivalent for ML artifacts. The training pipeline was written without a cleanup step because "we might want to roll back to an older model."

3. **S3 versioning enabled without lifecycle rules.** Versioning was enabled as a safety measure to prevent accidental deletion. Without lifecycle rules to expire old versions, it silently doubled storage in re-run scenarios.

4. **Gradual growth evaded account-level alerts.** Monthly growth of 40–60% falls below the 120% threshold alert trigger.

---

## Impact

| Metric | Value |
|---|---|
| S3 cost increase (Oct 2025 → Jan 2026) | +$6,550/month (+312%) |
| Storage volume increase | +2.22 TB |
| Duration of undetected growth | ~90 days |
| Total estimated overspend | ~$12,000 (Oct–Dec 2025) |
| Models retained that were actually needed | Last 3 versions (confirmed with James Okonkwo) |
| Models retained that were unnecessary | ~87 model snapshots |

---

## Resolution

### Immediate cleanup (2026-01-20)

James Okonkwo manually audited the `cloudshop-ml-models` bucket and deleted all but the 3 most recent model versions, reducing bucket size from 2.4 TB to 18 GB. S3 Standard storage cost fell from $8,650/month to approximately $55/month.

### Structural fixes (implemented by 2026-04-02)

**Fix 1: S3 lifecycle policy**
A lifecycle policy was added to the `cloudshop-ml-models` bucket:
- Objects with prefix `models/` transition to S3 Intelligent-Tiering after 30 days.
- Objects with prefix `models/` are deleted after 90 days.
- Object versions (non-current) are deleted after 14 days.

**Fix 2: Model versioning strategy**
The training pipeline was refactored to implement explicit model versioning:
- Models are tagged with semantic version (`major.minor.patch`) and a training run ID.
- The pipeline retains the last 3 production-promoted model versions indefinitely (using a `keep=true` object tag, which is excluded from the lifecycle expiry rule).
- All other artifacts are subject to the 90-day lifecycle rule.

**Fix 3: Cost alerting**
- Bucket-level cost alerts added in AWS Cost Explorer for `cloudshop-ml-models`: alert at 150% of 7-day rolling average spend.
- Monthly cost review now includes per-bucket S3 breakdown (owned by Alex Kim, reviewed in monthly #infra-costs Slack channel).

**Fix 4: Cost ownership policy**
A cost ownership policy was drafted and accepted by Engineering leadership:
- Every S3 bucket must have a designated cost owner (a named engineer or team), documented via a `CostOwner` bucket tag.
- New buckets require a lifecycle policy defined at creation time or an explicit documented exception.
- This policy is enforced via a Terraform policy check in the infrastructure CI pipeline.

---

## Action Items

| # | Action | Owner | Status | Due |
|---|---|---|---|---|
| 1 | Delete unnecessary model artifacts from `cloudshop-ml-models` | James Okonkwo | Done | 2026-01-20 |
| 2 | Add S3 lifecycle policy to `cloudshop-ml-models` | Alex Kim | Done | 2026-01-25 |
| 3 | Refactor training pipeline for explicit model versioning | James Okonkwo | Done | 2026-03-01 |
| 4 | Add bucket-level S3 cost alert | Alex Kim | Done | 2026-01-25 |
| 5 | Audit all S3 buckets for missing lifecycle policies | Alex Kim | Done | 2026-02-15 |
| 6 | Draft and socialize cost ownership policy | Sarah Chen | Done | 2026-03-01 |
| 7 | Enforce lifecycle policy requirement in Terraform CI | Alex Kim | Done | 2026-04-02 |

---

## Lessons Learned

1. **Infrastructure ownership and cost ownership are different.** The team that creates a resource is not always the team responsible for its ongoing cost. These responsibilities must be explicitly assigned.

2. **S3 versioning without lifecycle rules is a cost trap.** Enabling versioning without expiring old versions silently compounds storage costs. The two configurations should always be set together.

3. **ML pipelines need MLOps discipline from day one.** A model training pipeline is not "just a cron job" — it produces artifacts that accumulate at volume. Artifact retention strategy should be part of the initial design.

4. **Gradual cost growth evades threshold alerts.** Account-level budget alerts are insufficient for detecting slow leaks. Per-bucket, per-service cost visibility is necessary.

---

## Related Documents

- [incidents/storage-cost-spike-postmortem.md](../incidents/storage-cost-spike-postmortem.md) — postmortem with blameless analysis and systemic recommendations
- [architecture/current_architecture.md](../architecture/current_architecture.md) — updated to reflect lifecycle policy and model versioning strategy
