# Indexing & Query

> **Exam mapping:** AZ-204 *(query cost, index policy)* · AZ-305 *(design for query patterns)*
> **One-liner:** Cosmos DB indexes every property automatically — great for dev, expensive at scale. Index tuning + partition-aligned queries are the two big performance levers.
> **Related:** [Partitioning](02-partitioning.md) · [Throughput](03-throughput-and-ru.md) · [Cost & performance](10-cost-and-performance.md)

## Default indexing policy (NoSQL API)

```jsonc
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [ { "path": "/*" } ],
  "excludedPaths": [ { "path": "/_etag/?" } ]
}
```

- Every path indexed as range (equality + ordering).
- Write RU ≈ `base + N × indexed properties`. Unindexing cold paths cuts write RU noticeably.

### Modes

| Mode | Behavior |
|------|----------|
| `consistent` *(default)* | Index updated synchronously on write; queries always see fresh data. |
| `none` | No indexing — use only for point-read-only stores. |
| `lazy` | **Deprecated** (was eventually consistent). Don't use in new designs. |

## Tuning patterns

### Exclude cold paths
Large blob-like properties (raw HTML, embedded JSON, vectors stored for display only) shouldn't be indexed:

```jsonc
{
  "includedPaths": [ { "path": "/*" } ],
  "excludedPaths": [
    { "path": "/raw/*" },
    { "path": "/description/?" }
  ]
}
```

### Composite indexes for ORDER BY + filter
A query like:

```sql
SELECT * FROM c WHERE c.tenantId = @t ORDER BY c.createdAt DESC
```

…is cheap with a composite index:

```jsonc
"compositeIndexes": [
  [ { "path": "/tenantId", "order": "ascending" },
    { "path": "/createdAt", "order": "descending" } ]
]
```

Without it, Cosmos falls back to in-memory sort → higher RU + latency.

### Spatial
Declare `geography` or `geometry` paths explicitly:

```jsonc
"spatialIndexes": [
  { "path": "/location/*", "types": [ "Point", "Polygon" ] }
]
```

### Vector (NoSQL API)
Vectors need **vector embedding policy** + **vector indexes** (covered in [`08-ai-and-vector-search.md`](08-ai-and-vector-search.md)).

## Query cost mental model

RU ∝ (documents visited × property evaluation cost) + result-set size.

Cheap:
- Point read by `id + pk` = 1 RU, bypasses query engine.
- Single-partition equality on indexed path.

Expensive:
- `SELECT *` cross-partition on an unindexed property.
- `OFFSET N LIMIT M` — Cosmos still evaluates + sorts N+M docs.
- `ORDER BY` without composite index.
- Cross-partition `COUNT`, `SUM` — fans out, merges.

### Useful tricks

```sql
-- Use VALUE to flatten scalar projections (smaller payload)
SELECT VALUE c.id FROM c WHERE c.tenantId = @t

-- Continuation tokens instead of OFFSET for paging
-- (SDK handles via FeedIterator; cost stays O(pageSize))

-- TOP is cheaper than LIMIT without ORDER BY
SELECT TOP 10 * FROM c WHERE c.status = "open"
```

### Cross-partition queries
- Set `MaxConcurrency = -1` on the query to fan out across partitions in parallel.
- Keep the partition key in `WHERE` whenever possible; even a partial filter drops fan-out count.

## Seeing what queries cost

SDK:
```csharp
var it = container.GetItemQueryIterator<Order>(sql,
    requestOptions: new QueryRequestOptions { MaxConcurrency = -1 });
while (it.HasMoreResults)
{
    var page = await it.ReadNextAsync();
    _log.LogInformation("{RU} RU, {Count} docs", page.RequestCharge, page.Count);
}
```

Portal:
- **Data Explorer → "Query Stats"** tab shows retrieved doc count, index lookup time, RU breakdown.
- **Metrics → Total Request Units by Operation Type** to find the expensive operations.

## Full-text search (GA on NoSQL API)

Opt-in via container-level `fullTextPolicy`:

```jsonc
"fullTextPolicy": {
  "defaultLanguage": "en-US",
  "fullTextPaths": [ { "path": "/description", "language": "en-US" } ]
}
```

Use in queries:

```sql
SELECT TOP 20 c.id, FullTextScore(c.description, ["wireless", "headphones"]) AS score
FROM c
WHERE FullTextContainsAny(c.description, "wireless", "headphones")
ORDER BY FullTextScore(c.description, ["wireless", "headphones"])
```

Combines with vector search for hybrid RAG (see [`08-ai-and-vector-search.md`](08-ai-and-vector-search.md)).

## Exam traps

- *"Lazy indexing is preferred for write-heavy"* — deprecated.
- Composite indexes are **ordered** — `(A asc, B desc)` ≠ `(B desc, A asc)`.
- Exclude wildcard MUST end with `/*` for a subtree or `/?` for the scalar path itself.
- `OFFSET/LIMIT` is paginated but **not cheaper** than the full scan.
- Full-text search requires an explicit policy; it's not on by default.

## Sources

- [Indexing overview](https://learn.microsoft.com/en-us/azure/cosmos-db/index-overview)
- [Indexing policy](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy)
- [Composite indexes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/composite-index)
- [Query performance tips](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/troubleshoot-query-performance)
- [Full-text search on NoSQL (blog)](https://devblogs.microsoft.com/cosmosdb/new-vector-search-full-text-search-and-hybrid-search-features-in-azure-cosmos-db-for-nosql/)
