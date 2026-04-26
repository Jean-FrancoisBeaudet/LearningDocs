# RavenDB performance

_Targets .NET 10 / C# 14 / RavenDB 6+. See also: [Document database query optimization](./document-database-query-optimization.md), [Bulk operations](./bulk-operations.md), [Streaming patterns](./streaming-patterns.md), [Pagination strategies](./pagination-strategies.md), [Aggregation efficiency](./aggregation-efficiency.md), [Connection pooling](./connection-pooling.md)._

RavenDB's performance model is unlike both relational and other document stores in one respect: **indexes are background, queries are stale-tolerant by default.** Writes return as soon as the document is on disk; the indexer catches up asynchronously. Queries return whatever the index has *now*, and the response includes an `IsStale` flag that tells you whether the indexer is behind. This is the source of most "RavenDB feels weird" first impressions — and the source of most of its throughput advantages.

> If you're calling `WaitForNonStaleResults()` on every query, you've turned RavenDB back into a synchronous-write database and given up its main performance feature. Embrace eventual consistency at the read site, or you're paying for an architecture you're not using.

## The asynchronous-indexer model

```
[Write doc]  →  [On-disk in <ms]  →  [SaveChangesAsync returns]
                      ↓
              [Indexer batch]  →  [Index entries written]  →  [Queries see new data]
```

The indexer runs at a throttled priority on its own thread. Under load, indexes fall behind — `IsStale = true` is normal, not an error. The fix is almost never "wait for non-stale"; it's *design the read site to tolerate staleness* (a few hundred ms in steady state, seconds during heavy writes).

When you actually need read-your-writes (e.g. user just clicked Save, page reloads):

```csharp
var orders = await session.Query<Order>()
    .Customize(c => c.WaitForNonStaleResults(TimeSpan.FromSeconds(5)))
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);
```

This blocks the *query* until the indexer catches up to the write that caused the read — measured against the etag, not wall-clock. Use sparingly: in admin UIs and within unit tests, not on a hot endpoint.

## Auto-indexes vs static indexes

RavenDB auto-creates an index the first time it sees a query shape it can't satisfy. That index then optimizes future queries with the same shape, and merges with overlapping shapes over time.

```csharp
// First call — auto-index "Auto/Orders/ByCustomerIdAndStatus" created in the background.
var orders = await session.Query<Order>()
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
    .ToListAsync(ct);
```

Auto-indexes are great for development and for queries you couldn't predict. Their costs:

- **First call is slow** — query waits for index creation. Use `WaitForNonStaleResults` only here.
- **Index proliferation** — every distinct shape spawns a new auto-index. A poorly-disciplined codebase can end up with hundreds, each consuming RAM and disk.
- **Merging is heuristic** — two queries that differ by one field may produce two indexes that the merger never combines. The Studio's "Index Merge Suggestions" surfaces these.

For production hot paths, **define static indexes** explicitly. They're versioned, deployable, can include MapReduce, and their lifecycle is under your control:

```csharp
public class Orders_ByCustomerStatus : AbstractIndexCreationTask<Order>
{
    public Orders_ByCustomerStatus()
    {
        Map = orders => from o in orders
                        select new { o.CustomerId, o.Status, o.CreatedAt, o.Total };

        Index(o => o.CustomerId, FieldIndexing.Default);
        Store(o => o.Total, FieldStorage.Yes);   // covers projections without doc fetch
    }
}

// Deploy at startup.
await IndexCreation.CreateIndexesAsync(typeof(Orders_ByCustomerStatus).Assembly, store);
```

Querying it:

```csharp
var page = await session.Query<Order, Orders_ByCustomerStatus>()
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .ToListAsync(ct);
```

`FieldStorage.Yes` is the equivalent of SQL `INCLUDE` — the value lives in the index entry and projection queries never fetch the document. Use for analytical/reporting queries on hot fields.

## Map-reduce indexes

Pre-aggregated reads at index build time, not at query time. Conceptually a SQL materialized view that auto-refreshes:

```csharp
public class Orders_DailyRevenueByCustomer : AbstractIndexCreationTask<Order, Orders_DailyRevenueByCustomer.Result>
{
    public class Result { public string CustomerId; public DateOnly Day; public decimal Total; public int Count; }

    public Orders_DailyRevenueByCustomer()
    {
        Map = orders => from o in orders
                        select new Result
                        {
                            CustomerId = o.CustomerId,
                            Day = DateOnly.FromDateTime(o.CreatedAt),
                            Total = o.Total,
                            Count = 1
                        };

        Reduce = results => from r in results
                            group r by new { r.CustomerId, r.Day } into g
                            select new Result
                            {
                                CustomerId = g.Key.CustomerId,
                                Day = g.Key.Day,
                                Total = g.Sum(x => x.Total),
                                Count = g.Sum(x => x.Count)
                            };
    }
}
```

Query cost is `O(matched groups)`, independent of input doc count. Use for dashboards, reporting, anything that aggregates millions of rows for a small chart. The trade is index build time and storage — RavenDB rebuilds incrementally on writes, which is cheap, but the initial build of a 100M-row reduce can take hours.

## Streaming queries

Materializing a large query into a `List<T>` defeats RavenDB's working memory model: the session caches every loaded entity, change tracking grows linearly. For reads of more than ~1000 docs, stream:

```csharp
await using var enumerator = await session.Advanced.StreamAsync(
    session.Query<Order, Orders_ByCustomerStatus>()
           .Where(o => o.Status == OrderStatus.Active),
    cancellationToken: ct);

while (await enumerator.MoveNextAsync())
{
    Order o = enumerator.Current.Document;
    await ProcessAsync(o, ct);
}
```

Streaming bypasses the session's identity map and change tracker entirely — entities are not tracked, so `SaveChangesAsync` won't persist mutations on streamed docs. For the rare case where you need to write back, load into a separate session per batch.

## Session boundaries

A `IAsyncDocumentSession` is the unit of work — it caches, tracks, dedupes, and batches. Three default limits matter:

| Limit | Default | Effect when exceeded |
|---|---|---|
| `MaxNumberOfRequestsPerSession` | 30 | Throws `InvalidOperationException` |
| Identity-map size | unbounded | Memory grows; GC pressure |
| Session lifetime | request-scoped (in ASP.NET Core convention) | None enforced; long-lived sessions leak |

The 30-request cap is deliberate: it's a tripwire against accidental N+1. If you hit it, the answer is rarely "raise the limit" — it's "load in batches via `Include` or `LoadAsync` of a list":

```csharp
var orderIds = ...;          // list of 200 IDs

// Bad: 200 individual loads → exceeds 30-request cap.
foreach (var id in orderIds) await session.LoadAsync<Order>(id);

// Good: one batch load.
Dictionary<string, Order> orders = await session.LoadAsync<Order>(orderIds);
```

`LoadAsync(IEnumerable<string>)` issues one server request and returns a dictionary — the cheapest possible bulk fetch.

For long-running workers (a hosted service that processes for hours), use a fresh session per logical operation:

```csharp
using var session = store.OpenAsyncSession();
// do one batch of work
await session.SaveChangesAsync(ct);
// session disposed; identity map released
```

## Bulk insert

For loads of >10k docs, `BulkInsert` bypasses the session entirely and ships docs over a streaming HTTP connection without the per-doc command overhead:

```csharp
await using var bulk = store.BulkInsert(token: ct);
await foreach (var o in source.WithCancellation(ct))
    await bulk.StoreAsync(o);
// disposing the bulk insert flushes and waits for ack
```

Throughput: 50–100k docs/sec on modest hardware vs 1–5k/sec via session `StoreAsync` + `SaveChangesAsync`. The trade: no change tracking, no session events, no plug-in `IDocumentStoreListener` hooks fire — bulk is a side-channel.

For sub-1000-doc loads, the regular session pattern is fine and keeps the conveniences (events, listeners, identity).

## Patches and PatchByQueryOperation

Server-side mutation without round-tripping the doc:

```csharp
// Single doc
session.Advanced.Patch<Order, decimal>(orderId, o => o.Total, newTotal);
await session.SaveChangesAsync(ct);

// Set-based update across millions of docs
var op = await store.Operations.SendAsync(new PatchByQueryOperation(@"
    from Orders
    where CreatedAt < $cutoff
    update {
        this.Status = 'Archived';
        this.ArchivedAt = new Date();
    }", parameters: new() { ["cutoff"] = cutoff }), token: ct);

await op.WaitForCompletionAsync(TimeSpan.FromMinutes(10));
```

`PatchByQueryOperation` is the equivalent of SQL `UPDATE ... WHERE` — set-based, server-side, no round-trips per doc. The bottleneck is the indexer keeping up with the rewrites; for very large patches, raise the index priority temporarily.

## Subscriptions for change feeds

A push-based stream of documents matching a predicate, with server-side acknowledgment and resume tokens:

```csharp
var name = await store.Subscriptions.CreateAsync<Order>(
    o => o.Status == OrderStatus.Pending);

await using var worker = store.Subscriptions.GetSubscriptionWorker<Order>(name);
await worker.Run(async batch =>
{
    foreach (var item in batch.Items) await ProcessAsync(item.Result, ct);
}, ct);
```

The worker resumes from the last acknowledged batch on restart — exactly-once delivery semantics within a single subscriber, at-least-once across multiple subscribers. Use for outbox, projection refresh, integration sinks. Cheaper and more flexible than polling.

## Sharding (RavenDB 6+)

Sharded clusters partition documents across nodes by a *shard key* (typically embedded in the document ID prefix). Like Cosmos:

- The shard key choice is the single biggest perf decision.
- Single-shard queries are cheap; multi-shard fan out and merge.
- Re-sharding is a migration, not an operation.

For typical multi-tenant apps, the tenant ID as a shard prefix (`tenants/acme/orders/123`) keeps each tenant's reads on one shard.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `WaitForNonStaleResults()` on every query | Throughput halves; you've made writes synchronous |
| `session.Query<T>()` without an existing static index for hot paths | Auto-index proliferation; first call slow |
| `LoadAsync(id)` in a loop | Hits the 30-request cap; throws |
| `StoreAsync` then `SaveChangesAsync` for 100k docs | Session memory blows; use `BulkInsert` |
| `Include`-ing 50 references that may not exist | Network bytes for null payloads; project instead |
| `OrderBy` on a non-indexed field | Forces a sort over the result set |
| Long-lived session in a worker | Identity map grows unbounded; OOM after hours |
| Patching a doc you also loaded into the session | `SaveChangesAsync` overwrites the patch with the loaded version |
| `session.Query<T>().ToList()` for pagination | RavenDB caps at 1024 results unless overridden — silent truncation |
| Using `store.OpenSession()` (sync) in async code | Blocks the I/O thread; always `OpenAsyncSession` |
| Multiple `DocumentStore` instances per database | Each does topology discovery; treat `DocumentStore` as singleton |
| `WaitForIndexesAfterSaveChanges` in production write path | Blocks the writer until all indexes catch up; latency spike |

## Worked example: dashboard with 5M orders

Goal: render "today's revenue per customer" for a tenant with 5M orders, sub-100ms. Naive approach: query orders, group in C#. With 5M docs, that's a TB of data and minutes of latency.

Right approach: a map-reduce static index, queried by tenant + day:

```csharp
public class Orders_RevenueByDay : AbstractIndexCreationTask<Order, Orders_RevenueByDay.Result>
{
    public class Result { public string TenantId; public string CustomerId; public DateOnly Day; public decimal Total; }

    public Orders_RevenueByDay()
    {
        Map = orders => from o in orders
                        select new Result { TenantId = o.TenantId, CustomerId = o.CustomerId,
                                            Day = DateOnly.FromDateTime(o.CreatedAt), Total = o.Total };
        Reduce = rs => from r in rs
                       group r by new { r.TenantId, r.CustomerId, r.Day } into g
                       select new Result { TenantId = g.Key.TenantId, CustomerId = g.Key.CustomerId,
                                           Day = g.Key.Day, Total = g.Sum(x => x.Total) };
    }
}

var today = DateOnly.FromDateTime(DateTime.UtcNow);
var revenue = await session.Query<Orders_RevenueByDay.Result, Orders_RevenueByDay>()
    .Where(r => r.TenantId == tenantId && r.Day == today)
    .ToListAsync(ct);
```

Query cost is `O(distinct customers active today)`, regardless of total order count. Sub-50 ms for any tenant size. The index handles incremental rebuilds on every new write — invisible to the read site.

## Senior-level gotchas

- **`session.Query<T>()` returns a maximum of 1024 documents by default**, even with `.Take(int.MaxValue)`. Override at the store: `store.Conventions.MaxNumberOfRequestsPerSession` and `store.Conventions.MaxResultsForGetTermsCount`. Forgetting this is the single most common silent-truncation bug.
- **The session's identity map dedupes by ID per session** — `LoadAsync<T>("orders/1")` twice returns the same instance. This is faithful to DDD but surprising if you assume "each query returns fresh state."
- **Static index changes are side-by-side by default** — RavenDB builds the new version while the old serves queries. Switching takes effect when the new build catches up. For huge collections, this can take hours; plan for it.
- **`SaveChangesAsync` writes are atomic per session, not per database** — multi-doc transactions across sessions need `Cluster Wide Transactions`, which double-write to consensus and are noticeably slower. Prefer one-doc-per-aggregate design.
- **The auto-index merger can produce indexes wider than necessary** — combining "by customer" and "by status" into "by customer + status" means the customer-only query now reads more entries than it needs. Static indexes for known shapes avoid this.
- **`DocumentStore` topology cache uses gossip-based updates** — after a node failover, in-flight requests retry against the new leader automatically, but the first call after failover may take 1–2 seconds. Not a leak; not a bug.
- **Indexing under load uses CPU based on `IndexingPriority`** — `Low` means "wait for the writers to be idle." For a constantly-written DB, low-priority indexes never catch up. Default `Normal` is fine; `High` for indexes that back hot reads.
- **`RavenQueryStatistics` exposes `DurationInMs`, `TotalResults`, `IsStale`** — log them to spot slowdowns and staleness early. Add `.Statistics(out var stats)` on hot queries.
- **`WhereLucene` and `WhereStartsWith` use Lucene query syntax server-side** — `WhereStartsWith("name", "joh")` matches `"john"` and `"johnson"`; case-folding is index-time. Use `WhereEquals` for exact match.
- **`PatchByQueryOperation` runs in the indexer thread pool** — too many concurrent patches starve the indexer and queries become stale. Throttle long-running patches via `MaxOpsPerSec` in the patch options.
- **The `Include` clause is shipped with the query, fetched in the same response** — it's a single round-trip that returns parent + referenced docs together. Cost is in result-set width, not round-trip count.
- **`session.Advanced.NoCaching = true`** disables the per-session cache — useful when you're polling for changes. Default caching can return stale instances within a session if a background process modified the doc.
- **`ConfigureAwait(false)` is unnecessary inside RavenDB's async APIs** — they don't capture sync context. But your *consumer* code should still consider it for library code paths.
- **Bulk insert + change tracking don't mix** — a `Store` then a `BulkInsert` in the same session is undefined. Bulk goes through a different code path entirely.
- **Sharded clusters expose `RavenQueryStatistics.Shards`** — the number of shards a query touched. A query you expect to be single-shard touching multiple is the sharding equivalent of Cosmos's "cross-partition fan-out." Inspect on hot paths.
- **`store.AggressivelyCache(...)` opts a session into stale-tolerant client-side caching** — useful for never-changing reference data, dangerous if applied broadly. Cached responses are served without a server round-trip until the etag changes, which the server pushes via a notification channel.
- **The Studio's "Performance Hints" tab catches most footguns automatically** — slow queries, large documents, missing indexes, request-count violations. Make checking it part of perf reviews; it's the cheapest perf audit tool RavenDB offers.
