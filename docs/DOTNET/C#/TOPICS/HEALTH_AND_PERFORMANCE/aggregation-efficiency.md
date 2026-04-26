# Aggregation efficiency

_Targets .NET 10 / EF Core 10 / C# 14. See also: [Pagination strategies](./pagination-strategies.md), [Relational database query optimization](./relational-database-query-optimization.md), [Document database query optimization](./document-database-query-optimization.md), [Bulk operations](./bulk-operations.md), [Streaming patterns](./streaming-patterns.md)._

The unifying rule of aggregate queries: **don't pull rows you only need to count, sum, or fold**. Every aggregate has a server-side form (`COUNT(*)`, `SUM`, `AVG`, `GROUP BY`, window functions) and a client-side form (LINQ-to-Objects after `ToList()`). The performance gap between them is often three orders of magnitude — a million-row aggregate that runs in 30 ms server-side takes 30 seconds client-side, plus a memory spike that triggers GC, plus a result set that may not even fit. This note covers how EF Core translates the common shapes, the traps that force client evaluation, and the in-process patterns when you actually do need to aggregate locally.

## The translation contract

In EF Core, an aggregate is server-side **iff the entire predicate-and-projection chain is translatable to SQL**. The moment a step isn't translatable, EF either throws (default since 3.0) or — for some `Select` shapes — silently materializes and runs the rest in memory. There is no warning; the only signal is the SQL log.

```csharp
// Server-side. Emits SELECT COUNT(*) WHERE ... — single integer comes back.
var n = await db.Orders.CountAsync(o => o.IsActive && o.TotalCents > 0, ct);

// Client-side. Emits SELECT * — pulls every row, then counts in process.
var n = (await db.Orders.ToListAsync(ct)).Count(o => MyCustomPredicate(o));
```

Always read the SQL log before optimizing — the wrong half of this is unfixable in C#.

```csharp
// Program.cs — or per-context in OnConfiguring
optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
optionsBuilder.EnableSensitiveDataLogging();   // dev only
```

Every "Executed DbCommand" entry is a real round trip. One per request is the goal.

## Aggregates that always translate

`Count`, `LongCount`, `Sum`, `Min`, `Max`, `Average`, `Any`, `All` against translatable predicates compile to single-row scalar SQL.

```csharp
var totalCents = await db.Orders.Where(o => o.UserId == userId).SumAsync(o => o.TotalCents, ct);
//   → SELECT SUM([o].[TotalCents]) FROM [Orders] [o] WHERE [o].[UserId] = @userId
```

`Sum` over `int` returns `0` for an empty set in C# but `NULL` in SQL — EF coalesces. `Sum` over `int?` returns `null` when *all rows* are null. This trips up reporting code that assumes "no orders → 0 revenue".

## `GroupBy` — the hardest aggregate to translate

EF Core 7+ translates `GroupBy` followed by an aggregate selector into `GROUP BY` SQL when the grouping key is a column or a tuple of columns and the projection is composed of aggregates over the group:

```csharp
// Translates to GROUP BY [Country], SELECT COUNT(*), SUM(...)
var byCountry = await db.Orders
    .GroupBy(o => o.ShippingCountry)
    .Select(g => new { Country = g.Key, Count = g.Count(), Revenue = g.Sum(o => o.TotalCents) })
    .ToListAsync(ct);
```

What still falls back to client evaluation:

- Grouping over a *result of a non-translatable expression* — e.g. `GroupBy(o => MyHelper(o.Status))`.
- Yielding the *groups themselves* (`g.ToList()` inside the projection) — EF would have to ship per-group row bags, which it cannot do efficiently.
- Mixing aggregates with non-aggregate projections that require row-level data.

When EF can't translate, EF Core 3+ throws `InvalidOperationException` rather than silently degrading — a deliberate behavior change after years of users hitting client-eval pitfalls. Configure with `optionsBuilder.ConfigureWarnings(w => w.Throw(RelationalEventId.NonQueryExecuted))` to fail loudly on near-misses.

## Subquery aggregates vs joins

Two SQL shapes for the same logical question — "for each customer, give me their order count":

```sql
-- Correlated subquery: one aggregate per outer row.
SELECT c.Id, (SELECT COUNT(*) FROM Orders o WHERE o.CustomerId = c.Id) AS OrderCount
FROM Customers c;

-- Join + GROUP BY: one scan of Orders, hash join to Customers.
SELECT c.Id, COUNT(o.Id) AS OrderCount
FROM Customers c LEFT JOIN Orders o ON o.CustomerId = c.Id
GROUP BY c.Id;
```

Which wins depends on cardinality and indexes:

- **Subquery**: optimal when filtered to a few outer rows (`WHERE c.Id IN (...)`) and `Orders.CustomerId` is indexed. The optimizer rewrites it to a stream-aggregate with a seek.
- **Join + GROUP BY**: optimal at full-table scale when most customers have orders.

EF's translation prefers correlated subqueries; if the plan is bad, project explicitly:

```csharp
var customerOrderCounts = await db.Customers
    .Select(c => new { c.Id, OrderCount = c.Orders.Count() })   // correlated subquery
    .ToListAsync(ct);
```

vs. a hand-shaped join:

```csharp
var customerOrderCounts = await db.Customers
    .GroupJoin(db.Orders, c => c.Id, o => o.CustomerId, (c, os) => new { c.Id, OrderCount = os.Count() })
    .ToListAsync(ct);
```

Always inspect the plan (`EXPLAIN`/`SET STATISTICS IO ON`) for queries that matter.

## Window functions

`SUM() OVER (...)`, `ROW_NUMBER()`, `RANK()`, `LAG`/`LEAD`. EF Core has limited direct LINQ support; the practical answer is `EF.Functions` for provider-specific window functions or raw SQL via `FromSqlInterpolated`.

```csharp
// Postgres example — running total with Npgsql.EntityFrameworkCore.PostgreSQL.
var rows = await db.Orders
    .Select(o => new
    {
        o.Id,
        o.TotalCents,
        RunningTotal = EF.Functions.Sum(o.TotalCents).Over(/* partition / order */),
    })
    .ToListAsync(ct);
```

For broad portability, accept that complex window queries belong in views or stored procedures and project to a keyless DTO:

```csharp
var rows = await db.Database
    .SqlQuery<OrderWithRunningTotal>($@"
        SELECT Id, TotalCents,
               SUM(TotalCents) OVER (PARTITION BY UserId ORDER BY CreatedAt) AS RunningTotal
        FROM Orders
        WHERE CreatedAt >= {since}")
    .ToListAsync(ct);
```

`SqlQuery<T>` (EF Core 8+) parameterizes safely — interpolated arguments become `DbParameter`s.

## Pagination + total count

The classic two-query trap:

```csharp
var total = await db.Orders.Where(filter).CountAsync(ct);                         // round trip 1
var page  = await db.Orders.Where(filter).OrderBy(o => o.Id)
                            .Skip(skip).Take(take).ToListAsync(ct);               // round trip 2
```

Two queries, two index seeks/scans. Often one window-function query is cheaper:

```csharp
var page = await db.Database.SqlQuery<OrderPage>($@"
    SELECT Id, TotalCents, COUNT(*) OVER () AS TotalCount
    FROM Orders WHERE UserId = {userId}
    ORDER BY Id OFFSET {skip} ROWS FETCH NEXT {take} ROWS ONLY")
    .ToListAsync(ct);
var total = page.FirstOrDefault()?.TotalCount ?? 0;
```

For deep pagination, switch to keyset (cursor) pagination — `OFFSET 100000` re-scans 100K rows even on an indexed sort. See [pagination-strategies.md](./pagination-strategies.md).

## Streaming aggregates without materialization

When the result set is too large to fit but the aggregate is a fold (`Sum`, `Count`, `Max`, histogram), stream and accumulate:

```csharp
long sum = 0;
await foreach (var o in db.Orders.AsAsyncEnumerable().WithCancellation(ct))
    sum += o.TotalCents;
```

Bounded memory regardless of row count. Note: each row still allocates an entity (or DTO). For tens of millions of rows, prefer a server-side `SUM` — streaming exists for cases where the per-row work is non-trivial and not expressible in SQL.

## In-process aggregation patterns

When the data is already in memory (file load, message batch, cache rebuild), the LINQ-to-Objects shapes have predictable allocation profiles:

```csharp
// LINQ — clear, allocates per group + per IGrouping enumerator.
var byKey = items.GroupBy(x => x.Key)
                 .ToDictionary(g => g.Key, g => g.Sum(x => x.Value));

// Manual — half the allocations, faster on hot paths.
var byKey = new Dictionary<TKey, long>();
foreach (var x in items)
    byKey[x.Key] = byKey.TryGetValue(x.Key, out var v) ? v + x.Value : x.Value;
```

`CollectionsMarshal.GetValueRefOrAddDefault` (.NET 6+) drops one hash lookup:

```csharp
var byKey = new Dictionary<TKey, long>();
foreach (var x in items)
{
    ref var slot = ref CollectionsMarshal.GetValueRefOrAddDefault(byKey, x.Key, out _);
    slot += x.Value;
}
```

`AggregateBy` / `CountBy` (.NET 9+) wrap this pattern with single-pass semantics:

```csharp
var byKey = items.AggregateBy(x => x.Key, seed: 0L, (acc, x) => acc + x.Value);
```

For numeric reductions over arrays, `TensorPrimitives.Sum` (.NET 8+, `System.Numerics.Tensors` package) and `Vector<T>` SIMD beat scalar loops 4–8× on modern hardware.

## `decimal` vs `double` aggregation

| Type | Throughput (per element) | Precision | When |
|---|---|---|---|
| `double` | ~1× (SIMD-friendly) | ~15-17 significant digits, drift on long sums | Scientific, statistical, anything tolerating ulp-level error |
| `decimal` | ~10–20× slower | exact (128-bit, base-10) | Money. Always money. |

`decimal` aggregation in SQL respects the column precision/scale. EF's `decimal(18,2)` summed over a billion rows can overflow silently — define columns wide enough or split the aggregate by partition. Postgres's `numeric` is unbounded by default; SQL Server's `DECIMAL` requires explicit precision.

## Document / NoSQL aggregates

| Store | Shape | Cost driver |
|---|---|---|
| RavenDB | Map-reduce indexes (computed at write time) | Index storage and rebuild on schema change |
| MongoDB | `$group` pipeline | Document scan unless covered by index; aggregation pipeline order matters |
| Cosmos DB | Cross-partition aggregate | RU charge — request unit cost grows linearly with scanned partitions |

The defining principle differs from SQL: aggregates are **typically pre-computed** on write rather than computed on read. RavenDB's map-reduce is the canonical example — you define the aggregate shape, the engine maintains it. The trade is consistency model and write amplification rather than read CPU.

For Cosmos DB: cross-partition `COUNT(*)` fans out to every partition. When metrics matter and partition count is large, maintain a counter document or a materialized-view container — the read cost of a real-time aggregate over 50 partitions is the RU sum of all 50.

See [document-database-query-optimization.md](./document-database-query-optimization.md) and [ravendb-performance.md](./ravendb-performance.md) for store-specific patterns.

## Tooling

| Tool | What it shows |
|---|---|
| EF Core logging at `Information` | Every executed SQL — count round trips |
| `EF.CompileAsyncQuery` | One-time translation cost amortized across calls |
| SQL Server Query Store / Postgres `pg_stat_statements` | Hot queries, plan changes over time |
| `EXPLAIN (ANALYZE, BUFFERS)` (Postgres) / `SET STATISTICS IO, TIME ON` (SQL Server) | Real plan and I/O |
| MiniProfiler / OpenTelemetry SQL instrumentation | Per-request query count, p99 query latency |
| `dotnet-counters` `time-in-gc`, `alloc-rate` | Symptoms of client-side aggregation pulling rows |

## Senior-level gotchas

- **`Count()` round trips silently.** A `Count()` inside a `foreach` body re-queries every iteration. Materialize the count once before the loop.
- **`GroupBy` followed by `Count()` per group must be one query.** Verify in the SQL log. If EF emits one query per group, you've hit a translation gap — refactor the projection.
- **`Sum` over `int`/`long` can overflow.** SQL `SUM(int)` returns `int` in some providers and `bigint` in others; EF maps the result type to the column type. For aggregating order totals, project `(long)o.TotalCents` to force `bigint` arithmetic.
- **`Sum` over a nullable column is `NULL` for an empty set in SQL but `0` in C#.** Explicit coalesce: `Sum(o => o.TotalCents ?? 0)` or `?? 0L` after the await — depends on which side you want to handle the empty-set case.
- **Pagination with `Count()` produces two index scans even with the right index.** A `COUNT(*) OVER ()` window function in the page query usually beats two queries; benchmark with realistic data.
- **`ToList()` before `Where` materializes the table.** A code review smell. Always check that filter operators precede materialization.
- **`AsNoTracking()` doesn't help aggregates** — there's nothing to track on a scalar `SUM`. The advice applies to entity-projection reads, not aggregates.
- **`decimal` SUM overflow is a real production issue** — `SUM(decimal(18,2))` of 100M rows averaging $1,000 = $100B = 12 digits before the decimal, leaving 6 of headroom. Plan column width per the long-tail of values, not the average.
- **Histograms over `string` keys allocate the key** unless interned. For known small key spaces (status codes, country codes), `string.Intern` (or a `FrozenDictionary<string, T>` lookup at construction) cuts dictionary allocations significantly.
- **`GroupBy` with hundreds of millions of distinct keys runs the database out of work memory** — server-side hash aggregate spills to disk. Sometimes a `DISTINCT ... ORDER BY` + streaming fold is cheaper than the natural `GROUP BY`.
- **Aggregates over partitioned tables in Postgres need `partitionwise_aggregate=on`** to plan the per-partition reduction. Same idea on SQL Server with partitioned views — the optimizer doesn't always pick the parallel plan.
- **Cross-partition Cosmos aggregates are billed per partition scanned, not per row returned.** A `SELECT COUNT(*)` across 50 partitions is 50× the RU of a partitioned single-key query. For dashboards, maintain a counter or materialized view; for one-offs, accept the cost.
- **`IAsyncEnumerable<T>.SumAsync`** doesn't exist on `IAsyncEnumerable<T>` directly — it's an EF Core extension. For LINQ-to-Objects async streams, use `System.Linq.Async` or fold manually with `await foreach`.
- **`Average` over an empty `IQueryable<int>`** throws `InvalidOperationException`, while `Average` over an `IQueryable<int?>` returns `null`. Match the column nullability to the expected empty-set behavior.
