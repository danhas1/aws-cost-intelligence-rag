# Runbook: S3 Storage Cleanup and Lifecycle Remediation

| Field           | Value                                                   |
|-----------------|---------------------------------------------------------|
| Runbook ID      | RB-STORAGE-001                                          |
| Version         | 2.1                                                     |
| Last Updated    | 2026-05-15                                              |
| Updated By      | marcos.rivera@cloudshop.com (platform-engineering)      |
| Trigger         | CloudWatch alarm S3-StorageCostAnomaly-* or S3-NullLifecyclePolicy-Compliance |
| Related Policy  | policies/storage_governance_policy.md (POL-STORAGE-002) |
| Related Incident| INC-2026-031 (analytics-data-bucket storage spike)      |
| Estimated Time  | 30–90 minutes depending on scope                        |
| Permissions     | s3:PutLifecycleConfiguration, s3:ListBucket, s3:DeleteObject (requires approval for Delete steps) |

---

## When to Use This Runbook

- A CloudWatch alarm fires for S3 storage cost anomaly on any of the following buckets:
  - `analytics-data-bucket` (team: data-engineering, CC-DATA)
  - `logs-backup-bucket` (team: security-ops, CC-SECOPS)
  - `customer-media-bucket` (team: product-team, CC-PRODUCT)
- A bucket is identified as missing a lifecycle policy via AWS Config or `storage/s3_inventory.csv`
- Monthly billing review shows unexpected S3 cost growth (`billing/aws_costs_may_2026.csv`)

---

## Pre-Requisites

- [ ] AWS CLI configured with profile `cloudshop-prod` (account 123456789012)
- [ ] Read access to the affected S3 bucket confirmed
- [ ] Incident ticket open in Jira (e.g. INC-2026-031)
- [ ] Owning team notified before any delete operations
- [ ] FinOps team (priya.patel@cloudshop.com) notified if projected monthly impact > $500

---

## Step 1: Identify the Problem Bucket

```bash
# List all buckets and their storage sizes from CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=StorageType,Value=StandardStorage \
  --start-time $(date -u -d '2 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Average \
  --region us-east-1

# Check which buckets are missing lifecycle policies
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  tr '\t' '\n' | while read bucket; do
    policy=$(aws s3api get-bucket-lifecycle-configuration --bucket "$bucket" 2>&1)
    if echo "$policy" | grep -q "NoSuchLifecycleConfiguration"; then
      echo "MISSING LIFECYCLE: $bucket"
    fi
  done
```

Expected output should flag:
- `logs-backup-bucket` — no lifecycle policy (INFRA-4440, CC-SECOPS)
- `analytics-data-bucket` — remediated 2026-05-14, verify rule is present

---

## Step 2: Analyze Bucket Contents

```bash
# Example: analytics-data-bucket prefix breakdown
aws s3 ls s3://analytics-data-bucket/ --recursive --human-readable --summarize | tail -5

# Get object count and size per prefix
aws s3api list-objects-v2 \
  --bucket analytics-data-bucket \
  --prefix raw-events/ \
  --query 'length(Contents)' \
  --output text

# Check oldest objects
aws s3api list-objects-v2 \
  --bucket logs-backup-bucket \
  --query 'sort_by(Contents, &LastModified)[0]' \
  --output json
```

**Known bucket contents (as of 2026-05-30):**

| Bucket                | Prefix                      | Size    | Notes                                        |
|-----------------------|-----------------------------|---------|----------------------------------------------|
| analytics-data-bucket | raw-events/                 | ~50 TB  | Written by analytics-service v2.4 Firehose   |
| analytics-data-bucket | raw-events/backfill/        | ~31.8 TB| One-time historical backfill from DEPLOY-2026-089 |
| analytics-data-bucket | aggregated/                 | ~0.6 TB | Safe — lifecycle rule applies (90-day expiry)|
| logs-backup-bucket    | application-logs/           | ~18 TB  | No expiry; oldest objects from 2023-01-08    |
| logs-backup-bucket    | audit-logs/                 | ~6.8 TB | No expiry; regulatory hold may apply — verify|
| logs-backup-bucket    | (Standard IA)               | ~9.8 TB | No expiry or transition rule                 |
| customer-media-bucket | (all)                       | ~12.1 TB| Lifecycle active — no action needed          |

---

## Step 3: Apply a Lifecycle Policy

### 3a — analytics-data-bucket (already remediated, use as template)

```bash
cat > /tmp/analytics-lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "raw-events-expire-30d",
      "Filter": { "Prefix": "raw-events/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 7, "StorageClass": "STANDARD_IA" }
      ],
      "Expiration": { "Days": 30 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 7 }
    },
    {
      "ID": "aggregated-expire-90d",
      "Filter": { "Prefix": "aggregated/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 60, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 90 }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket analytics-data-bucket \
  --lifecycle-configuration file:///tmp/analytics-lifecycle.json
```

### 3b — logs-backup-bucket (PENDING — INFRA-4440)

> **IMPORTANT:** Before applying expiry rules to `logs-backup-bucket`, confirm
> with lena.wu@cloudshop.com (security-ops) whether `audit-logs/` is under a
> regulatory hold. Expiring audit logs prematurely may violate compliance obligations.

```bash
cat > /tmp/logs-backup-lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "application-logs-expire-90d",
      "Filter": { "Prefix": "application-logs/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 7,  "StorageClass": "STANDARD_IA" },
        { "Days": 30, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 90 }
    },
    {
      "ID": "audit-logs-glacier-365d",
      "Filter": { "Prefix": "audit-logs/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER_IR" }
      ],
      "Expiration": { "Days": 365 }
    }
  ]
}
EOF

# Requires approval from lena.wu@cloudshop.com before running
aws s3api put-bucket-lifecycle-configuration \
  --bucket logs-backup-bucket \
  --lifecycle-configuration file:///tmp/logs-backup-lifecycle.json
```

**Estimated monthly savings after INFRA-4440 completion: ~$400–$550/month**

---

## Step 4: Manual Object Deletion (Requires Approval)

> Only proceed if explicitly approved by the owning team and FinOps.
> For analytics-data-bucket backfill deletion, approval required from:
> james.okoro@cloudshop.com (ML platform — consumes backfill data)

```bash
# DRY RUN — list objects that would be deleted before executing
aws s3 rm s3://analytics-data-bucket/raw-events/backfill/ \
  --recursive \
  --dryrun

# Actual delete (only after explicit written approval)
# aws s3 rm s3://analytics-data-bucket/raw-events/backfill/ --recursive
```

---

## Step 5: Verify and Monitor

```bash
# Verify lifecycle rule is applied
aws s3api get-bucket-lifecycle-configuration --bucket analytics-data-bucket
aws s3api get-bucket-lifecycle-configuration --bucket logs-backup-bucket

# Monitor bucket size in CloudWatch (metric updates every 24 hours)
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=analytics-data-bucket Name=StorageType,Value=StandardStorage \
  --start-time $(date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Average
```

---

## Step 6: Update Incident and Close

1. Add a comment to the incident ticket (INC-2026-031 or the applicable incident) confirming:
   - Lifecycle policy applied
   - Projected monthly savings
   - Any manual deletions performed (with approval reference)
2. Confirm CloudWatch alarm `S3-NullLifecyclePolicy-Compliance` returns to OK state within 24 hours.
3. Update `storage/s3_inventory.csv` with corrected `lifecycle_policy_exists` and `policy_compliant` fields.
4. Notify FinOps (priya.patel@cloudshop.com) so cost projections can be updated.

---

## Expected Cost Impact

| Bucket                  | Current Monthly Cost | Projected Post-Remediation | Estimated Savings |
|-------------------------|---------------------|----------------------------|-------------------|
| analytics-data-bucket   | $2,113.60           | $820.00                    | ~$1,293/month     |
| logs-backup-bucket      | $694.20             | $160.00                    | ~$534/month       |
| **Total**               | **$2,807.80**       | **$980.00**                | **~$1,827/month** |
