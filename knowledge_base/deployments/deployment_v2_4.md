# Deployment Record: analytics-service v2.4.0

| Field              | Value                                      |
|--------------------|--------------------------------------------|
| Deployment ID      | DEPLOY-2026-089                            |
| Service            | analytics-service                          |
| Version            | 2.4.0                                      |
| Previous Version   | 2.3.6                                      |
| Environment        | production                                 |
| Cluster            | prod-eks-cluster                           |
| Namespace          | analytics                                  |
| Deployed By        | sofia.chen@cloudshop.com (data-engineering)|
| Deployment Date    | 2026-05-03 14:22 UTC                       |
| Deployment Method  | ArgoCD GitOps (main branch SHA: a7f3c91)   |
| Status             | Deployed — cost regression identified      |
| Rollback Available | YES (v2.3.6 image pinned in ECR)           |
| Related Incident   | INC-2026-031                               |

---

## Summary of Changes

Version 2.4.0 introduced a new real-time clickstream pipeline designed to capture
raw user interaction events for downstream ML feature engineering consumed by
recommendation-service. Key changes:

1. **Raw Event Retention** — Added a Kinesis Firehose delivery stream writing raw
   clickstream events directly to `analytics-data-bucket` under the prefix
   `raw-events/`. Previous versions applied a server-side 7-day TTL filter and
   wrote only aggregated hourly summaries.

2. **Removed S3 Prefix Filter** — The lifecycle-aware prefix `aggregated/` was the
   only path previously written. v2.4.0 writes to `raw-events/` with **no S3
   lifecycle rule** covering that prefix.

3. **Backfill Job** — A one-time Spark backfill job reprocessed 14 months of
   historical events (2025-03-01 through 2026-04-30) and wrote results to
   `analytics-data-bucket/raw-events/backfill/`. This single job deposited
   approximately **31.8 TB** of data between 2026-05-03 and 2026-05-06.

4. **EKS Resource Requests** — Pod resource requests increased from
   `cpu: 500m / memory: 1Gi` to `cpu: 2000m / memory: 4Gi`, causing the
   `prod-eks-cluster` Cluster Autoscaler to provision two additional
   `m6i.4xlarge` nodes between 2026-05-03 and 2026-05-07.

---

## Infrastructure Impact

| Resource                | Before Deployment     | After Deployment         | Change          |
|-------------------------|-----------------------|--------------------------|-----------------|
| analytics-data-bucket   | 38.2 TB               | 82.4 TB                  | +44.2 TB        |
| analytics-data-bucket cost | $878.60/month     | $1,895.20/month (est.)   | +$1,016.60/month|
| S3 PUT requests         | ~3.1M/month           | ~7.4M/month              | +4.3M/month     |
| Data egress (analytics) | 186 TB/month          | 214 TB/month             | +28 TB/month    |
| prod-eks-cluster nodes  | 8                     | 10 (autoscaler peak)     | +2 temporary    |
| EKS EC2 cost            | $17,632.00/month      | $18,240.00/month         | +$608.00/month  |

---

## Pre-Deployment Checklist

| Item                                           | Status  |
|------------------------------------------------|---------|
| Unit tests passed                              | PASS    |
| Integration tests passed                       | PASS    |
| Staging environment validated                  | PASS    |
| Cost impact assessment completed               | FAIL    |
| S3 lifecycle policy created for new prefix     | FAIL    |
| FinOps team review                             | SKIPPED |
| Storage governance policy review               | SKIPPED |

> **Note:** The cost impact assessment and lifecycle policy creation steps were
> skipped because the deployment was fast-tracked to meet the Q2 ML roadmap
> deadline. This directly caused the storage cost spike documented in INC-2026-031.

---

## Rollback Plan

```bash
# Rollback to v2.3.6 via ArgoCD
argocd app set analytics-service --revision v2.3.6
argocd app sync analytics-service

# Stop Kinesis Firehose delivery stream to analytics-data-bucket
aws firehose stop-delivery-stream-encryption \
  --delivery-stream-name cloudshop-clickstream-firehose

# Optionally bulk-delete raw-events prefix (requires data-engineering approval)
aws s3 rm s3://analytics-data-bucket/raw-events/ --recursive
```

Rollback does NOT automatically remove existing data from `analytics-data-bucket`.
Storage cost reduction requires the cleanup runbook:
`runbooks/storage_cleanup_runbook.md`.

---

## Deployment Log

```
2026-05-03 14:22:01 UTC  ArgoCD sync started (SHA a7f3c91)
2026-05-03 14:24:18 UTC  analytics-service pods rolling update initiated (3 replicas)
2026-05-03 14:31:45 UTC  All pods healthy — readiness probes passing
2026-05-03 14:32:00 UTC  Kinesis Firehose stream activated: cloudshop-clickstream-firehose
2026-05-03 14:45:00 UTC  Backfill Spark job submitted to prod-eks-cluster
2026-05-06 02:18:33 UTC  Backfill job completed — 31.8 TB written to analytics-data-bucket
2026-05-07 09:00:00 UTC  Cluster Autoscaler scaled prod-eks-cluster from 8 to 10 nodes
2026-05-12 08:47:00 UTC  CloudWatch alarm S3-StorageCostAnomaly-analytics-data-bucket fired
2026-05-12 09:05:00 UTC  INC-2026-031 opened by on-call engineer (marcos.rivera@cloudshop.com)
```
