# CloudShop AWS Knowledge Base

## Company

**CloudShop** is a fictional global e-commerce company running its infrastructure entirely on AWS.
It operates three core backend services: `payment-service`, `analytics-service`, and `recommendation-service`,
all deployed as containers on a shared Amazon EKS cluster.

---

## Purpose of This Dataset

This knowledge base simulates the internal operational data of a mid-size cloud-native company.
It is designed to serve as the retrieval corpus for an AI-powered
**Cloud Cost & Infrastructure Intelligence Assistant** built with RAG (Retrieval-Augmented Generation).

All documents are internally consistent — the same resources, costs, incidents, and teams
appear across multiple files, enabling the RAG system to answer complex, multi-document questions.

---

## AWS Services Covered

`Amazon EKS` · `Amazon RDS (PostgreSQL)` · `Amazon ElastiCache (Redis)` ·
`Amazon S3` · `Amazon CloudWatch` · `AWS WAF` · `Amazon ECR` ·
`AWS Lambda` · `Amazon SNS` · `Amazon SQS` · `Amazon Route 53` · `AWS Secrets Manager`

**Named resources:** `prod-eks-cluster` · `payment-rds` · `redis-cluster` ·
`analytics-data-bucket` · `logs-backup-bucket` · `customer-media-bucket`

---

## Folder Structure

| Folder | File | Description |
|--------|------|-------------|
| `billing/` | `aws_costs_may_2026.csv` | Line-item AWS costs for May 2026 with MoM comparison vs. April, per resource and team |
| `storage/` | `s3_inventory.csv` | S3 bucket inventory: sizes, object counts, lifecycle policy status, compliance flags |
| `deployments/` | `deployment_v2_4.md` | Full deployment record for `analytics-service v2.4.0` — the root cause of the storage cost spike |
| `incidents/` | `storage_cost_spike.md` | Incident report INC-2026-031: timeline, root cause analysis, remediation actions, cost impact |
| `cloudwatch/` | `storage_alert.json` | CloudWatch alarm definitions for S3 cost anomalies, lifecycle compliance, RDS CPU, and Redis utilization |
| `policies/` | `storage_governance_policy.md` | Official storage governance policy (POL-STORAGE-002): lifecycle requirements, retention limits, enforcement |
| `runbooks/` | `storage_cleanup_runbook.md` | Step-by-step operational runbook for S3 lifecycle remediation and cost recovery |
| `infrastructure/` | `aws_architecture.md` | Full architecture reference: compute, database, cache, storage, networking, and known optimization targets |
| `ownership/` | `team_resource_mapping.csv` | Maps each AWS resource to its owning team, team lead, cost center, and open tickets |
| `ownership/` | `team_costs.csv` | Monthly cost breakdown per team with MoM change, primary cost driver, and policy violations |

---

## Example RAG Questions

- Why did storage costs increase in May 2026?
- Which deployment caused the highest AWS cost increase?
- Which S3 buckets are missing a lifecycle policy?
- Which resources violate the storage governance policy?
- Which team had the largest month-over-month cost increase?
- Which AWS resources appear underutilized or over-provisioned?
- What was the root cause of incident INC-2026-031?
- How much money can CloudShop save by remediating storage issues?
- Which team stores the most data on S3?
- What steps should an engineer follow to clean up S3 storage costs?
