# CloudShop Storage Governance Policy

| Field           | Value                                      |
|-----------------|--------------------------------------------|
| Policy ID       | POL-STORAGE-002                            |
| Version         | 1.4                                        |
| Status          | Active                                     |
| Effective Date  | 2025-09-01                                 |
| Last Reviewed   | 2026-03-15                                 |
| Next Review     | 2026-09-15                                 |
| Policy Owner    | priya.patel@cloudshop.com (FinOps)         |
| Approver        | CTO — alex.morgan@cloudshop.com            |
| Applies To      | All AWS S3 buckets in account 123456789012 |

---

## 1. Purpose

This policy establishes minimum standards for AWS S3 storage management at
CloudShop to control costs, ensure data hygiene, enforce compliance with
data-retention obligations, and reduce infrastructure waste.

---

## 2. Scope

This policy applies to all teams that own or operate S3 buckets within CloudShop's
AWS account (ID: 123456789012), including but not limited to:

- data-engineering (analytics-data-bucket)
- security-ops (logs-backup-bucket)
- product-team (customer-media-bucket)
- platform-engineering (any shared or ephemeral buckets)

---

## 3. Definitions

| Term                  | Definition                                                              |
|-----------------------|-------------------------------------------------------------------------|
| Lifecycle Policy      | An S3 lifecycle configuration that automates object transition or expiry |
| Raw Data              | Unprocessed event data, logs, or telemetry stored in S3                 |
| Cold Data             | Data not accessed in the last 90 days                                   |
| Orphaned Object       | An object whose owner team cannot be identified via resource tags        |
| Policy-Compliant      | A bucket that satisfies all sections of this document                   |

---

## 4. Requirements

### 4.1 — Lifecycle Policy (MANDATORY)

**Every S3 bucket must have at least one S3 lifecycle rule configured.**

Minimum rules required:

| Object Category    | Transition to STANDARD_IA | Transition to GLACIER | Expiry         |
|--------------------|---------------------------|------------------------|----------------|
| Raw logs / events  | After 7 days              | After 30 days          | After 90 days  |
| Application data   | After 30 days             | After 90 days          | After 365 days |
| Aggregated reports | After 60 days             | After 180 days         | After 730 days |
| Media assets       | After 30 days (IT tier)   | N/A                    | None required  |

**Current violations as of 2026-05-30:**

| Bucket                  | Violation                         | Ticket      |
|-------------------------|-----------------------------------|-------------|
| logs-backup-bucket      | No lifecycle policy               | INFRA-4440  |
| analytics-data-bucket   | Remediated 2026-05-14 (INC-2026-031) — see DEPLOY-2026-089 for root cause | N/A |

### 4.2 — Raw Data Retention Limit

Raw unfiltered data (clickstream events, application logs, telemetry) **must not
be retained in S3 STANDARD storage class for more than 30 days** without explicit
written approval from the FinOps team (finops@cloudshop.com).

Any new S3 prefix receiving raw data must have a corresponding lifecycle rule
before the first write. This is a **blocking gate** in the CloudShop deployment
checklist for any deployment that modifies S3 write targets.

**Root cause of INC-2026-031:** analytics-service v2.4.0 (DEPLOY-2026-089) wrote
to `analytics-data-bucket/raw-events/` without a lifecycle rule, violating this
section.

### 4.3 — Versioning

Buckets storing business-critical data must have S3 versioning enabled.
Versioning-enabled buckets must also define a non-current version expiry rule
(recommended: 90 days) to prevent unbounded version accumulation.

**Current violations as of 2026-05-30:**

| Bucket             | Versioning Status | Non-Current Expiry Rule |
|--------------------|-------------------|------------------------|
| logs-backup-bucket | Disabled          | N/A — INFRA-4440       |

### 4.4 — Tagging

All S3 buckets must carry the following mandatory resource tags:

| Tag Key       | Description                               | Example                   |
|---------------|-------------------------------------------|---------------------------|
| team          | Owning team slug                          | data-engineering          |
| cost-center   | Cost center code                          | CC-DATA                   |
| environment   | prod / staging / dev                      | prod                      |
| data-class    | public / internal / confidential / secret | confidential              |
| managed-by    | terraform / manual / cdk                  | terraform                 |

Untagged buckets will be flagged by AWS Config rule `cloudshop-s3-tagging-required`
and escalated to the owning team's manager within 72 hours.

### 4.5 — Public Access

All S3 buckets must have Block Public Access enabled on all four settings. No
exceptions without written CISO approval (security@cloudshop.com).

### 4.6 — Cost Threshold Alerts

Each bucket with monthly costs exceeding $200 must have a CloudWatch alarm
configured to alert when daily estimated cost exceeds 150% of the 30-day average.

See: `cloudwatch/storage_alert.json` for configured alarms.

---

## 5. Enforcement

| Non-Compliance Stage | Action                                                              | Timeline |
|----------------------|---------------------------------------------------------------------|----------|
| Stage 1              | Automated Jira ticket created; team notified by email               | Day 0    |
| Stage 2              | Engineering manager notified; ticket escalated                      | Day 7    |
| Stage 3              | FinOps team applies corrective lifecycle rule on behalf of team      | Day 14   |
| Stage 4              | Cost of non-compliance charged back to team's cost center            | Day 30   |

---

## 6. Exemptions

Teams may request exemptions by submitting a request to finops@cloudshop.com with:

1. Business justification for the exemption
2. Alternative cost control measure proposed
3. Defined expiry date for the exemption (maximum 90 days)

Exemptions must be approved by the FinOps team and the Policy Owner.

---

## 7. Related Documents

| Document                             | Path                                              |
|--------------------------------------|---------------------------------------------------|
| Storage Cleanup Runbook              | runbooks/storage_cleanup_runbook.md               |
| S3 Inventory                         | storage/s3_inventory.csv                          |
| Incident INC-2026-031                | incidents/storage_cost_spike.md                   |
| Deployment DEPLOY-2026-089           | deployments/deployment_v2_4.md                    |
| CloudWatch Storage Alarms            | cloudwatch/storage_alert.json                     |
| Team Resource Mapping                | ownership/team_resource_mapping.csv               |

---

## 8. Revision History

| Version | Date       | Author                         | Change Summary                                      |
|---------|------------|--------------------------------|-----------------------------------------------------|
| 1.0     | 2024-06-01 | priya.patel@cloudshop.com      | Initial policy                                      |
| 1.1     | 2025-01-10 | priya.patel@cloudshop.com      | Added raw data 30-day limit (section 4.2)           |
| 1.2     | 2025-04-20 | priya.patel@cloudshop.com      | Added cost threshold alert requirement (section 4.6)|
| 1.3     | 2025-09-01 | marcos.rivera@cloudshop.com    | Added deployment gate for new S3 prefixes           |
| 1.4     | 2026-03-15 | priya.patel@cloudshop.com      | Updated enforcement timeline; added INC-2026-031 reference |
