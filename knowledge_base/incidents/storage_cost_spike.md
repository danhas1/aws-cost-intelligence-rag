# Incident Report: INC-2026-031

| Field              | Value                                               |
|--------------------|-----------------------------------------------------|
| Incident ID        | INC-2026-031                                        |
| Title              | analytics-data-bucket storage cost spike — 115% MoM increase |
| Severity           | SEV-2                                               |
| Status             | Resolved (cost remediation in progress)             |
| Opened             | 2026-05-12 09:05 UTC                                |
| Resolved           | 2026-05-14 16:40 UTC                                |
| Duration           | 55 hours 35 minutes                                 |
| Incident Commander | marcos.rivera@cloudshop.com (platform-engineering)  |
| Reported By        | CloudWatch alarm S3-StorageCostAnomaly-analytics-data-bucket |
| Affected Resources | analytics-data-bucket, prod-eks-cluster             |
| Related Deployment | DEPLOY-2026-089 (analytics-service v2.4.0, 2026-05-03) |
| Related Runbook    | runbooks/storage_cleanup_runbook.md                 |
| Cost Impact        | +$1,139 in May 2026 (storage + egress delta vs. April) |
| Ticket             | https://jira.cloudshop.internal/browse/INFRA-4421   |

---

## Timeline

| Time (UTC)            | Event                                                                 |
|-----------------------|-----------------------------------------------------------------------|
| 2026-05-03 14:32      | analytics-service v2.4.0 deployed to prod-eks-cluster (DEPLOY-2026-089). Kinesis Firehose stream activated, writing raw clickstream events to analytics-data-bucket/raw-events/ with no S3 lifecycle rule. |
| 2026-05-03 to 05-06   | Backfill Spark job writes 31.8 TB of historical event data to analytics-data-bucket/raw-events/backfill/. |
| 2026-05-07 09:00      | Cluster Autoscaler provisions 2 additional m6i.4xlarge nodes on prod-eks-cluster due to increased pod resource requests in v2.4.0. |
| 2026-05-12 08:47      | CloudWatch alarm S3-StorageCostAnomaly-analytics-data-bucket fires. Threshold: $150/day. Observed: $229.40/day. |
| 2026-05-12 09:05      | INC-2026-031 opened by on-call engineer marcos.rivera@cloudshop.com. |
| 2026-05-12 10:20      | Root cause identified: raw-events/ prefix in analytics-data-bucket has no lifecycle rule. v2.4.0 removed the 7-day TTL filter from the previous version (v2.3.6). |
| 2026-05-12 11:00      | sofia.chen@cloudshop.com (data-engineering, deployment owner) engaged. |
| 2026-05-12 13:30      | Kinesis Firehose delivery stream paused to halt new writes to raw-events/ prefix. |
| 2026-05-13 09:00      | FinOps team (priya.patel@cloudshop.com) confirms total projected monthly cost overrun: +$1,139. |
| 2026-05-14 11:00      | S3 lifecycle rule applied to analytics-data-bucket/raw-events/ — expire after 30 days, transition to STANDARD_IA after 7 days. |
| 2026-05-14 14:00      | Decision: retain backfill data, do not delete — ML team (james.okoro@cloudshop.com) requires it for model training. Lifecycle rule will expire it after 30 days going forward. |
| 2026-05-14 16:40      | Incident resolved. Kinesis Firehose stream re-enabled with updated prefix filter. |

---

## Root Cause Analysis

**Primary cause:** analytics-service v2.4.0 (DEPLOY-2026-089) introduced a new
Kinesis Firehose delivery stream writing raw clickstream events to
`analytics-data-bucket/raw-events/`. This prefix had no corresponding S3 lifecycle
rule. The previous version (v2.3.6) only wrote aggregated summaries to the
`aggregated/` prefix, which is covered by an existing 90-day expiry rule.

**Contributing factors:**

1. The pre-deployment checklist item "S3 lifecycle policy created for new prefix"
   was skipped. See DEPLOY-2026-089 pre-deployment checklist.
2. FinOps team review was marked SKIPPED due to Q2 roadmap deadline pressure.
3. The AWS Cost Anomaly Detector had a 3-day detection lag, meaning the spike on
   2026-05-03 was not alerted until 2026-05-12.
4. `logs-backup-bucket` also has no lifecycle policy and is accumulating ~11% MoM,
   but was not in scope for this incident. Tracked separately under INFRA-4440.

---

## Impact Assessment

| Metric                          | Value                                    |
|---------------------------------|------------------------------------------|
| Storage growth (analytics-data-bucket) | 38.2 TB → 82.4 TB (+44.2 TB)     |
| Estimated monthly cost overrun  | $1,139.00                                |
| Estimated annual projection (if unresolved) | +$13,668/year               |
| Affected teams                  | data-engineering, platform-engineering   |
| SLA breach                      | NO (storage, not availability incident)  |
| Data loss                       | NO                                       |
| Customer impact                 | NO                                       |

---

## Remediation Actions

| Action                                                        | Owner                        | Status      | Due Date   |
|---------------------------------------------------------------|------------------------------|-------------|------------|
| Apply lifecycle rule to analytics-data-bucket/raw-events/     | sofia.chen@cloudshop.com     | DONE        | 2026-05-14 |
| Apply lifecycle rule to analytics-data-bucket/raw-events/backfill/ | sofia.chen@cloudshop.com | DONE        | 2026-05-14 |
| Create S3 lifecycle policy for logs-backup-bucket             | lena.wu@cloudshop.com (security-ops) | IN_PROGRESS | 2026-06-07 |
| Add lifecycle policy check to deployment checklist            | marcos.rivera@cloudshop.com  | IN_PROGRESS | 2026-06-01 |
| Lower CloudWatch anomaly threshold from $150/day to $80/day   | marcos.rivera@cloudshop.com  | DONE        | 2026-05-15 |
| Mandate FinOps review for storage-impacting deployments       | priya.patel@cloudshop.com    | PLANNED     | 2026-06-15 |
| Rightsize redis-cluster (28% avg utilization)                 | backend-team                 | PLANNED     | 2026-06-30 |

---

## Lessons Learned

1. S3 lifecycle policies must be created before any new prefix is written to in
   production. This must be a blocking gate in the deployment checklist.
2. Cost anomaly detection lag of 3–9 days is insufficient. Real-time S3 storage
   metric alarms should be configured per bucket per prefix.
3. Backfill jobs that write large volumes of data require a dedicated cost-impact
   review, independent of the feature deployment.
4. `logs-backup-bucket` represents a secondary governance risk. Monthly growth
   without a lifecycle policy will compound costs.
