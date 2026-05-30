# CloudShop AWS Infrastructure Architecture

| Field          | Value                                   |
|----------------|-----------------------------------------|
| Document ID    | ARCH-001                                |
| Version        | 3.2                                     |
| Last Updated   | 2026-05-15                              |
| Author         | marcos.rivera@cloudshop.com             |
| AWS Account    | 123456789012                            |
| Primary Region | us-east-1 (N. Virginia)                 |
| DR Region      | us-west-2 (Oregon) — cold standby       |

---

## Overview

CloudShop runs a microservices-based e-commerce platform entirely on AWS.
All production workloads are hosted in `us-east-1`. Three core services are
deployed as containerized applications on a shared EKS cluster.

---

## Core Services

| Service                | Language | Team                | Deployment Target    | Primary Data Store       |
|------------------------|----------|---------------------|----------------------|--------------------------|
| payment-service        | Go 1.22  | payments-team       | prod-eks-cluster     | payment-rds (PostgreSQL) |
| analytics-service      | Python 3.12 | data-engineering | prod-eks-cluster     | analytics-data-bucket (S3) |
| recommendation-service | Python 3.12 | ml-platform      | prod-eks-cluster     | customer-media-bucket (S3), redis-cluster |

---

## Compute: prod-eks-cluster

| Parameter               | Value                                     |
|-------------------------|-------------------------------------------|
| Cluster Name            | prod-eks-cluster                          |
| EKS Version             | 1.30                                      |
| Region                  | us-east-1                                 |
| Node Group              | cloudshop-prod-ng-01                      |
| Instance Type           | m6i.4xlarge (16 vCPU / 64 GB RAM)        |
| Node Count (baseline)   | 8                                         |
| Node Count (peak May 2026) | 10 (Cluster Autoscaler triggered by analytics-service v2.4 resource requests) |
| Min Nodes               | 6                                         |
| Max Nodes               | 16                                        |
| Monthly Cost (May 2026) | $18,240.00                                |
| Monthly Cost (April 2026) | $17,632.00                              |
| Networking              | VPC: vpc-0a1b2c3d, private subnets only  |
| Ingress                 | AWS ALB Ingress Controller + WAF (prod-waf-acl) |
| Service Mesh            | AWS App Mesh                              |
| Observability           | CloudWatch Container Insights             |
| IAM                     | IRSA (IAM Roles for Service Accounts)     |

**Namespace layout:**

| Namespace     | Services Running                          |
|---------------|-------------------------------------------|
| payments      | payment-service (3 replicas)              |
| analytics     | analytics-service (3 replicas)            |
| ml-platform   | recommendation-service (4 replicas)       |
| monitoring    | Prometheus, Grafana                       |
| ingress-nginx | ALB Ingress Controller                    |

---

## Database: payment-rds

| Parameter            | Value                                       |
|----------------------|---------------------------------------------|
| Identifier           | db-pay-prod-01                              |
| Engine               | PostgreSQL 15.4                             |
| Instance Class       | db.r6g.2xlarge (8 vCPU / 64 GB RAM)        |
| Deployment           | Multi-AZ                                    |
| Storage              | 920 GB gp3, IOPS: 6000                      |
| Storage Encryption   | Enabled (AWS KMS, key: cloudshop-rds-key)  |
| Backup Retention     | 14 days                                     |
| Maintenance Window   | Sunday 03:00–04:00 UTC                      |
| Monthly Cost (May 2026) | $4,410.00 (instance + storage)           |
| Avg CPU Utilization  | 72% (approaching alarm threshold of 85%)    |
| Read Replica         | None — under evaluation (INFRA-4455)        |
| Depends On           | payment-service                             |

---

## Cache: redis-cluster

| Parameter              | Value                                       |
|------------------------|---------------------------------------------|
| Cluster ID             | redis-cld-prod                              |
| Engine                 | ElastiCache for Redis 7.2                   |
| Node Type              | cache.r6g.large (2 vCPU / 13.07 GB RAM)    |
| Number of Nodes        | 3 (cluster mode disabled, 1 primary + 2 replicas) |
| Encryption at Rest     | Enabled                                     |
| Encryption in Transit  | Enabled (TLS)                               |
| Avg Memory Utilization | 28% (30-day average) — underutilized        |
| Recommended Rightsize  | cache.r6g.medium — estimated saving $720/month |
| Monthly Cost (May 2026)| $2,160.00                                   |
| Depends On             | recommendation-service, payment-service     |
| CloudWatch Alarm       | ElastiCache-LowMemoryUtilization-redis-cluster (ALARM) |
| Ticket                 | INFRA-4452 (rightsizing under review)       |

---

## Object Storage: S3 Buckets

### analytics-data-bucket

| Parameter          | Value                                                                  |
|--------------------|------------------------------------------------------------------------|
| Bucket Name        | analytics-data-bucket                                                  |
| Region             | us-east-1                                                              |
| Purpose            | Raw clickstream events and aggregated analytics reports for analytics-service |
| Total Size (May)   | 82.4 TB (SPIKE from 38.2 TB — see INC-2026-031, DEPLOY-2026-089)     |
| Lifecycle Policy   | Remediated 2026-05-14 (raw-events/ expires 30d, IA at 7d)             |
| Versioning         | Enabled                                                                |
| Monthly Cost       | $2,113.60                                                              |
| Data Feeds         | Kinesis Firehose (cloudshop-clickstream-firehose)                      |
| Consumers          | analytics-service (Spark jobs), recommendation-service (feature store) |

### logs-backup-bucket

| Parameter          | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| Bucket Name        | logs-backup-bucket                                                    |
| Region             | us-east-1                                                             |
| Purpose            | Application and audit log archival from all services and prod-eks-cluster |
| Total Size (May)   | 34.6 TB (24.8 TB Standard + 9.8 TB Standard-IA)                      |
| Lifecycle Policy   | NONE — governance violation; INFRA-4440 open                          |
| Versioning         | Disabled — governance violation                                       |
| Monthly Cost       | $694.20                                                               |
| Last Accessed      | 2026-02-14 (compliance audit)                                         |
| Growth Rate        | ~11% per month (unbounded)                                            |
| Owner              | security-ops (lena.wu@cloudshop.com)                                  |

### customer-media-bucket

| Parameter          | Value                                                                 |
|--------------------|-----------------------------------------------------------------------|
| Bucket Name        | customer-media-bucket                                                 |
| Region             | us-east-1                                                             |
| Purpose            | Product images, thumbnails, and media assets served via CloudFront    |
| Total Size (May)   | 12.1 TB (Intelligent-Tiering, ~42% in archive tier)                  |
| Lifecycle Policy   | Active — transition to IT at 30 days; non-current expiry 90 days     |
| Versioning         | Enabled                                                               |
| Monthly Cost       | $398.50                                                               |
| Consumers          | recommendation-service (image embeddings), CloudFront CDN             |
| Policy Compliant   | YES                                                                   |

---

## Networking

| Resource                | Value                                          |
|-------------------------|------------------------------------------------|
| VPC                     | vpc-0a1b2c3d (CIDR: 10.0.0.0/16)             |
| Private Subnets         | 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24 (3 AZs)|
| Public Subnets          | 10.0.101.0/24, 10.0.102.0/24, 10.0.103.0/24  |
| NAT Gateway             | 3 × NAT Gateway (one per AZ)                  |
| Load Balancer           | AWS ALB (prod-alb-01)                          |
| WAF                     | prod-waf-acl (CloudShop rules + AWS managed)   |
| Route 53 Hosted Zones   | cloudshop.com (public), cloudshop.internal (private) |

---

## Observability

| Tool               | Purpose                                          |
|--------------------|--------------------------------------------------|
| CloudWatch Metrics | AWS native metrics for all resources             |
| CloudWatch Alarms  | Cost anomaly, storage, CPU, cache utilization    |
| CloudWatch Logs    | Pod logs, RDS slow query, VPC Flow Logs          |
| Container Insights | EKS pod/node level metrics                       |
| AWS Cost Explorer  | Monthly spend analysis and anomaly detection     |
| AWS Config         | Resource compliance rules (tagging, lifecycle)   |

---

## Known Issues and Optimization Targets (May 2026)

| Resource                | Issue                                               | Ticket      | Potential Saving |
|-------------------------|-----------------------------------------------------|-------------|-----------------|
| analytics-data-bucket   | Storage spike from DEPLOY-2026-089 (INC-2026-031)  | Remediated  | ~$1,293/month   |
| logs-backup-bucket      | No lifecycle policy, growing ~11% MoM               | INFRA-4440  | ~$534/month     |
| redis-cluster           | 28% avg memory utilization — oversized              | INFRA-4452  | ~$720/month     |
| payment-rds             | No read replica; 72% avg CPU; single write endpoint | INFRA-4455  | Performance risk |
| prod-eks-cluster        | Cluster Autoscaler provisioned 2 extra nodes post-DEPLOY-2026-089; since scaled back | Resolved | — |
