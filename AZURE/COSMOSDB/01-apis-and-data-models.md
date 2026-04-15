# APIs & Data Models

> **Exam mapping:** AZ-305 *(choose the right data store)* · AZ-204 *(SDK choice)* · AI-102/103 *(vector-capable API selection)*
> **One-liner:** Cosmos DB exposes the same distributed engine under multiple wire protocols; the API is fixed at account creation and dictates tooling, indexing, features, and limits.
> **Related:** [Overview](00-overview.md) · [Partitioning](02-partitioning.md) · [Throughput](03-throughput-and-ru.md)

## The API menu

| API | Wire protocol / SDK | Engine | Best for | Vector search? |
|-----|---------------------|--------|----------|----------------|
| **NoSQL** *(native, aka SQL API)* | REST + language SDKs (C#/JS/Py/Java/Go) | Cosmos core | New apps, AI/vector, full feature set | ✅ flat / quantizedFlat / **DiskANN** |
| **MongoDB (RU)** | MongoDB wire (3.6/4.0/4.2/5.0/6.0/7.0) | Cosmos core w/ Mongo shim | Lift-and-shift of small Mongo apps | ✅ limited |
| **MongoDB vCore** *(a.k.a. Azure DocumentDB)* | MongoDB wire | **Separate** DocumentDB engine on open-source PG | Lift-and-shift of real Mongo apps, HNSW + **DiskANN** | ✅ HNSW + DiskANN GA |
| **Cassandra** | CQL v4 | Cosmos core | CQL-based apps needing Cosmos SLAs | ✅ (preview) |
| **Gremlin** | Apache TinkerPop Gremlin | Cosmos core | Graph traversal (social, fraud) — *maintenance mode* | ❌ |
| **Table** | Azure Table REST | Cosmos core | Table Storage w/ better perf + indexing | ❌ |
| **PostgreSQL** *(Citus)* | Postgres wire | Citus/Postgres, **not** Cosmos core | Distributed Postgres, relational workloads | ✅ `pgvector` + DiskANN |

> ⚠️ **"Cosmos DB for MongoDB" is two different products.** *RU-based* = Cosmos core with a Mongo shim (RU pricing). *vCore-based* = a different engine (DocumentDB), billed per vCPU/hour, with richer Mongo compatibility and HNSW indexes. For net-new Mongo workloads Microsoft steers you to **vCore**.

## Pick-an-API decision flow

1. **New app, no lock-in?** → **NoSQL**. Full feature set (hierarchical partition keys, DiskANN, hybrid search, change feed all-versions-and-deletes, analytical store, Fabric mirroring first).
2. **Existing MongoDB app?** → **MongoDB vCore** (Azure DocumentDB). Go RU only if the workload is tiny and bursty (< 10 k ops/s).
3. **Existing Cassandra app?** → Cosmos Cassandra API. Else, skip.
4. **Relational + scale-out + Postgres ecosystem?** → **Cosmos DB for PostgreSQL**.
5. **Graph?** Evaluate AGE on Postgres or a dedicated graph DB; Gremlin API is effectively legacy.

## Data model — NoSQL API

```jsonc
{
  "id": "order-123",          // required string, unique per logical partition
  "pk": "customer-42",        // partition key path declared on the container
  "items": [ { "sku": "A", "qty": 2 } ],
  "_ts": 1716150000,          // system: last-modified epoch seconds
  "_etag": "\"00001234-...\""  // system: optimistic concurrency token
}
```

Rules:
- `id` is unique **within a logical partition**, not globally.
- Every doc must carry the partition-key property (nested paths OK: `/customer/id`).
- Max doc size **2 MB**. Large blobs belong in Blob Storage + a reference.
- Arrays/objects are fully indexed by default — budget RU or tune the [indexing policy](06-indexing-and-query.md).

## Minimal C# SDK v3 example (Managed Identity, no keys)

```csharp
using Azure.Identity;
using Microsoft.Azure.Cosmos;

var client = new CosmosClient(
    accountEndpoint: "https://mycosmos.documents.azure.com:443/",
    tokenCredential: new DefaultAzureCredential());

var container = client.GetContainer("shop", "orders");

await container.CreateItemAsync(
    new { id = "order-123", pk = "customer-42", total = 99.50m },
    new PartitionKey("customer-42"));
```

Notes:
- `CosmosClient` is thread-safe and expensive — **singleton**.
- Prefer **Direct + TCP** connection mode (default in SDK v3) for P99 latency.
- `DefaultAzureCredential` + **data-plane RBAC role** replaces account keys (see [`09-security.md`](09-security.md)).

## Migration gotchas

- Mongo RU → vCore: portal migration tool is GA; keys/connection strings change; indexes rebuild.
- NoSQL across accounts: use **Data Migration Tool** or **Azure Data Factory / Synapse** pipelines; no in-place API switch.
- Serverless → Provisioned: GA in-place migration (no downtime). Reverse is also supported.

## Exam traps

- *"Switch API later"* — **false**, fixed at account creation.
- *"MongoDB API = MongoDB"* — partially true for RU (shim), truly separate engine for vCore.
- *"Gremlin is recommended for new graph workloads"* — no; treat as legacy.
- *"Table API ≡ Table Storage"* — same surface, different engine, much better indexing + throughput.

## Sources

- [Choose an API](https://learn.microsoft.com/en-us/azure/cosmos-db/choose-api)
- [NoSQL API overview](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/overview)
- [MongoDB vCore docs](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/vcore/)
- [Cosmos DB for PostgreSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/postgresql/)
- [SDK v3 (.NET) reference](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/sdk-dotnet-v3)
