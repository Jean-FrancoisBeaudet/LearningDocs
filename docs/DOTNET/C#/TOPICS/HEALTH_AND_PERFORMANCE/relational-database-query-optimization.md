# Relational database query optimization

_Targets .NET 10 / EF Core 10 / C# 14. See also: [SQL index analysis](./sql-index-analysis.md), [SQL query plan analysis](./sql-query-plan-analysis.md), [Pagination strategies](./pagination-strategies.md), [Aggregation efficiency](./aggregation-efficiency.md), [Bulk operations](./bulk-operations.md), [Connection pooling](./connection-pooling.md), [LINQ performance misuse](./linq-performance-misuse.md)._

A relational query is fast or slow for one of five reasons: too many round-trips, the wrong execution plan, missing indexes, transferring rows you don't need, or the JIT/parse cost of building the SQL itself. .NET — through EF Core, Dapper, or raw ADO.NET — gives you levers for each. Most "the database is slow" tickets in a typical service are really one of these five, and the fix lives in C#, not in the database.

> If your trace shows the query running in 4 ms server-side and the request takes 400, the database isn't the problem. Stop tuning indexes and look at the round-trips and the result-set width.

This page is the umbrella for relational perf. Specific subjects have their own notes — index design ([sql-index-analysis](./sql-index-analysis.md)), reading plans ([sql-query-plan-analysis](./sql-query-plan-analysis.md)), pagination, aggregation, bulk writes — link in.

## The cost model

Every query costs you, in order:

1. **C# composition** — building the `IQueryable`, lambda compilation, expression-tree walking. Negligible per call but accumulates with `EF.CompileAsyncQuery` reuse vs cold-built queries.
2. **TDS / wire transmission** — connection acquisition (see [connection pooling](./connection-pooling.md)), command preparation, parameter binding.
3. **Server-side parse + bind + plan** — usually cached after first execution; missing the cache costs single-digit ms.
4. **Server-side execution** — logical reads, physical I/O, CPU on joins/sorts/aggregates. The cost the DBA cares about; usually the smallest part of total latency for OLTP.
5. **Wire transmission back** — N rows × M columns × encoding. The slowest step for wide result sets, and the easiest to cut with projection.
6. **Client materialization** — for EF, hydrating entities and tracking them. For Dapper / ADO.NET, just reading the `DbDataReader`.

Step 5 (rows back) and step 6 (materialization) dominate p95 latency far more often than step 4 (the actual query). Round-trip count (step 2 × N) dominates p99.

## EF Core translation traps

The translation contract: a LINQ tree is server-side iff every step composes to SQL. Where it can't, EF Core 3+ throws (no silent client evaluation). The traps below all *do* translate — they just translate to bad SQL.

### N+1 (the cardinal sin)

```csharp
var orders = await db.Orders.Where(o => o.CreatedAt > cutoff).ToListAsync(ct);
foreach (var o in orders)
    o.Items = await db.OrderItems.Where(i => i.OrderId == o.Id).ToListAsync(ct); // N round-trips
```

Fix with explicit `Include` (single query, possible cartesian explosion) or `AsSplitQuery` (one query per nav, joined client-side):

```csharp
var orders = await db.Orders
    .AsSplitQuery()
    .Include(o => o.Items)
    .Where(o => o.CreatedAt > cutoff)
    .ToListAsync(ct);
```

### Cartesian explosion

`Include(o => o.Items).Include(o => o.Tags)` joins both collections in a single query. If an order has 50 items and 50 tags, the result set is 2500 rows × every order column repeated. Server-side cheap; wire transmission catastrophic.

`AsSplitQuery()` flips this to three round-trips that return their natural cardinality. The trade is that now you can't have a transactionally-consistent snapshot across the three reads — for read-only queries that's fine; for read-modify-write, wrap in a serializable transaction or use a single `Include` with care.

### Projection beats `Include`

```csharp
// Pulls every column of Order and OrderItem.
var orders = await db.Orders.Include(o => o.Items).ToListAsync(ct);

// Pulls only what the caller renders.
var orderViews = await db.Orders
    .Select(o => new OrderListItem(o.Id, o.Total, o.Items.Count, o.CreatedAt))
    .ToListAsync(ct);
```

Projection bypasses change-tracking entirely, ships fewer columns, and produces tighter SQL. Default to projections for read paths; use entity loading only when the caller will mutate and save.

### `AsNoTracking` and `AsNoTrackingWithIdentityResolution`

Read-only entity queries should be `AsNoTracking()` — skips the change-tracker insertion, halves the per-row materialization cost, and prevents accidental mutation persistence. For queries that load the same entity multiple times via different navs (e.g. `Include` with shared references), `AsNoTrackingWithIdentityResolution()` keeps the dedupe behavior without tracking — at the cost of an internal dictionary.

```csharp
db.Orders.AsNoTracking()                          // 99% of reads
db.Orders.AsNoTrackingWithIdentityResolution()    // shared refs across includes
```

For the whole context, set `db.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTrackingWithIdentityResolution` once at startup if read-heavy.

### `IQueryable` vs `IEnumerable` boundaries

```csharp
public IEnumerable<Order> ActiveOrders() => db.Orders.Where(o => o.Active);  // returns IEnumerable

// Caller composes:
var page = ActiveOrders().Where(o => o.Total > 100).Take(50).ToList();
//         ^^^^^^^^^^^^^ already IEnumerable — Where/Take run client-side
```

Returning `IEnumerable<T>` from a repository method silently moves filters and `Take` to the client. Return `IQueryable<T>` from layers that compose; only collapse to `IReadOnlyList<T>` at the use site.

## Compiled queries

EF Core's expression-tree → SQL pipeline is reflective and not free. For hot queries — endpoints called thousands of times per second — pre-compile:

```csharp
private static readonly Func<AppDbContext, Guid, CancellationToken, Task<Order?>> GetById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefault(o => o.Id == id));

public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) => GetById(db, id, ct);
```

Wins 10–30% off per-call latency by skipping expression compilation and SQL generation. The trade: parameters must be primitives (no inline lambdas, no `IEnumerable<T>` — use a separate query or a parameter object). Don't bother for queries called less than ~100 times per process lifetime; the JIT cost of the compiled delegate dominates.

## Parameter sniffing & plan reuse

The first execution of a parameterized query teaches the optimizer "what kind of values this gets" — that plan is then cached and reused. If the first call is for a tenant with 10 rows and the next is for a tenant with 10 million, the plan optimized for the first will be terrible for the second. This is **parameter sniffing**. Full diagnostic detail in [sql-query-plan-analysis](./sql-query-plan-analysis.md); from .NET-side, the levers are:

- **Use parameters** — never string-concat values. EF / Dapper / SqlClient all parameterize by default; the only way to break this is `FromSqlRaw($"... {value}")` (don't), use `FromSqlInterpolated` instead.
- **Distinct queries for distinct workloads** — if "small tenant" and "large tenant" have radically different shapes, give them different SQL (a tag, a `WHERE` predicate that the optimizer treats as separate plans, or just two methods).
- **Last-resort: `OPTION (RECOMPILE)`** — append via `.TagWith("recompile")` + a Query Store hint, or via raw SQL. Burns the cache benefit for that query; use only when sniffing dominates.

EF Core 7+: `.TagWithCallSite()` injects a comment with the C# file/line into the SQL so DMV / Query Store traces map back to source.

## Raw SQL escape hatches

EF translates ~95% of common shapes well. For the other 5% — window functions, recursive CTEs, vendor-specific syntax, hand-tuned hints — drop down without leaving EF:

```csharp
// Type-safe interpolation: every {value} becomes a SqlParameter, no injection.
var top = await db.Orders
    .FromSqlInterpolated($@"
        SELECT TOP 100 *
        FROM Orders
        WHERE CustomerId = {customerId}
        ORDER BY Total DESC")
    .AsNoTracking()
    .ToListAsync(ct);

// Set-based update without entity materialization.
await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Orders SET Status = {OrderStatus.Cancelled} WHERE CreatedAt < {cutoff}", ct);
```

For queries that don't map to entity types, keyless entity types (`db.Set<OrderReport>()` mapped via `HasNoKey().ToView(...)`) are the EF-native option. For one-off DTOs, Dapper composes well with the same `DbConnection` EF gives you via `db.Database.GetDbConnection()`.

When to drop ADO.NET entirely: when you need streaming with `IAsyncEnumerable<T>` over a `DbDataReader`, when you need fine control over `CommandBehavior.SequentialAccess`, or when the entity model is fighting you.

## Retry & resiliency

Transient errors are not bugs — they're physics. Network blips, Azure SQL throttling, pool starvation, deadlock victims. Wire retries at the DB layer:

```csharp
// EF Core — provider-specific transient classification.
opts.UseSqlServer(connStr, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 3,
    maxRetryDelay: TimeSpan.FromSeconds(5),
    errorNumbersToAdd: null));   // null = use default Azure SQL transient list

// Or with Polly / Microsoft.Extensions.Http.Resilience for fine control:
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new() { ShouldHandle = new PredicateBuilder().Handle<SqlException>(IsTransient) })
    .Build();
```

`EnableRetryOnFailure` does *not* compose with user-managed transactions: a retry replays the entire `BeginTransaction` → ... → `Commit` block, but if you opened the transaction outside the EF execution strategy, the retry breaks. Use `db.Database.CreateExecutionStrategy().ExecuteAsync(...)` to wrap multi-statement transactional units.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `await db.Orders.ToListAsync()` then `.Where(...)` in C# | Pulls the whole table; client-side filter |
| `Include(o => o.Items).Include(o => o.Tags)` without `AsSplitQuery` | Cartesian explosion — N×M rows over the wire |
| Returning `IEnumerable<T>` from repository methods | Composition collapses to client-side |
| `FromSqlRaw($"... {value}")` with string interpolation | SQL injection; use `FromSqlInterpolated` |
| Loading entities to mutate one column | Hydration + change tracking for a single `UPDATE`; use `ExecuteUpdateAsync` |
| `db.Orders.Count() > 0` translated as a count subquery | Use `AnyAsync()` — translates to `EXISTS`, short-circuits |
| Synchronous `.ToList()` on a query inside `async` controller | Blocks a thread on I/O; thread-pool starvation under load |
| `using var ctx = new AppDbContext(...)` per call inside a loop | Rebuilds the model, breaks pooling; inject `IDbContextFactory<T>` |
| Loading an entity by `Single(o => o.Id == id)` | Two-pass scan to verify uniqueness; use `FindAsync` (cache-hit) or `FirstAsync` |
| `EnableRetryOnFailure` plus manual `BeginTransaction` | Retry blows up mid-transaction; wrap in `CreateExecutionStrategy().ExecuteAsync` |
| Logging `EnableSensitiveDataLogging()` in production | Parameter values in logs — PII / secret leak |

## Worked example: a typical "endpoint feels slow" fix

Before:

```csharp
public async Task<IReadOnlyList<OrderDto>> GetRecent(Guid customerId, CancellationToken ct)
{
    var orders = await db.Orders
        .Include(o => o.Items)
        .Include(o => o.ShippingAddress)
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt)
        .ToListAsync(ct);                                   // tracked + cartesian
    return orders.Select(MapToDto).ToList();
}
```

After:

```csharp
public async Task<IReadOnlyList<OrderDto>> GetRecent(Guid customerId, CancellationToken ct) =>
    await db.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt)
        .Take(50)
        .Select(o => new OrderDto(
            o.Id,
            o.Total,
            o.CreatedAt,
            o.Items.Count,                                  // subquery, no item rows shipped
            o.ShippingAddress!.City))                       // single column from join
        .ToListAsync(ct);
```

Same business output. SQL changes from a 2-way join with a cartesian product across `Items` to a single query that returns 50 narrow rows. On a customer with 200 orders × 8 items each, that's ~1600 rows → 50, and roughly ten columns → four. Wall-clock latency typically drops by an order of magnitude.

## Senior-level gotchas

- **`SaveChangesAsync` is one round-trip per *batch*, not per entity** — but it's still a transaction. A batched 1000-row update inside a slow transaction holds locks longer than a streaming approach. For large data movement, see [bulk operations](./bulk-operations.md).
- **EF Core 7+ `ExecuteUpdateAsync` / `ExecuteDeleteAsync` bypass the change tracker** — entities you've already loaded are now stale. Either run these before loading or call `db.ChangeTracker.Clear()` after.
- **`FindAsync` checks the local cache first** — if the entity is tracked, no SQL runs. `FirstAsync(o => o.Id == id)` always queries. For repeated lookups in a single scope, `FindAsync` is the cheap choice.
- **`DbContext` is not thread-safe** — a single context shared across parallel `await Task.WhenAll(...)` operations corrupts the change tracker. Use one context per logical operation; `IDbContextFactory<T>` (or `AddPooledDbContextFactory<T>`) for fan-out.
- **`AsSplitQuery` cannot be combined with `IAsyncEnumerable<T>` streaming** — split queries pre-execute each part. Stream only single-query reads.
- **The default `MAX_BATCH_SIZE` is 42 rows for SQL Server** — calibrated against TDS packet size. Raising it past ~100 rarely helps; the packet fragments anyway.
- **`Include` filters via `Include(o => o.Items.Where(...))`** translate to JOIN with the predicate inline. The trap: the predicate filters the *included rows*, not the parent — `Where` on the outer query still drives parent selection. Mixing the two causes double-filtering bugs.
- **`OwnsOne` / `OwnsMany` translate as joins to the same table** — they're a model-side abstraction, not a perf one. They don't reduce columns shipped; projection still does.
- **Compiled queries hold a reference to the `DbContextOptions`** — across context disposal, they keep working. But if you swap providers (test → prod), you must rebuild them.
- **`ChangeTracker.AutoDetectChangesEnabled = false` plus manual `db.Entry(e).State = ...`** is the right pattern for high-throughput write loops. Re-enable detection or call `DetectChanges` once before `SaveChangesAsync` if you do mixed work.
- **`GROUP BY` with a key that's a complex expression often falls back to client eval in EF 6** — but EF Core 7+ handles most cases. Always read the SQL log; the regression test for "did EF translate this?" is one log line.
- **`Distinct()` after `Include` joins blows up** — it's `DISTINCT` on every column of every joined table. Project first, distinct second.
- **Long-lived contexts grow the change tracker without bound** — a worker that loads 10k entities per minute and never disposes its context will OOM. Pooled contexts reset the tracker on return; non-pooled need explicit `Clear()`.
- **`Database.OpenConnectionAsync()` extends the connection lifetime past the next query** — useful for batching multiple statements over one connection, but you own the dispose. Forget and you've leaked a pool slot.
- **TVPs, table types, and JSON parameters bypass the parameter-sniffing trap by design** — the optimizer sees the row-count estimate from the parameter at runtime. For variadic-cardinality queries (`WHERE Id IN (@ids)`), pass a TVP or `OPENJSON` instead of expanding to N parameters.
- **`SET TRANSACTION ISOLATION LEVEL` set in C# does not survive across pool reuse** unless `sp_reset_connection` is disabled (it isn't, by default). The connection always returns to the pool's default isolation. To force snapshot isolation, set it at the database level (`ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON`) or in every transaction explicitly.
