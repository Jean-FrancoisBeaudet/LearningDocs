# Pagination and Grouping

_Targets .NET 10 / C# 14. See also: [IEnumerable vs IQueryable](./ienumerable-vs-iqueryable.md), [Filtering, Projections, Sorting](./filtering-projections-sorting.md), [Aggregate functions](./aggregate-functions.md), [Array, List, Dictionary, HashSet, Stack, Queue](../COLLECTIONS/array-list-dictionary-hashset-stack-queue.md)._

`Skip`/`Take` and `GroupBy` look trivial in tutorials and quietly cause more production incidents than any other LINQ operators. Senior view: when offset pagination is wrong, why `GroupBy` materializes its input even though it claims to be deferred, and how EF Core changed the rules in 6.0.

## Offset pagination — `Skip` + `Take`

```csharp
const int PageSize = 50;
var page = await db.Orders
    .OrderBy(o => o.CreatedAt).ThenBy(o => o.Id)   // unique tiebreaker mandatory
    .Skip(pageNumber * PageSize)
    .Take(PageSize)
    .Select(o => new OrderListDto(o.Id, o.Total, o.CreatedAt))
    .ToListAsync(ct);
```

Two non-negotiable rules:
1. **`OrderBy` is required** — without it, the database is free to return rows in any order, and `Skip` becomes nondeterministic. EF Core warns at translation; SQL Server's `OFFSET ... FETCH` requires an `ORDER BY` clause and the query won't even parse without one.
2. **The order key must be unique** — if 100 rows share `CreatedAt`, page 2 may overlap or skip rows from page 1 because the sort within equal keys is undefined. Append a unique tiebreaker (`ThenBy(o => o.Id)`).

### The deep-page cliff

`OFFSET 100000 FETCH 50` makes the database read and discard 100,000 rows before returning 50. As `pageNumber` grows, query time grows linearly. For "show me page 2,000" UIs, the database is doing 100× more work than the user sees. Page-load times degrade silently until someone opens an old report.

### Keyset (cursor) pagination

The senior alternative — page on the value of the last seen sort key, not on a row number:

```csharp
// Caller sends back the (CreatedAt, Id) of the last row from the previous page.
var page = await db.Orders
    .Where(o => o.CreatedAt > lastSeenCreatedAt
             || (o.CreatedAt == lastSeenCreatedAt && o.Id > lastSeenId))
    .OrderBy(o => o.CreatedAt).ThenBy(o => o.Id)
    .Take(PageSize)
    .ToListAsync(ct);
```

The `WHERE` clause uses the index, no `OFFSET` scan, page-N latency is constant. Trade-off: you can't jump to "page 50" — only forward/backward from a cursor. For infinite-scroll feeds, that's exactly what you want. For "page X of Y" UIs with random access, offset is the only choice and you cap `pageNumber`.

## `Chunk` (.NET 6+) — slice an in-memory sequence

```csharp
foreach (T[] batch in items.Chunk(100))
    await ProcessBatch(batch, ct);
```

Returns `IEnumerable<T[]>` — the last chunk may be smaller than `size`. In-memory only; not translated by EF Core. Useful for:
- Bulk-insert batching (`SqlBulkCopy`, EF Core `ExecuteUpdate`).
- Throttled API calls — chunk + `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`.

## `GroupBy` — the deferred-but-materializing operator

```csharp
public interface IGrouping<out TKey, out TElement> : IEnumerable<TElement>
{
    TKey Key { get; }
}

var byCustomer = orders.GroupBy(o => o.CustomerId);
```

`GroupBy` is deferred (returns `IEnumerable<IGrouping<TKey, T>>`), but the moment you start iterating, it **fully materializes the source** into an internal lookup before yielding the first group. There's no streaming way to compute groups — you'd have to know all elements before deciding which group is "complete". So `GroupBy` over a 10M-row source allocates a 10M-element internal structure, regardless of how few groups you peek at.

If you only need per-key aggregates and don't need the elements themselves, **don't use `GroupBy`**:

```csharp
// Bad — materializes every order, builds groups, then aggregates.
var totals = orders.GroupBy(o => o.CustomerId)
                   .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Amount) });

// Good — single pass, one slot per key. (.NET 9+)
var totals2 = orders.AggregateBy(
    keySelector: o => o.CustomerId,
    seed:        0m,
    func:        (sum, o) => sum + o.Amount);
```

For .NET 8 and earlier, hand-write the loop with a `Dictionary<TKey, TAccumulator>`.

### Composite-group keys

```csharp
// Anonymous type — value equality of the projected fields, in-memory friendly.
var byMonth = orders.GroupBy(o => new { o.CustomerId, o.CreatedAt.Year, o.CreatedAt.Month });

// Tuple — same equality semantics, sometimes more readable in EF Core 7+.
var byMonth2 = orders.GroupBy(o => (o.CustomerId, o.CreatedAt.Year, o.CreatedAt.Month));
```

Both work in LINQ-to-Objects. EF Core translates both to `GROUP BY` over the underlying columns. Anonymous types translated more reliably in older EF Core versions; if you're stuck on EF Core 5 or below, prefer them.

### `ToLookup` vs `GroupBy`

```csharp
ILookup<int, Order> lookup = orders.ToLookup(o => o.CustomerId);
IEnumerable<Order> mine = lookup[customerId];   // empty if missing — never throws
```

`ToLookup` is **eager** — materializes immediately and returns `ILookup<TKey, TElement>`, which is a multi-map you can index by key. `GroupBy` is deferred. Use `ToLookup` when you'll do many lookups; use `GroupBy` when you'll iterate the groups once.

`ILookup<TKey, TElement>` returns an empty sequence for missing keys — no `KeyNotFoundException`, no need for `TryGetValue`. That's the senior reason to reach for it over `Dictionary<TKey, List<T>>`.

## EF Core `GroupBy` — the rewrite

Up to EF Core 2.x, many `GroupBy` queries silently fell back to client evaluation. EF Core 3.0 switched to "translate or throw". EF Core 6.0+ added wide support for grouping translation, but with hard constraints:

- The `Select` projection over groups must contain only **aggregations** (`Count`, `Sum`, `Min`, `Max`, `Average`, `Key`). Selecting `g.First()` or `g.ToList()` will fall back/throw.
- Grouping by a navigation property generally needs to be expressed as a join in the query plan; EF Core does it for you when it can.

```csharp
// Translates to GROUP BY CustomerId in SQL.
var totals = await db.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Total), Count = g.Count() })
    .ToListAsync(ct);

// Does NOT translate cleanly — needs the rows themselves grouped.
// Restructure: load orders ordered by CustomerId, then group in memory.
var withItems = await db.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Orders = g.ToList() })
    .ToListAsync(ct);  // ⚠ may throw or be slow depending on EF version
```

When you need both the rollup and the source rows, the canonical EF Core 6+ pattern is to fetch the rows once with a projection and group client-side:

```csharp
var rows = await db.Orders
    .Where(o => o.CreatedAt >= since)
    .Select(o => new { o.CustomerId, o.Id, o.Total })
    .ToListAsync(ct);

var grouped = rows.GroupBy(r => r.CustomerId)
                  .Select(g => (CustomerId: g.Key, Items: g.ToList()))
                  .ToList();
```

## Streaming pagination with `IAsyncEnumerable<T>`

For exports, ETL, or "all rows" jobs, page through the result set without materializing it whole:

```csharp
await foreach (var order in db.Orders
    .Where(o => o.CreatedAt >= since)
    .OrderBy(o => o.Id)
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await sink.WriteAsync(order, ct);
}
```

EF Core streams rows as the `DbDataReader` advances; memory pressure stays flat instead of growing with row count. The trade-off is that a single long query holds a connection open — not OK for multi-hour scans. For those, keyset-paginate in `Chunk`-sized batches and release the connection between batches.

**Senior-level gotchas:**
- `Skip(n).Take(m)` over a non-ordered source is not a bug at compile time but **is** a bug at runtime. EF Core warns; LINQ-to-Objects silently uses source order, which for a `HashSet<T>` or `Dictionary<TKey, TValue>` is undefined.
- For UIs that show "page X of Y", the `Y` requires a separate `COUNT(*)` round trip — and that count is stale the moment it returns. Consider "next/prev" cursor UIs that don't need a total.
- `GroupBy` over an `IQueryable` that returns groups is only translated when followed by an aggregating projection. `db.Orders.GroupBy(o => o.CustomerId).ToListAsync()` (no `Select`) will throw on most providers.
- `IGrouping<TKey, T>` is `IEnumerable<T>` — re-iterating the group re-enumerates the materialized lookup's slice, but it does **not** re-fetch from the source. Safe to iterate twice; the second pass is cheap.
- `Chunk(n)` returns the last partial chunk; downstream code must handle `batch.Length < n`. Don't assume every batch is full when computing offsets.
- `Take(0)` is legal and returns an empty sequence — does **not** throw. `Skip(int.MaxValue)` is also legal and returns empty. Negative arguments are clamped to zero. So invalid input quietly produces empty results — validate at the API boundary.
- `OrderBy` followed by `Where` in EF Core: the SQL still applies `WHERE` first (the optimizer reorders), but in LINQ-to-Objects the order matters because `OrderBy` materializes a buffer. Prefer `Where` first for clarity and parity.
- `GroupBy` retains element order *within* a group (LINQ-to-Objects). Don't rely on the order *between* groups — it's the order in which keys are first seen.
- `Take(^n..)` and range overloads (`.Take(Range)`, .NET 6+) let you pluck the last N or a slice without `Skip(...).Take(...)` — `items.Take(^10..)` returns the last 10. In-memory only.
- For paginated APIs, return the **cursor** (last `(CreatedAt, Id)`) to the client, not a page number. Clients that retain stale cursors degrade gracefully (just resume from a known row); clients with stale page numbers re-fetch overlapping data on every call.
