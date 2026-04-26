# SQL query plan analysis

_Targets SQL Server 2022 / PostgreSQL 16 / .NET 10. See also: [SQL index analysis](./sql-index-analysis.md), [Relational database query optimization](./relational-database-query-optimization.md), [Pagination strategies](./pagination-strategies.md), [Aggregation efficiency](./aggregation-efficiency.md), [Connection pooling](./connection-pooling.md)._

A query plan is the optimizer's compiled answer to "how do I execute this SQL." Reading it well is the highest-leverage skill for database performance work — it tells you whether the index you built is being used, whether the join order is sensible, whether estimates match reality, and where the real cost lives. Most "I added an index but the query is still slow" tickets resolve by reading the plan and finding that the optimizer chose to scan, lookup, or sort instead of seeking.

> If you're guessing at perf — "maybe the join is wrong, maybe an index helps" — and you haven't read the plan, you're tuning blind. The plan is *the* primary diagnostic; everything else (DMVs, profiler, Query Store) is corroboration.

This page covers SQL Server primarily, with PostgreSQL `EXPLAIN` notes inline.

## Estimated vs actual

Two flavors of plan, captured differently:

| Plan | What it shows | When to use |
|---|---|---|
| Estimated | Optimizer's compile-time choice + estimated row counts | The query won't finish, won't run safely, or you're comparing alternatives |
| Actual | Estimated plan *plus* actual row counts and CPU/IO per operator | Diagnosing a real slow query — almost always what you want |

Trust *actual* plans for diagnosis. The estimated plan tells you what the optimizer *thought* would happen; the actual plan tells you whether it was right. The gap between them is where most tuning lives.

Get them in SSMS via Ctrl+M (include actual plan), or programmatically:

```sql
SET STATISTICS XML ON;
-- run query
SET STATISTICS XML OFF;
```

The XML is what `dm_exec_query_plan` returns and what the Query Store stores.

PostgreSQL: `EXPLAIN` is the estimated plan; `EXPLAIN (ANALYZE, BUFFERS)` runs the query and reports actual row counts, timing, and buffer access.

## Operator quick reference

The operators that account for >95% of plan reading:

| Operator | What it does | Cost shape | When it's right |
|---|---|---|---|
| Index Seek | B-tree traversal to first matching row | `O(log N)` | Equality / leading-edge range on indexed column |
| Index Scan | Full read of an index | `O(N)` | No predicate matches the index, or stats say scan is cheaper |
| Clustered Index Scan | Full table read (the table *is* the clustered index) | `O(N)` | Almost never right at scale |
| Key Lookup | Random I/O back to clustered index for non-included columns | `O(matched rows × IO)` | Acceptable for small result sets; deadly at scale |
| Nested Loops Join | Outer row → inner index seek per outer | `O(outer × log inner)` | Small outer + indexed inner |
| Hash Join | Build hash of one side, probe with the other | `O(build + probe)` | Large unsorted inputs, equality join |
| Merge Join | Both sides sorted, walk together | `O(N + M)` | Both inputs naturally sorted by join key |
| Sort | In-memory or tempdb sort | `O(N log N)` | No index satisfies the ORDER BY |
| Hash Aggregate | Group via hash | `O(N)` | Many distinct groups |
| Stream Aggregate | Group on already-sorted input | `O(N)` | Input pre-sorted by group key |
| Lazy/Eager Spool | Worktable to materialize a subexpression | Hidden cost in tempdb | Optimizer fallback; usually a smell |
| Gather Streams | Reassembles parallel branches | Network-like cost in CPU | Parallel plan; check `MAXDOP` |

The two cheap winners are **Index Seek** (predicates) and **Stream Aggregate** (group-bys). The two expensive losers are **Sort** (no covering index for ORDER BY) and **Key Lookup at scale** (non-covering nonclustered index).

PostgreSQL operator names map roughly: `Index Scan` (= seek), `Index Only Scan` (= covering seek), `Bitmap Index Scan + Bitmap Heap Scan` (= seek with many matches), `Seq Scan`, `Hash Join`, `Merge Join`, `Nested Loop`, `HashAggregate`, `GroupAggregate`.

## Reading the fat arrow

In SSMS, arrows between operators are sized by *actual* row count. The diagnostic shape is the **fat-arrow problem**: an operator with a thin estimated arrow but a fat actual arrow.

```
Index Seek → Estimated: 10 rows | Actual: 2,500,000 rows
```

The optimizer chose its plan assuming 10 rows; getting 2.5M means downstream operators (probably a Nested Loops join with an index seek per row) execute 2.5M times instead of 10. Plan was right in shape but built on bad numbers.

Causes:

- **Stale statistics.** `UPDATE STATISTICS` on the table; `WITH FULLSCAN` if the column is heavily skewed.
- **Parameter sniffing** (see below) — the plan was compiled for a parameter value with low cardinality, then reused for one with high cardinality.
- **Local variables.** SQL Server can't sniff `DECLARE @x = ...` values; treats them as average density.
- **Table variables.** Cardinality estimate is hardcoded to 1 row pre-2019 (since 2019, deferred compilation can sample, but only on first run).
- **Multi-statement TVFs.** Same problem — fixed estimate of 100.
- **`OPTION (RECOMPILE)`** is the brute-force fix when statistics are right but the parameter changes everything.

PostgreSQL's analog: `EXPLAIN (ANALYZE)` shows `rows=10` (estimated) `actual rows=2500000`. Same diagnosis — `ANALYZE table_name` to refresh statistics, or `default_statistics_target` to deepen the histograms.

## Parameter sniffing

```sql
CREATE PROC GetOrders @TenantId INT AS
SELECT * FROM Orders WHERE TenantId = @TenantId;
```

First execution: `@TenantId = 1` (a tenant with 50 rows). Optimizer compiles a Nested Loops plan with an index seek — perfect for 50 rows.
Second execution: `@TenantId = 2` (a tenant with 2 million rows). Plan reused: now it's 2 million Nested Loops iterations. p99 latency goes from 5 ms to 30 seconds.

Symptoms:

- "Sometimes it's fast, sometimes it's slow" with the same query and similar inputs.
- The plan in cache looks ridiculous for the parameter that's currently slow.
- Recompiling the proc temporarily fixes it (until the next bad first call).

Fixes, in order of preference:

1. **Better indexes.** A truly good covering index serves both shapes well; sniffing matters less.
2. **`OPTION (RECOMPILE)`.** Compiles fresh every call. Burns the plan cache benefit (~1–5 ms compile time per call). Acceptable for queries called <100/sec; intolerable for OLTP hot paths.
3. **`OPTION (OPTIMIZE FOR (@TenantId = 1))`.** Pin the plan to one parameter value. Useful when one shape dominates and you accept the other being slow.
4. **`OPTION (OPTIMIZE FOR UNKNOWN)`.** Disable sniffing; estimate from histogram density. Average plan, no extreme cases either way.
5. **Query Store plan forcing.** Pin a known-good plan from history. Survives recompiles, restarts, deploys. The senior tool — but read [Query Store](#query-store) below first.
6. **Split the query.** Two SQL shapes for two cardinality regimes. Crude but bulletproof when sniffing dominates.

EF Core: `.TagWith("OPTIMIZE_FOR_UNKNOWN")` plus a Query Store hint, or `EF.CompileQuery` for hot static-shape queries that want one plan forever.

## Query Store

SQL Server's built-in plan history (since 2016, on by default in 2022). Captures plans, runtime stats, regression detection, and lets you force a plan.

```sql
-- Top 10 queries by total CPU last 24 hours
SELECT TOP 10
    qsq.query_id,
    qsqt.query_sql_text,
    SUM(qsrs.count_executions)        AS executions,
    SUM(qsrs.avg_cpu_time * qsrs.count_executions) / 1000 AS total_cpu_ms,
    AVG(qsrs.avg_duration) / 1000     AS avg_duration_ms,
    SUM(qsrs.avg_logical_io_reads * qsrs.count_executions) AS total_reads
FROM sys.query_store_query qsq
JOIN sys.query_store_query_text qsqt ON qsq.query_text_id = qsqt.query_text_id
JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
JOIN sys.query_store_runtime_stats qsrs ON qsp.plan_id = qsrs.plan_id
JOIN sys.query_store_runtime_stats_interval qsrsi
    ON qsrs.runtime_stats_interval_id = qsrsi.runtime_stats_interval_id
WHERE qsrsi.start_time > DATEADD(HOUR, -24, SYSUTCDATETIME())
GROUP BY qsq.query_id, qsqt.query_sql_text
ORDER BY total_cpu_ms DESC;
```

The Studio's "Top Resource Consuming Queries" view is this query, prettier. Use it as the starting point for any "the database is slow" investigation — it's a cumulative view of what's actually expensive, not what looks expensive.

Plan forcing:

```sql
EXEC sp_query_store_force_plan @query_id = 12345, @plan_id = 67890;
```

Forces the optimizer to use the chosen plan for that query. SQL Server still validates the plan is still legal (indexes exist, schema matches); if it's not, it falls back gracefully. The right tool when you've identified a known-good plan and the optimizer keeps regressing to a known-bad one.

PostgreSQL: `pg_stat_statements` (extension) for cumulative query stats; `pg_hint_plan` (extension) for plan hints; no native plan-forcing at the per-query level.

## Reading plans from .NET

You don't need SSMS to inspect plans for queries your service runs. The DMVs expose them:

```sql
-- Find a plan by partial query text
SELECT TOP 10
    qs.execution_count,
    qs.total_worker_time / qs.execution_count / 1000 AS avg_cpu_ms,
    qs.total_logical_reads / qs.execution_count        AS avg_reads,
    SUBSTRING(st.text, qs.statement_start_offset/2 + 1,
        ((CASE qs.statement_end_offset
              WHEN -1 THEN DATALENGTH(st.text)
              ELSE qs.statement_end_offset
          END - qs.statement_start_offset)/2) + 1)     AS statement_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)        st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle)     qp
WHERE st.text LIKE '%Orders%CreatedAt%'
ORDER BY avg_cpu_ms DESC;
```

`qp.query_plan` is XML — paste into SSMS for the visual tree, or parse server-side.

For per-query inspection from C#, EF Core 7+: `.TagWithCallSite()` injects the C# file/line as a SQL comment, making it trivial to find a query's plan in the cache:

```csharp
var orders = await db.Orders
    .TagWithCallSite()
    .Where(o => o.TenantId == tenantId)
    .ToListAsync(ct);

// SQL contains:  -- file: OrdersService.cs:42
// Find in plan cache:
//   WHERE st.text LIKE '%OrdersService.cs:42%'
```

## Plan cache hygiene

```sql
-- Single-use plan bloat (queries that compile once, never reuse)
SELECT
    objtype, COUNT(*) AS plans, SUM(size_in_bytes)/1024/1024 AS size_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1
GROUP BY objtype
ORDER BY size_mb DESC;
```

A pile of single-use plans means the workload is *not parameterizing*. The cause is usually:

- String-built SQL with literals (`"WHERE Id = " + id`).
- `FromSqlRaw($"... {id}")` instead of `FromSqlInterpolated`.
- Auto-parameterization disabled / forced parameterization not on.

The fix is at the source — parameterize. The temporary mitigation: `ALTER DATABASE ... SET PARAMETERIZATION FORCED;` makes the engine try harder to substitute literals for parameters. Side effects exist (some plans get worse) — measure.

`DBCC FREEPROCCACHE` evicts everything; useful in test, dangerous in prod (a server with cold cache compiles every query for a while).

## PostgreSQL EXPLAIN cheat sheet

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT ... FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.tenant_id = $1 AND o.status = 'active'
ORDER BY o.created_at DESC LIMIT 50;
```

What to read:

- **`actual time=X..Y rows=N loops=L`** — `loops` is what to watch on Nested Loops; total time is `(Y) × loops`.
- **`Buffers: shared hit=H read=R`** — `read` means physical I/O (warm cache → near zero is healthy). `dirtied` and `written` flag write contention.
- **`Rows Removed by Filter: N`** — the operator filtered N rows post-fetch; high values mean the predicate isn't pushed to the index.
- **`Heap Fetches: N`** on `Index Only Scan` — non-zero means VACUUM is behind and the visibility map is stale.
- **`Workers Planned / Workers Launched`** — discrepancies mean the planner asked for parallelism but the executor couldn't grant it.

`EXPLAIN (ANALYZE, BUFFERS)` *runs the query for real* — don't run it on a 10-billion-row `DELETE` without `BEGIN; ... ROLLBACK;`.

## Worked example

A nightly report query took 14 minutes. Trace and fix:

1. **Capture the plan.** Query Store showed one query consuming 78% of last night's CPU. Fetched the plan via `dm_exec_query_plan`.
2. **Read the operators.** Topmost: `Hash Match (Aggregate)` over a `Nested Loops Join` driven by a `Clustered Index Scan` on `Orders` (50M rows). Estimated rows on the inner side: 1. Actual: 50M.
3. **Find the fat arrow.** The inner side was a `Key Lookup` against `Customers` for every order. 50M lookups × ~3 ms each = 41 hours of work, parallelized down to 14 minutes.
4. **Diagnose.** The optimizer used a non-covering index on `Orders.CustomerId` and decided lookup-per-row was cheaper than scanning `Customers` and hash-joining. Statistics on `Customers` claimed 100k rows; actual was 8M (last `UPDATE STATISTICS` was 6 months ago).
5. **Fix in two passes.** First, `UPDATE STATISTICS Customers WITH FULLSCAN` — re-ran query, plan flipped to `Hash Match Join`, query down to 90 seconds. Second, added a covering index on `Orders (CustomerId) INCLUDE (Total, CreatedAt)` for the lookup case — re-ran, query down to 12 seconds.
6. **Lock it in.** Forced the new plan in Query Store so the next nightly run can't regress.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| Tuning without an actual plan | Random changes; sometimes works, often doesn't |
| Treating estimated rows as truth | Misses the fat-arrow signal |
| `OPTION (RECOMPILE)` on every query | Eliminates plan cache; query compile time becomes the bottleneck |
| Forcing a plan without monitoring | Forced plan can become wrong as data shifts |
| `DBCC FREEPROCCACHE` in production "to refresh things" | Cold cache; every query recompiles; outage-shaped |
| Ignoring single-use plan bloat | Cache fills with one-off plans; useful plans evicted |
| Per-call `Recompile` for "simplicity" | Hides parameter-sniffing problems instead of fixing them |
| Long-running query without `STATISTICS IO ON` first | No baseline to confirm the fix worked |
| Reading the plan but ignoring missing-index hints in it | The optimizer literally tells you what to add; people skip it |
| `MAXDOP 1` everywhere "to be safe" | Serial plans for queries that should parallelize; report queries 4× slower |

## Senior-level gotchas

- **The yellow exclamation mark on an operator** in SSMS plans flags warnings: implicit conversions, no-statistics, spills to tempdb. Always click and read; these are the optimizer telling you something is wrong.
- **Implicit conversions kill seek behavior.** `WHERE NvarcharColumn = @VarcharParam` causes a `CONVERT_IMPLICIT(NVARCHAR, ...)` on every row, defeating the index. Match parameter types to column types — `SqlParameter` declares its type explicitly; EF infers from the property type, but raw ADO.NET requires care.
- **A "Compute Scalar" operator costing 0%** can still be wrong — it can hide a function call (`DateAdd`, `ISNULL`, a UDF) that prevents predicate push-down. Hover and read the expression.
- **Scalar UDFs serialize the plan**. Pre-2019, every call to a scalar UDF disables parallelism for the entire query. SQL 2019+ inlines simple UDFs (search "Scalar UDF Inlining"); complex ones still don't. Inline the logic or rewrite as iTVFs.
- **Table variables vs temp tables**: table variables have hardcoded cardinality estimates (1 pre-2019, sampled in 2019+); temp tables have full statistics. For >100 rows, prefer temp tables in stored procs.
- **`OPTION (FAST N)`** tells the optimizer to optimize for returning the first N rows quickly, not for total throughput. Useful for paginated UIs; harmful for ETL.
- **Spools (Lazy / Eager / Index Spool)** are the optimizer compensating for missing indexes by building a worktable. They mean "I really want a covering index here." Find the source columns and add one.
- **Parallelism cost threshold (`cost threshold for parallelism`) defaults to 5** — a 1990s value. On modern hardware, raise to 50–100; otherwise small queries waste a worker thread on a 2-row plan.
- **`MAXDOP` at the query, database, or server level** — the most-restrictive wins. A server `MAXDOP 0`, database `MAXDOP 4`, query `MAXDOP 8` runs at 4. Read all three before blaming serial plans.
- **The plan for a `SELECT TOP 1` against an unsorted predicate** can be very different from the same query with `TOP 100` — the optimizer uses the row goal to pick a Nested Loops plan that "stops early." A `TOP 100000` may flip to Hash Join with 100× the throughput. Inspect plans across `TOP` values for paginated reads.
- **Adaptive Joins (SQL 2017+)** let the runtime choose Hash vs Nested Loops based on actual outer rowcount at execution time. Look for "Adaptive Join" operator. Mitigates parameter sniffing for join shape, not for predicate selectivity.
- **Memory grants** are estimated at compile time and can't grow at runtime. An underestimated grant spills sorts and hashes to tempdb (look for the spill warning); an overestimated grant blocks other queries waiting for memory. `RESOURCE_SEMAPHORE` waits in `dm_exec_requests` are the symptom of overestimation.
- **Query Store has overhead** — typically 2–5% CPU for the recommended capture mode (`AUTO`). On extreme write workloads (millions of distinct queries/sec) the capture itself becomes a bottleneck; switch to `CUSTOM` and only capture meaningful queries.
- **The "Estimated Subtree Cost" number on a plan operator** is in *abstract optimizer units*, not seconds. Compare relatively (this branch is more expensive than that one), not absolutely. Use actual execution time / IO from the plan or `STATISTICS TIME, IO ON` for wall-clock numbers.
- **`SHOWPLAN_XML ON` doesn't run the query** — you can use it to capture an estimated plan for a query that would never finish. Combine with `SET STATISTICS XML ON` for actual plans, or use Query Store on a real run.
- **`OPTION (USE HINT (...))`** is the modern, supported way to inject hints (vs the legacy trace flags) — `'FORCE_DEFAULT_CARDINALITY_ESTIMATION'`, `'DISABLE_PARAMETER_SNIFFING'`, `'ENABLE_QUERY_OPTIMIZER_HOTFIXES'`. Survives upgrades; trace flags don't.
- **PostgreSQL `random_page_cost = 4`** (default) assumes spinning disks. On SSDs, lower to 1.1; the planner will switch to index scans more aggressively because random I/O is cheap. Single biggest plan-quality lever in PG on modern hardware.
