# Partitioning

> **Exam mapping:** AZ-204 · AZ-305 *(partition-key design is the most-tested Cosmos topic)*
> **One-liner:** Cosmos DB scales by splitting data into **logical partitions** (grouped by partition-key value) that are placed on **physical partitions**; a bad key caps your scale forever.
> **Related:** [Throughput & RU](03-throughput-and-ru.md) · [Indexing & query](06-indexing-and-query.md)

## The two partition concepts

| Concept | What it is | Hard limits |
|---------|-----------|-------------|
| **Logical partition** | All items sharing the same partition-key value | **20 GB** data · 10 k RU/s if container < 50 k RU/s *(higher on larger containers, still bounded)* |
| **Physical partition** | Internal compute/storage unit that hosts ≥1 logical partitions | **50 GB** + **10 k RU/s** each; split automatically |

You don't provision physical partitions — Cosmos computes them from max-throughput + data size and **splits** transparently. Your job is the partition-*key*.

## Partition-key rules of thumb

A good key has:

1. **High cardinality** — many distinct values (millions).
2. **Even access pattern** — reads and writes spread across values.
3. **Common filter** — most queries include `WHERE pk = @x` so they hit a single partition.
4. **Immutable** — you cannot change a document's partition-key value without delete+insert.

Classic winners: `customerId`, `tenantId`, `deviceId + yyyy-MM` (synthetic), `userId`.
Classic losers: `country`, `status`, `date` alone, `true/false`.

## Anti-patterns

- **Hot partition:** one key value receives 80 % of traffic (e.g. `tenantId = "BIG_CUSTOMER"`). Symptom: 429s despite low total RU consumption. Check **Insights → Throughput → Partition key RU/s**.
- **Cardinality cliff:** few distinct values → can't split past a handful of physical partitions.
- **Unbounded logical partition:** time-series keyed only by `deviceId` → hits 20 GB after months; can't split the logical partition, must rewrite.

## Hierarchical partition keys (subpartitioning)

**Generally Available.** Declare up to **3 levels** of partition keys on the container:

```bicep
properties: {
  partitionKey: {
    paths: [ '/tenantId', '/userId', '/sessionId' ]
    kind: 'MultiHash'
    version: 2
  }
}
```

- Queries filtering a prefix (`tenantId = 'A'`) hit a **subset** of partitions — still efficient.
- Lets you keep a natural tenant key without hot partitions: Cosmos can split inside a tenant on `userId`.
- **The 20 GB logical limit applies to the full key**, so a `(tenantId, userId, sessionId)` logical partition can exceed 20 GB of *total tenant data*.

Use when you have a few big tenants alongside many small ones.

## Cross-partition queries

```sql
-- Single-partition (cheap): one RU charge per logical partition read
SELECT * FROM c WHERE c.tenantId = "A" AND c.status = "paid"

-- Cross-partition (fan-out): SDK hits every physical partition, merges
SELECT * FROM c WHERE c.status = "paid"
```

The SDK still serves cross-partition queries but charges ~RU × #physical partitions and hurts P99 latency. Always design so the hot path includes the partition key.

## Synthetic keys

When no single property fits, concatenate:

```csharp
doc.pk = $"{tenantId}|{yyyyMM}";
```

Useful to:
- Bound logical partitions by time (e.g. monthly buckets for IoT).
- Spread a hot key (`tenantId|randint(0..31)`) — read side must fan out to all 32 buckets, so use only when reads are analytical or rare.

## Diagnosing hot partitions

Portal path: **Cosmos account → Insights → Throughput → Normalized RU consumption (by PK range)**.

- Values > 80 % consistently → that physical partition is saturated.
- Also check **Metrics → `Throttled Requests`** filtered by `PartitionKeyRangeId`.
- Log Analytics KQL:

```kql
AzureDiagnostics
| where Category == "DataPlaneRequests" and statusCode_s == "429"
| summarize count() by partitionKeyRangeId_s, bin(TimeGenerated, 5m)
```

## Container-level sharing vs dedicated throughput

- **Database-shared throughput** — up to 25 containers share RU/s; cheap for many small collections, but noisy-neighbor risk.
- **Dedicated container throughput** — isolation, required for autoscale beyond certain limits, required for large containers.

## Exam traps

- *"Change partition key later"* — not possible; you must recreate the container and migrate.
- *"20 GB is the container limit"* — no, it's the **logical partition** limit; containers are effectively unlimited.
- *"Hierarchical partition keys require NoSQL API + version 2"* — yes.
- *"Use `/id` as partition key"* — acceptable only for point-read-only workloads; kills range queries.

## Sources

- [Partitioning and horizontal scaling](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)
- [Choose a partition key](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-model-partition-example)
- [Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys)
- [Service limits](https://learn.microsoft.com/en-us/azure/cosmos-db/concepts-limits)
