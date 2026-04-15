# Change Feed & Analytics

> **Exam mapping:** AZ-204 *(Functions + Cosmos trigger)* · AZ-305 *(HTAP patterns, Synapse Link, Fabric mirroring)* · AI-300 *(analytics pipelines feeding GenAIOps)*
> **One-liner:** Cosmos DB emits a persistent, partition-ordered **change feed** of every insert/update; combine with analytical store / Fabric mirroring for HTAP without ETL.
> **Related:** [Throughput](03-throughput-and-ru.md) · [AI & vector](08-ai-and-vector-search.md)

## Change feed modes

| Mode | What you get | Deletes? | Notes |
|------|--------------|----------|-------|
| **Latest version** *(default)* | Final state of each changed doc, in partition order | ❌ (use TTL + "marker" doc workaround) | GA since launch |
| **All versions and deletes** | Every intermediate update + delete tombstones | ✅ | GA; requires **continuous backup** on the account |

Retention: since account creation (latest-version mode) or the continuous-backup window (all-versions mode, max 30 days).

## Consumers

### 1. Azure Functions Cosmos trigger

Zero-infra way to process change feed:

```csharp
[Function("ProjectOrders")]
public async Task Run(
    [CosmosDBTrigger(
        databaseName: "shop",
        containerName: "orders",
        Connection = "CosmosDb",                     // Managed-Identity-backed
        LeaseContainerName = "leases",
        CreateLeaseContainerIfNotExists = true,
        MaxItemsPerInvocation = 500)]
    IReadOnlyList<Order> changes, ILogger log)
{
    foreach (var o in changes) await _projector.ApplyAsync(o);
}
```

**Leases** container stores per-partition checkpoints; it's normal Cosmos data and counts against RU. Size it at ~400 RU/s for small workloads.

### 2. Change-feed processor library (SDK v3)

More control; runs anywhere (AKS, Container Apps, VM). Supports elastic scaling across instances via lease rebalancing.

```csharp
var processor = container
    .GetChangeFeedProcessorBuilder<Order>("ordersProjector",
        (changes, ct) => HandleAsync(changes, ct))
    .WithInstanceName(Environment.MachineName)
    .WithLeaseContainer(leaseContainer)
    .WithStartTime(DateTime.UtcNow.AddHours(-1))
    .Build();

await processor.StartAsync();
```

### 3. Kafka connectors / Event Hubs bridges

Community connectors pull from change feed into Kafka/Event Hubs — useful when existing streaming infra already exists.

## Common patterns

- **Materialized views** — denormalize for a query your primary schema can't answer cheaply (fan-out of order line items keyed by `productId`).
- **CQRS** — write model in one container, projections in another container or search index.
- **Cache invalidation** — push updates into Redis / CDN purge via change feed.
- **Audit log / outbox** — all-versions mode replaces the transactional outbox pattern when you don't need cross-system atomicity.

## Analytical store

Column-oriented, auto-synced copy of your container data.
- **Separate storage** — no RU impact on transactional workload.
- **Sync lag** ~2 min from commit.
- Queryable from **Azure Synapse Link** (serverless SQL / Spark) without ETL.
- Schema inferred; schema changes are auto-handled in *schema-agnostic* mode (NoSQL API) or *well-defined* mode.

Enable per-container at creation:

```bicep
resource c 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2024-05-15' = {
  properties: {
    resource: {
      id: 'orders'
      analyticalStorageTtl: -1    // enable + retain forever
    }
  }
}
```

> ⚠️ `analyticalStorageTtl` must be set **at container creation**; you can't retro-enable on an existing container.

## Microsoft Fabric mirroring

The newer, recommended HTAP path. Cosmos DB → **Fabric OneLake** in Delta format, continuously synced.

- No analytical store required; simpler to set up.
- Queryable from Fabric SQL, KQL, notebooks, Power BI.
- Eventual consistency (seconds-to-minutes lag).
- Cost: no Cosmos RU impact; Fabric capacity consumes CUs during query.

For net-new designs in 2026, **prefer Fabric mirroring** over Synapse Link unless you already have a Synapse-heavy estate.

## Change feed vs Event Grid vs Event Hubs

| Need | Use |
|------|-----|
| Reliable, ordered "someone changed this doc" stream | **Change feed** |
| Fan-out notifications to many subscribers (webhooks, Functions, Logic Apps) | **Event Grid** |
| High-throughput ingestion pipeline (IoT, telemetry) | **Event Hubs** |

Event Grid has a **Cosmos DB resource-level** source for data-plane events (preview → GA depending on region); verify current status before committing.

## Exam traps

- Change feed is **ordered within a partition**, not globally.
- Latest-version mode **doesn't expose deletes** — use all-versions mode (requires continuous backup).
- Enabling analytical store retroactively on an existing container = **not supported**.
- Functions Cosmos trigger needs a **lease container**; shared leases between functions = data corruption.
- Fabric mirroring ≠ Synapse Link; different pipelines, both valid.

## Sources

- [Change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed)
- [Change feed modes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-modes)
- [Functions Cosmos trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2-trigger)
- [Change feed processor](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor)
- [Analytical store / Synapse Link](https://learn.microsoft.com/en-us/azure/cosmos-db/synapse-link)
- [Fabric mirroring for Cosmos DB](https://learn.microsoft.com/en-us/fabric/database/mirrored-database/azure-cosmos-db)
