# CloudShop Engineering Knowledge Base

## About CloudShop

CloudShop is a mid-sized e-commerce platform serving roughly 4 million active customers across North America and Western Europe. The platform processes an average of 85,000 orders per day, peaking at over 300,000 during major sale events (Black Friday, Prime-equivalent promotions). CloudShop operates a microservices architecture on AWS, with its three core backend services handling payments, user identity, and product recommendations.

CloudShop's engineering organization consists of approximately 60 engineers split across product, platform, data, and SRE functions. The company has been public since 2022 and faces significant compliance obligations under PCI-DSS (payment data) and GDPR (EU customer data).

### Core Services

| Service | Responsibility | Primary Language | Primary Datastore |
|---|---|---|---|
| `payment-service` | Payment authorization, capture, refund, fraud scoring | Java (Spring Boot) | PostgreSQL, Redis |
| `user-service` | Authentication, session management, user profiles, preferences | Go | PostgreSQL, Redis |
| `recommendation-service` | Real-time and batch product recommendations using collaborative filtering | Python (FastAPI) | DynamoDB, S3, Redis |

---

## Purpose of This Dataset

This knowledge base is the source corpus for the **AI Engineering Organizational Memory Assistant** — a RAG-powered internal tool that allows engineers, managers, and new hires to query the institutional memory of CloudShop's engineering organization.

Engineering knowledge is notoriously difficult to preserve. Architecture Decision Records (ADRs) capture formal decisions but miss the debate that preceded them. Postmortems describe what broke but rarely surface the systemic pressures that made a failure likely. Slack threads contain the richest reasoning but are ephemeral and unsearchable at scale. Jira tickets close and are forgotten.

This dataset is designed to demonstrate that when these artifacts are combined in a retrieval-augmented system, an AI assistant can answer questions like:

- *"Why did we move from ECS to EKS?"*
- *"What caused the November 2025 payment outage, and what did we change afterward?"*
- *"Who owns the Redis cache TTL configuration for the payment service?"*
- *"What were the alternatives we considered before adopting Redis?"*

These are questions that today require tracking down the right person, hoping they remember, and hoping their answer is accurate.

---

## Folder Structure

```
knowledge_base/
├── README.md                          # This file
├── SUPPORTED_QUERIES.md               # Example questions the RAG system should answer
│
├── architecture/
│   └── current_architecture.md        # Living document: current system architecture
│
├── adrs/
│   ├── ADR-001-migrate-to-eks.md      # Decision to migrate from ECS to EKS
│   └── ADR-002-adopt-redis.md         # Decision to adopt Redis as shared cache layer
│
├── jira/
│   ├── JIRA-101-payment-latency.md    # Payment service p99 latency spike investigation
│   └── JIRA-102-storage-growth.md     # S3 storage cost growth investigation
│
├── slack/
│   └── redis-discussion.md            # Archived Slack thread: #eng-platform Redis debate
│
├── incidents/
│   ├── payment-outage-postmortem.md   # Post-mortem: payment-service outage, Nov 2025
│   └── storage-cost-spike-postmortem.md # Post-mortem: S3 cost spike, Q1 2026
│
├── migrations/
│   └── ecs-to-eks-migration.md        # Full migration plan and execution log: ECS → EKS
│
└── runbooks/
    └── payment-service-runbook.md     # Operational runbook for payment-service on-call
```

---

## How the RAG System Will Use This Data

The AI Engineering Organizational Memory Assistant ingests every document in this knowledge base and builds a vector index over chunked document segments. At query time, the system:

1. **Embeds the user's natural-language question** using the same embedding model used during ingestion.
2. **Retrieves the top-k most semantically similar document chunks** from the vector store (e.g., pgvector, Pinecone, or Weaviate).
3. **Passes the retrieved chunks as grounding context** to the generation model (Claude), along with the original question.
4. **Returns a synthesized answer** that cites the source documents by filename and section.

### Design Principles for This Corpus

- **Cross-document consistency**: The same incident, decision, or service appears in multiple files from different perspectives (a Jira ticket, a Slack thread, an ADR, a postmortem). This tests the RAG system's ability to synthesize across sources.
- **WHY over WHAT**: Every document includes the reasoning and tradeoffs behind decisions, not just the outcome. This is the information most valuable to engineers — and most often lost.
- **Temporal anchoring**: Documents include dates, which allows the system to answer questions about historical state vs. current state.
- **Structured and unstructured mix**: ADRs and runbooks are highly structured; Slack threads and postmortems are prose-heavy. A robust retrieval system must handle both.
