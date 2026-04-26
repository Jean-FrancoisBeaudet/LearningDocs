# Document database query optimization

_Targets .NET 10 / C# 14 / MongoDB 7+ / Cosmos DB SDK v3. See also: [RavenDB performance](./ravendb-performance.md), [SQL index analysis](./sql-index-analysis.md), [Pagination strategies](./pagination-strategies.md), [Aggregation efficiency](./aggregation-efficiency.md), [Bulk operations](./bulk-operations.md), [Serialization overhead](./serialization-overhead.md), [Connection pooling](./connection-pooling.md)._

Document stores trade joins for embedding. The performance model that won in relational — *normalize, then join cheaply* — inverts: cross-document lookups (`$lookup` in Mongo, `JOIN` in Cosmos) are expensive enough that the right answer is usually to pre-shape the document so the query you ship at runtime is a single point read or a single covered range scan. Most "Mongo is slow" or "Cosmos is burning RUs" tickets are *modeling* failures, not query-tuning failures.

> If your query plan shows `COLLSCAN` at scale, no amount of driver tuning saves you. The fix is an index, a partition key, or a different document shape — in that order of cheapness.

This page covers MongoDB and Cosmos DB — the two document stores with first-party .NET SDKs in everyday use. RavenDB has its own page ([ravendb-performance](./ravendb-performance.md)) because its query model (background indexes, stale-by-default reads) is different enough to deserve standalone treatment.

## The modeling axis

| Decision | Embed | Reference |
|---|---|---|
| Read pattern | "I always need them together" | "I sometimes need them, separately" |
| Write pattern | Atomic updates within one doc | Independent lifecycles, separate writers |
| Cardinality | Bounded (< ~100 children) | Unbounded |
| Mongo doc-size limit | Embed must fit in 16 MB BSON | Reference scales beyond it |
| Cosmos partition | Embed → one logical partition | Reference may span partitions |

Embed wins when the access pattern is "read parent + all children together". Reference wins when children are large, mutated independently, or unbounded in count. Most modeling errors are "we normalized like SQL" — three collections to read what could have been one document.

## MongoDB: the diagnostic flow

The single most useful command:

```javascript
db.orders.find({ customerId: ObjectId("..."), status: "Active" })
         .sort({ createdAt: -1 }).limit(50).explain("executionStats")
```

Read the `winningPlan.stage`:

| Stage | Meaning | Want to see |
|---|---|---|
| `IXSCAN` | Index scan | Yes — confirms the index is used |
| `COLLSCAN` | Full collection scan | No — missing or unusable index |
| `FETCH` | Doc retrieval after index hit | Acceptable; reduce with covered queries |
| `SORT` (with `STAGE_SORT_NONE`) | In-memory sort | No — sort 32 MB cap; use index ordering |
| `PROJECTION_COVERED` | Index covers projection | Yes — fastest possible read |

`executionStats.totalDocsExamined / nReturned` is the selectivity ratio — over ~10:1 means the index is too broad or the wrong one.

### Compound index ordering — the ESR rule

Same rule as SQL Server: **Equality, Sort, Range** for column ordering in a compound index.

```javascript
// Query: WHERE customerId = X AND status = "Active" AND createdAt > T  ORDER BY createdAt DESC
// Right index: equality fields first, sort field next, range field last.
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
```

Wrong ordering forces the engine to scan past rows it doesn't want, sort in memory, or both.

### Projection from .NET

```csharp
var filter = Builders<Order>.Filter.And(
    Builders<Order>.Filter.Eq(o => o.CustomerId, customerId),
    Builders<Order>.Filter.Eq(o => o.Status, OrderStatus.Active));

var projection = Builders<Order>.Projection
    .Include(o => o.Id)
    .Include(o => o.Total)
    .Include(o => o.CreatedAt);

var page = await collection
    .Find(filter)
    .Project<OrderListItem>(projection)
    .Sort(Builders<Order>.Sort.Descending(o => o.CreatedAt))
    .Limit(50)
    .ToListAsync(ct);
```

`Project<T>` is the wire-size lever — without it, every document ships every field, every embedded array, every nested object. For documents with large embedded arrays (line items, audit logs), projection cuts wire bytes by 10–100×.

### LINQ provider V3

```csharp
// Program.cs
var settings = MongoClientSettings.FromConnectionString(connStr);
settings.LinqProvider = LinqProvider.V3;
```

V3 (default since driver 3.0) translates more LINQ constructs to the aggregation pipeline than V2 — `GroupBy` with multiple keys, `Window` functions, set operators. V2 silently fell back to client-side eval for unsupported expressions; V3 throws. Behaviorally safer, but a migration may surface latent bugs. Always read the translated pipeline:

```csharp
var queryable = collection.AsQueryable().Where(o => o.Total > 100);
Console.WriteLine(queryable.ToString());   // prints the BSON pipeline
```

### Aggregation pipeline traps

`$lookup` in a pipeline is *the* expensive operator — it's an embedded query per input doc. Always:

- Push `$match` and `$project` *before* `$lookup` to reduce input cardinality.
- Index the `from` collection on the `localField` ↔ `foreignField` pair.
- For unbounded `$lookup` results, enable `allowDiskUse: true` (otherwise the pipeline fails at the 100 MB stage limit).

```csharp
var pipeline = collection.Aggregate()
    .Match(o => o.Status == OrderStatus.Active)        // filter early
    .Project(o => new { o.Id, o.CustomerId, o.Total }) // drop unused fields
    .Lookup<Order, Customer, OrderWithCustomer>(...);  // join on a smaller set
```

The Mongo aggregation framework is more capable than `Find` (window functions, faceted search, full-text scoring) but pays for it in compile cost. For a one-shot filter+project, prefer `Find`.

### Anti-patterns (Mongo)

| Anti-pattern | Failure mode |
|---|---|
| Querying without `.explain()` for hot paths | Hidden COLLSCAN; latency spikes only at scale |
| Sort fields not in any compound index | In-memory sort; 32 MB cap then error |
| `Find(...).ToListAsync()` for large reads | Materializes whole result set; memory blow |
| Per-call `new MongoClient(...)` | Connection storm; client is meant to be singleton |
| `ObjectId.GenerateNewId()` ignored, app-generated `string` _id | Loses the time-encoded ordering of `ObjectId`; index entropy worse |
| `$lookup` after a 1M-row `$match` | Joins for every matched row; ship the join to the index instead |
| Capped collection used as a queue without a TTL index | Reads past the cap fail silently; no removal signal |

## Cosmos DB: the partition key is the only knob that matters

Everything else in Cosmos perf is a footnote next to *partition key choice*. The partition key determines:

- Which physical partition stores the doc (10 GB hard cap per logical partition).
- Whether a query is single-partition (cheap, O(matched docs)) or cross-partition (expensive fan-out, O(partitions × matched docs)).
- The throughput ceiling — RU/s is provisioned per *physical* partition; a hot key throttles the whole logical partition.

### The point-read optimization

The cheapest possible Cosmos operation is a point read by `id` + `partitionKey`. It costs **1 RU**, regardless of doc size up to 1 KB. A query for the same doc costs 2.5–5 RU minimum because it has to plan and execute, even with the partition key.

```csharp
// Point read — 1 RU.
var resp = await container.ReadItemAsync<Order>(
    id: orderId.ToString(),
    partitionKey: new PartitionKey(customerId),
    cancellationToken: ct);

// Query — 2.5+ RU even for the same doc.
var query = container.GetItemQueryIterator<Order>(
    queryDefinition: new QueryDefinition("SELECT * FROM c WHERE c.id = @id")
        .WithParameter("@id", orderId.ToString()),
    requestOptions: new() { PartitionKey = new PartitionKey(customerId) });
```

If you've ever caught yourself writing `SELECT * FROM c WHERE c.id = @id` in Cosmos — that's a point read in disguise. Use `ReadItemAsync`.

### Cross-partition queries

```csharp
var query = container.GetItemQueryIterator<Order>(
    queryDefinition: new QueryDefinition("SELECT * FROM c WHERE c.status = @s")
        .WithParameter("@s", "Active"));
    // No PartitionKey in QueryRequestOptions = cross-partition fan-out.
```

Without a partition key in `QueryRequestOptions`, the SDK fans out to *every physical partition* and merges. RU cost = sum of per-partition costs. Latency = max of per-partition latencies. For a 50-partition container and a query that hits 2 partitions, you've paid for 50.

If the query genuinely spans partitions, set `MaxConcurrency` to enable parallel fan-out:

```csharp
new QueryRequestOptions { MaxConcurrency = -1, MaxItemCount = 100 }
```

`-1` means "as parallel as the SDK can manage". Default is 0 (serial), which is almost never what you want for cross-partition.

### RU economics

| Operation | Typical RU |
|---|---|
| Point read (1 KB doc) | 1 |
| Point read (10 KB doc) | ~10 |
| Single-doc insert (1 KB) | ~5.7 |
| Query returning 1 doc (single-partition, indexed) | ~2.5 |
| Query returning 1 doc (cross-partition fan-out, 10 partitions) | ~25 |
| Indexed range query, 100 docs returned | ~10 |
| `ORDER BY` requires a composite index — without one, query fails at scale |

The `RequestCharge` is on every response — log it for your top endpoints:

```csharp
var iter = container.GetItemQueryIterator<Order>(...);
double total = 0;
while (iter.HasMoreResults)
{
    var page = await iter.ReadNextAsync(ct);
    total += page.RequestCharge;
}
_logger.LogInformation("Query consumed {RU} RU", total);
```

When you see 429 (rate-limited), it means the consumed RU/s exceeded provisioned. The SDK auto-retries with `x-ms-retry-after-ms`; after exhaustion you get a `CosmosException` with `StatusCode == TooManyRequests`. Either provision more RU/s, change the partition key to spread load, or batch via [bulk operations](./bulk-operations.md).

### Continuation tokens for keyset-style paging

Cosmos doesn't support `OFFSET` efficiently — every skipped row still costs RU. Use continuation tokens (the Cosmos analog of cursors):

```csharp
var iter = container.GetItemQueryIterator<Order>(
    queryDefinition: new QueryDefinition("SELECT * FROM c WHERE c.customerId = @c ORDER BY c.createdAt DESC")
        .WithParameter("@c", customerId),
    continuationToken: tokenFromPreviousResponse,
    requestOptions: new() { PartitionKey = new PartitionKey(customerId), MaxItemCount = 50 });

var page = await iter.ReadNextAsync(ct);
return new { Items = page.ToList(), Next = page.ContinuationToken };
```

The continuation token is opaque, encodes physical position, and is RU-free to resume from. Don't try to decode or parse it — return it to the client and accept it back.

### Anti-patterns (Cosmos)

| Anti-pattern | Failure mode |
|---|---|
| Partition key with low cardinality (e.g. `tenantId` with 3 tenants) | Hot partitions; throttling on largest tenant; can't scale |
| Cross-partition query for `id` lookups | 2.5+ RU vs 1 RU for `ReadItemAsync` |
| `OFFSET N` for deep paging | Linear RU cost in N |
| String-built queries (no parameters) | Plan cache miss; full RU charge each call |
| Unbounded `MaxItemCount` (default 100) | Each page fetch may be slow; tune to working-set size |
| Indexing every property when only 5 are queried | Write RU 2–10× higher; `IndexingPolicy` should exclude unused paths |
| `Container.CreateItemAsync` per row in a loop | N × write RU; use `TransactionalBatch` per partition |
| `SELECT *` when you only need 3 fields | Wire bytes + RU charge for hydration; project explicitly |
| Cosmos LINQ provider with non-translatable expressions | Either throws or, in older SDK, ships full set then filters client-side |

## .NET driver lifecycle (both stores)

```csharp
// MongoDB — singleton MongoClient.
builder.Services.AddSingleton<IMongoClient>(_ => new MongoClient(settings));
builder.Services.AddSingleton(sp => sp.GetRequiredService<IMongoClient>().GetDatabase("orders"));

// Cosmos DB — singleton CosmosClient with bulk + gateway settings.
builder.Services.AddSingleton(_ => new CosmosClient(connStr, new CosmosClientOptions
{
    AllowBulkExecution = true,
    ConnectionMode = ConnectionMode.Direct,
    MaxRetryAttemptsOnRateLimitedRequests = 9,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30),
}));
```

`MongoClient` and `CosmosClient` are both expensive to construct (TLS handshakes, topology discovery, gateway pinging) and thread-safe — singleton or process-scoped. Per-request construction is the canonical mistake; symptoms include socket exhaustion (see [connection pooling](./connection-pooling.md)) and 5–50 ms added to every call for handshake.

## Senior-level gotchas

- **MongoDB `ObjectId` encodes a timestamp** — sort by `_id` is implicitly time-ordered. App-generated `Guid` IDs lose this and randomize the index, increasing B-tree churn on insert.
- **Mongo's WiredTiger compresses by collection** — wide documents with repeated field names (`{ "customerName": ..., "customerEmail": ... }` × millions) compress less efficiently than narrow ones with short keys (`{ "n": ..., "e": ... }`). Doesn't justify obfuscation, but matters for very large collections.
- **`Find().Sort().Limit()` without an index that satisfies both filter and sort** triggers a `SORT` stage. Mongo caps in-memory sorts at 32 MB; past that the query *fails*, it doesn't degrade. Always check the plan for `STAGE_SORT_NONE`.
- **Cosmos partition keys are immutable** once a doc is written — picking the wrong one is a migration, not a refactor. Pick for the *most frequent query pattern*, not the most natural model attribute.
- **Cosmos `IndexingMode = Consistent` (default) charges write RU per indexed path** — for write-heavy workloads, a tighter `IndexingPolicy` (only index queried paths) can cut write RU by 50%+.
- **Cross-partition queries' continuation token is partition-aware** — if you scale out the container (re-shard), tokens from before the scale event become invalid. Restart pagination from the top.
- **`AllowBulkExecution = true` in Cosmos changes single-op cost** — operations within ~10 ms of each other batch transparently. Throughput jumps 10–100×; latency for one-shot ops adds ~10 ms wait.
- **Mongo `$exists: true` doesn't use a sparse index unless you ask** — combine with `{ field: { $exists: true, $ne: null } }` and index `{ field: 1 }` as `sparse: true`.
- **Mongo transactions are limited to 60 seconds and require a replica set** — they exist, they work, they're slower than single-doc atomicity. Design embed-first to avoid them.
- **Cosmos "auto" indexing of arrays uses the path-array notation** — `c.tags[]`, `c.items[].sku`. Indexing on `c.items` (no `[]`) only indexes the existence of the array, not its contents. Read query plans before assuming.
- **Mongo `readPreference: secondary` reads from a replica** — eventually consistent. Acceptable for analytics; wrong for "I just wrote and now I read." Use `readConcern: majority` plus the primary if read-after-write matters.
- **Cosmos `ConsistencyLevel.Session` is the right default for most apps** — `Strong` doubles RU and forbids multi-region writes; `Eventual` is fine for analytics but breaks read-your-writes.
- **The Mongo `find` cursor times out server-side after 10 minutes by default** — long-running iterations need `noCursorTimeout: true` plus explicit close, or batch the workload into smaller cursors.
- **`Skip(N).Take(M)` translates to skip-and-fetch in both stores** — same `O(N+M)` cost as SQL `OFFSET`. Use cursor / continuation-token pagination per [pagination strategies](./pagination-strategies.md).
- **Cosmos triggers (pre/post) execute server-side and consume RU on every write** — convenient for cross-cutting concerns, expensive at scale. Prefer change feed processing for fan-out logic.
- **The Mongo `BsonDocument` deserialization path is faster than mapping to `T`** — for ad-hoc admin tools and one-off queries, `BsonDocument` skips the reflection cost of poco binding. Don't use it in production code paths; it loses type safety.
- **Cosmos query plan caching is per-container per-query-string** — string-built queries with concatenated values miss the cache every call, paying parse cost. Always parameterize via `WithParameter`.
