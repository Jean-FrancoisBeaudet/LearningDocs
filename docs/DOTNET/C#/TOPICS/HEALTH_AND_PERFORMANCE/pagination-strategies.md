# Pagination strategies

_Targets .NET 10 / C# 14. See also: [Relational database query optimization](./relational-database-query-optimization.md), [SQL index analysis](./sql-index-analysis.md), [SQL query plan analysis](./sql-query-plan-analysis.md), [Streaming patterns](./streaming-patterns.md), [Document database query optimization](./document-database-query-optimization.md), [Aggregation efficiency](./aggregation-efficiency.md), [Bulk operations](./bulk-operations.md)._

**Pagination** is how a service returns a slice of an ordered result set without loading the rest. The two common implementations have radically different cost curves: offset/limit is `O(N)` in the number of rows skipped, keyset (cursor) is `O(log N)` regardless of how deep into the set you are. The decision is structural, not stylistic — at any non-trivial size, offset breaks under its own weight.

> If the answer to "what page is the user on?" is `OFFSET 1_000_000`, you've already lost. The database is reading and discarding a million rows on every request, and the user got the same answer five seconds slower.

This page is also the cleanest source for keyset/cursor pagination as a streaming primitive — see [streaming-patterns.md](./streaming-patterns.md) for how to compose them.

## The three families

| Strategy | Cost (page N) | Stable under writes? | Best for |
|---|---|---|---|
| Offset / limit (`OFFSET N FETCH M`) | `O(N + M)` — reads & discards N rows | No (rows shift on insert/delete) | Small datasets (<1k rows total), admin pages |
| Keyset (cursor) | `O(log N + M)` — index seek to last position | Yes (cursor is a value, not a position) | Anything user-visible at scale |
| Token-based (signed keyset) | Same as keyset | Yes | Public APIs where tampering matters |

There is no fourth strategy worth considering. "Page numbers in the URL" is just offset/limit with extra steps.

## Offset/limit: when it's actually OK

```csharp
var pageItems = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderBy(o => o.CreatedAt).ThenBy(o => o.Id)
    .Skip((page - 1) * size)
    .Take(size)
    .ToListAsync(ct);
```

Translates to `OFFSET (N) ROWS FETCH NEXT (M) ROWS ONLY`. Acceptable when:

- The total result set fits on a few pages (< ~1000 rows).
- The data is mostly static for the duration of a session (admin dashboards, reports).
- You actually need random page access (page jumper UI: 1, 2, 3, … 50). Most "infinite scroll" UIs don't.

The cost: the engine must read all rows up to OFFSET and discard them. At page 10000 with size 50, that's 500050 rows read to return 50. Add an `ORDER BY` with no covering index and you get a sort on top of the scan.

## Keyset / cursor pagination

```sql
-- First page
SELECT TOP 50 *
FROM Orders WHERE CustomerId = @c
ORDER BY CreatedAt DESC, Id DESC;

-- Next page — cursor = (last row's CreatedAt, last row's Id)
SELECT TOP 50 *
FROM Orders
WHERE CustomerId = @c
  AND (CreatedAt, Id) < (@lastCreatedAt, @lastId)   -- row-value comparison
ORDER BY CreatedAt DESC, Id DESC;
```

The `(CreatedAt, Id) < (...)` is **lexicographic** comparison: same `CreatedAt` rows are tie-broken by `Id`. This is the trick — without `Id` (or another unique column) as the tie-breaker, two rows with the same `CreatedAt` end up on different pages or the same page, depending on how the planner orders them.

Index requirement: `(CustomerId, CreatedAt DESC, Id DESC) INCLUDE (...)` — a covering index on the predicate + sort columns. The engine seeks to `(@lastCreatedAt, @lastId)` and reads forward. `O(log N + M)`.

EF Core 7+ supports row-value comparison in LINQ:

```csharp
public async Task<IReadOnlyList<Order>> GetPageAsync(
    Guid customerId, DateTime? cursorCreatedAt, Guid? cursorId, int size, CancellationToken ct)
{
    IQueryable<Order> q = db.Orders
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt).ThenByDescending(o => o.Id);

    if (cursorCreatedAt is { } t && cursorId is { } id)
        q = q.Where(o => o.CreatedAt < t || (o.CreatedAt == t && o.Id.CompareTo(id) < 0));

    return await q.Take(size).AsNoTracking().ToListAsync(ct);
}
```

(`AsNoTracking` because pagination is read-only — there's no point paying for the change tracker.)

## Signed-token cursors for public APIs

A naked `?lastId=42&lastCreatedAt=2026-01-01` works but exposes internal schema and lets clients fabricate cursors. Wrap it in an opaque, signed token:

```csharp
public sealed record Cursor(DateTime CreatedAt, Guid Id);

public sealed class CursorCodec(IDataProtectionProvider dpp)
{
    private readonly IDataProtector _p = dpp.CreateProtector("PaginationCursor.v1");

    public string Encode(Cursor c) =>
        _p.Protect(JsonSerializer.SerializeToUtf8Bytes(c)).ToBase64Url();

    public Cursor? Decode(string? token) =>
        token is null ? null
        : JsonSerializer.Deserialize<Cursor>(_p.Unprotect(Convert.FromBase64String(token)));
}

// API surface
app.MapGet("/orders", async (
    Guid customerId, string? cursor, int? limit,
    CursorCodec codec, IOrderRepository repo, CancellationToken ct) =>
{
    var size = Math.Clamp(limit ?? 50, 1, 200);   // bound it
    var c = codec.Decode(cursor);
    var page = await repo.GetPageAsync(customerId, c?.CreatedAt, c?.Id, size + 1, ct);
    var hasMore = page.Count > size;
    var items = hasMore ? page[..size] : page;
    var next = hasMore
        ? codec.Encode(new Cursor(items[^1].CreatedAt, items[^1].Id))
        : null;
    return Results.Ok(new { items, nextCursor = next });
});
```

The `+1` trick: ask for `size + 1` rows; if you got `size + 1`, there's another page (and you drop the last row before returning). Avoids a `COUNT(*)`.

`IDataProtector` (built into ASP.NET Core) provides authenticated encryption — the client can't tamper, can't read, can't replay across cookie scopes. Versioning the protector purpose (`v1`) lets you rotate the schema later.

## Counts: usually wrong to compute

`SELECT COUNT(*)` over the same predicate is the most-overused query in pagination. For a public list endpoint, you almost certainly don't need it.

| Need | Approach |
|---|---|
| "Show 'next' if there are more rows" | Fetch `size + 1`, check overflow (above) |
| "Page 1 of 50" UI element | Reconsider — keyset can't compute total page count cheaply, and offset can't be fast |
| Approximate total ("about 12,500") | `sys.dm_db_partition_stats` (SQL Server), `pg_class.reltuples` (Postgres) — instant, ~accurate |
| Exact filtered count | `SELECT COUNT(*) WHERE …` with a covering index; budget for it as its own query |

If the UI is "infinite scroll" or "load more," you don't need a count. If it's a true pager (1, 2, 3, …, 50), you've chosen offset/limit and accepted the cost.

## EF Core specifics

- `Skip(n).Take(m)` translates to OFFSET/FETCH on SQL Server / PostgreSQL, LIMIT/OFFSET on MySQL. Always pair with `OrderBy` — without it, the result is engine-defined and may differ between runs of the same query.
- `OrderBy` must include a unique tie-breaker. `OrderBy(o => o.CreatedAt)` alone is non-deterministic when timestamps tie; `.ThenBy(o => o.Id)` makes it stable.
- For keyset: row-value comparison `WHERE (a, b) < (@a, @b)` is supported in EF Core 7+; for older versions, expand to `WHERE a < @a OR (a == @a && b < @b)` manually.
- `AsAsyncEnumerable()` over a keyset query is a streaming pager — no buffering of pages in memory:

```csharp
public async IAsyncEnumerable<Order> StreamAsync(
    Guid customerId, [EnumeratorCancellation] CancellationToken ct = default)
{
    DateTime? cAt = null; Guid? cId = null;
    while (true)
    {
        var page = await GetPageAsync(customerId, cAt, cId, 1000, ct);
        if (page.Count == 0) yield break;
        foreach (var o in page) yield return o;
        cAt = page[^1].CreatedAt; cId = page[^1].Id;
    }
}
```

## NoSQL pagination

| Store | Mechanism |
|---|---|
| MongoDB | `Skip + Limit` is offset (slow at depth); use `_id` cursor: `{ _id: { $gt: lastId } }.sort({ _id: 1 }).limit(N)` |
| Cosmos DB | Continuation tokens — opaque, server-generated, single-use across the same query |
| RavenDB | `Skip + Take` with `Statistics.TotalResults`; for huge sets, streaming queries |
| DynamoDB | `LastEvaluatedKey` continuation token; required, not optional |
| Elasticsearch | `search_after` (keyset over sort values), `point_in_time` for stable iteration |

All of them are keyset-shaped underneath; only Mongo's `Skip` lets you shoot yourself in the foot the same way SQL does.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `Skip(1_000_000).Take(50)` | Reads 1M rows, returns 50. Linear cost in offset. |
| `OFFSET` without `ORDER BY` | Engine-defined row order; "page 2" overlaps "page 1." |
| `OrderBy(o => o.CreatedAt)` (no tie-breaker) | Two rows at same timestamp drift between pages. |
| `?lastId=42` exposed to clients without signing | Clients fabricate cursors, scrape ranges, or break on schema change. |
| Unbounded `?limit=` | DoS vector — one client asks for 10M rows. Always clamp. |
| `COUNT(*)` per page | Doubles the cost; usually unnecessary. |
| Cursor over a non-unique sort key | Same row appears on two pages, or one row skipped between pages. |
| Mutable cursor (e.g. `WHERE id > @lastId` over a re-used `id` column) | Deletes/inserts make it ambiguous. Pin to immutable identifier. |
| Returning the cursor as a query parameter the user sees in the URL | Embedded in browser history, link-shared, sent to logs. Body or response header is better. |
| `OFFSET` over a soft-deleted predicate | Deletes change page composition retroactively. |

## Worked example: keyset endpoint with all the pieces

```csharp
// Index (SQL Server):
// CREATE INDEX IX_Orders_Customer_Created ON Orders (CustomerId, CreatedAt DESC, Id DESC)
//    INCLUDE (Total, Status);

public sealed class OrdersController(IOrderRepository repo, CursorCodec codec) : ControllerBase
{
    [HttpGet("orders")]
    public async Task<ActionResult<PageResult<OrderDto>>> List(
        [FromQuery] Guid customerId,
        [FromQuery] string? cursor,
        [FromQuery] int? limit,
        CancellationToken ct)
    {
        if (customerId == default) return BadRequest("customerId required");
        var size = Math.Clamp(limit ?? 50, 1, 200);
        var c = codec.Decode(cursor);

        var rows = await repo.GetPageAsync(customerId, c?.CreatedAt, c?.Id, size + 1, ct);
        var hasMore = rows.Count > size;
        var items = (hasMore ? rows.Take(size) : rows)
            .Select(OrderDto.From).ToList();

        var nextCursor = hasMore
            ? codec.Encode(new Cursor(rows[size - 1].CreatedAt, rows[size - 1].Id))
            : null;

        return Ok(new PageResult<OrderDto>(items, nextCursor));
    }
}

public sealed record PageResult<T>(IReadOnlyList<T> Items, string? NextCursor);
```

Properties:

- O(log N + M) on every page (covering index + seek).
- Stable under inserts/deletes (cursor is a value).
- Tamper-resistant (cursor is signed).
- Bounded `limit` (DoS-safe).
- No `COUNT(*)` (cheap).
- No allocations beyond the page (`Take` on `IList` doesn't materialize the suffix).

## Senior-level gotchas

- **Tie-breakers must be unique.** `(CreatedAt, Id)` works because `Id` is unique. `(LastName, FirstName)` doesn't — two people with identical names break the cursor. Always anchor the cursor to a unique key (the PK is fine).
- **Descending vs. ascending requires matching index direction.** SQL Server can scan an ascending index backwards, but with poorer cache behavior; Postgres can scan both directions but the planner sometimes picks a sort if statistics are stale. If you order DESC, build the index DESC.
- **`NULL`s in the sort key break naive keyset.** `WHERE (a, b) < (@a, @b)` returns no rows when `a` is NULL because NULL comparison is unknown. Either NOT NULL the column, or split the cursor into "non-null then null" buckets explicitly.
- **`OFFSET` semantics differ across engines.** SQL Server's `OFFSET …  ROWS FETCH NEXT … ROWS ONLY` requires `ORDER BY`; MySQL's `LIMIT N OFFSET M` doesn't (but should). Don't assume portability.
- **Cursor expiry matters under schema change.** A token encoding `(CreatedAt, Id)` becomes meaningless if the sort key changes to `(UpdatedAt, Id)`. Version the cursor (`v1.{base64}`) and reject old versions with `400 Bad Request` after a deploy.
- **`+1` over-fetch can return inconsistent pages under concurrent writes.** Between fetching size+1 for page N and the cursor-resume for page N+1, a row at the boundary may be inserted. Two valid options: accept eventual visibility, or use snapshot isolation (RCSI on SQL Server, repeatable read on Postgres) within a single session.
- **`Skip` defeats the index in many planners.** EF Core's `Skip(n).Take(m)` produces `OFFSET (n) FETCH NEXT (m)`, which the planner often resolves with an index scan + filter rather than a seek. Plan-check the SQL — see [sql-query-plan-analysis.md](./sql-query-plan-analysis.md).
- **`COUNT(*)` is a separate query — its cost is not "free because we already scanned."** SQL Server caches buffer pages but not aggregate results. Per-page count = full scan per page if the predicate isn't covering.
- **Deduping across pages requires a stable iteration window.** If a row is updated *during* iteration such that its sort key changes, it may appear on multiple pages or skip pages. Snapshot or accept duplicates and dedupe client-side by ID.
- **Don't sign cursors with HMAC over plaintext.** Use authenticated encryption (`IDataProtector`) — HMAC alone leaks the cursor contents, which often includes user IDs or timestamps you'd rather not expose.
- **`OrderBy` clauses with computed columns can't use the index.** `ORDER BY UPPER(LastName)` ignores any index on `LastName`. If you sort by a transformation, materialize it (computed column + index, or stored case-folded copy).
- **MongoDB `Skip + Sort` without an index is `O(N log N)` per page.** Same trap as SQL but with no plan visibility unless you explicitly explain. For Mongo, `_id` is your friend — it's monotonic enough for most cursor needs.
- **Elasticsearch `from + size` is capped at 10000 by default** (`index.max_result_window`). Past that you must use `search_after`. The cap is a feature — deep offset against a sharded engine is an anti-pattern at any scale.
- **Cosmos DB continuation tokens are query-bound.** A token issued by query A cannot be used with query B even if the predicates look the same. Same partition, same query string, or it errors.
- **`ORDER BY id DESC LIMIT 50` is *not* the same as `ORDER BY id ASC LIMIT 50` reversed.** The first returns the 50 newest; the second returns the 50 oldest then reverses in memory. If your data is append-only, you almost always want DESC.
- **The pager is the test surface for the index strategy.** A "fast at page 1, slow at page 50" complaint is almost always a missing index entry, not a code bug. Run `SET STATISTICS IO ON` on page 1 and page 50 and compare logical reads — they should be within 10% of each other.
