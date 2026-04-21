# Azure Cosmos DB — Overview

> **Exam mapping:** AZ-204 *(Develop solutions that use Azure Cosmos DB)* · AZ-305 *(Design data storage solutions)* · AI-102 / AI-103 *(Implement generative AI solutions — vector store choice)*
> **One-liner:** Globally distributed, multi-model, SLA-backed NoSQL + vector database with guaranteed single-digit-ms latency at any scale.
> **Related:** [APIs & data models](01-apis-and-data-models.md) · [Throughput & RU](03-throughput-and-ru.md) · [Consistency](04-consistency-levels.md) · [AI & vector](08-ai-and-vector-search.md)

## What Cosmos DB is

Microsoft markets Cosmos DB as the **"unified AI database"**: one service that stores operational JSON data *and* serves as a first-class vector/full-text/hybrid search store, so RAG and agent workloads don't need a separate vector DB.

Core promises (all SLA-backed):

| Guarantee | Value |
|-----------|-------|
| Read/write latency @ P99 | **< 10 ms** |
| Availability (single region) | 99.99% |
| Availability (multi-region reads) | **99.999%** |
| Throughput | Any RU/s up to tens of millions |
| Consistency | Choice of **5 levels**, not just eventual |

## Resource hierarchy

```
Azure subscription
└── Cosmos DB account              ← API choice is fixed here
    ├── Database                   ← optional shared throughput scope
    │   └── Container              ← unit of scaling, indexing, partitioning
    │       └── Item (document)    ← ≤ 2 MB, must include the partition key
    └── (Users / Permissions)      ← legacy; prefer Entra RBAC
```

- **Account** = regional endpoint set; picks API (NoSQL, MongoDB, Cassandra, …) at creation time and **cannot** change later.
- **Database** = logical grouping; can hold database-level shared throughput.
- **Container** = the real workhorse. Own throughput, indexing policy, partition key, TTL, stored procs/UDFs/triggers.
- **Item** = JSON doc up to 2 MB.

## When to pick Cosmos DB (vs alternatives)

| You need… | Pick |
|-----------|------|
| Global distribution + < 10 ms writes from anywhere | **Cosmos DB (multi-region writes)** |
| Relational joins, strict schema, T-SQL | Azure SQL / SQL MI |
| Open-source PostgreSQL w/ horizontal scale | **Cosmos DB for PostgreSQL** (Citus) |
| Dedicated vector DB with rich ANN tuning only | AI Search *or* Cosmos DB NoSQL + DiskANN |
| Cheap time-series / append-only logs | Data Explorer (Kusto) or Log Analytics |
| MongoDB workloads already in prod | **Cosmos DB for MongoDB vCore** (aka *Azure DocumentDB*) |
| Graph traversal (Gremlin) | **Cosmos DB Gremlin API** *(legacy — prefer NoSQL + graph libraries for new work)* |

Cosmos DB is *not* a cheap "first DB" for simple CRUD — RU pricing punishes unindexed scans. Use it when you need ≥2 of: global distribution, extreme scale, vector+operational unification, guaranteed low latency.

## Pricing at a glance

Two capacity modes (full detail in [`03-throughput-and-ru.md`](03-throughput-and-ru.md)):

- **Provisioned throughput** — pay per RU/s per hour. Sub-modes: **manual** (flat) and **autoscale** (10 %→max, 1.5× rate).
- **Serverless** — pay per RU consumed; capped at **1 TB / 5 000 RU/s per container**. Good fit for dev/test, spiky, low-traffic.

Storage billed separately (~$0.25/GB/month, transactional store). Backups are free for the default 2-copy periodic retention.

## Exam mapping cheatsheet

| Exam | What they ask about Cosmos DB |
|------|-------------------------------|
| **AZ-204** | SDK v3 CRUD, partition-key choice, consistency levels, change feed with Functions trigger, RU throttling/retry, Managed Identity auth. |
| **AZ-305** | Global distribution topology, SLA composition, multi-region writes + conflict resolution, HA/DR posture, choosing API, analytical store / Synapse Link. |
| **AI-102 → AI-103** | Cosmos DB as vector store (DiskANN vs AI Search), hybrid RAG, LLM response caching, AI Foundry "On Your Data" integrations. |
| **AI-300** | GenAIOps: evaluation loops that read/write to Cosmos containers, PITR for training data, responsible-AI telemetry. |

## Exam traps

- API is **immutable** after account creation.
- Serverless caps are **per container**, not per account.
- SLA tiers require **multi-region + strong-or-session + SDK-direct-mode** to hit 99.999 %.
- "Cosmos DB" without qualifier = NoSQL API; the MongoDB API is a wire-protocol shim, not the same engine.

## Sources

- [Cosmos DB overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/cosmos-db/overview)
- [NoSQL API intro](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/overview)
- [SLA for Cosmos DB](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services)
- [Service quotas & limits](https://learn.microsoft.com/en-us/azure/cosmos-db/concepts-limits)
