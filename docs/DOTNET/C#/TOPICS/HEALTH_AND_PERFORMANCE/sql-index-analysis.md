# SQL index analysis

_Targets SQL Server 2022 / PostgreSQL 16 / .NET 10. See also: [SQL query plan analysis](./sql-query-plan-analysis.md), [Relational database query optimization](./relational-database-query-optimization.md), [Pagination strategies](./pagination-strategies.md), [Aggregation efficiency](./aggregation-efficiency.md), [Bulk operations](./bulk-operations.md)._

An index is a sorted, narrow projection of one or more columns plus a pointer back to the row. Every read is a tradeoff between *which sorted projection serves the query* and *what the writes pay to keep them all sorted*. Most "the query is slow" tickets resolve to one of three things: there's no index for the predicate, there's an index but it's the wrong shape, or there are too many indexes and writes are throttling. This page is the diagnostic and design guide.

> The right index for a query is the one that lets the engine *seek* to the first matching row and *read forward sequentially* until done. Anything else — scan, sort, lookup — is overhead that scales with data size.

This is a SQL Server-first treatment with PostgreSQL deltas inline. Use [sql-query-plan-analysis](./sql-query-plan-analysis.md) for reading the plans these indexes change.

## Index types you actually use

| Type | Storage | When it fits | Caveat |
|---|---|---|---|
| Clustered (B-tree) | Pages sorted by key; *is* the table | One per table; pick the most natural sort key | Random `INSERT` keys cause page splits |
| Nonclustered (B-tree) | Separate B-tree, leaf has key + row pointer | Predicate / sort columns | Lookups back to clustered if non-covering |
| `INCLUDE` columns | Stored at leaf, not in key | Cover a query without bloating key | Adds storage but no maintenance cost on the included columns themselves |
| Filtered | Indexes only rows matching a predicate | "Most rows are inactive" patterns | Predicate must match query exactly |
| Columnstore (CCI / NCCI) | Column-major, batch-mode | OLAP / wide aggregates over millions of rows | OLTP-hostile; row writes go through delta store |
| Hash (In-Memory OLTP) | Hash bucket per key | Equality lookup at memory speed | No range scans; bucket count must be sized |
| Full-text | Inverted index of tokens | `CONTAINS`, `FREETEXT` | Async population; not transactional |
| Spatial | R-tree-ish (grid) | `STIntersects`, `STDistance` | Tuning hints required for selectivity |

PostgreSQL deltas: **B-tree** is default (analog of nonclustered); **GIN** for arrays / JSONB / full-text; **GiST** for geometric; **BRIN** for very large, naturally-clustered tables (timestamps in append-only tables); **Hash** for equality only (rebuilt to be crash-safe in 10+).

## The ESR rule for compound indexes

Order key columns as **Equality, Sort, Range**. The engine seeks to the leftmost equality predicates, traverses the sort columns in order, and ranges over the rightmost.

```sql
-- Query: WHERE TenantId = @t AND Status = @s AND CreatedAt > @c ORDER BY CreatedAt DESC
-- Right index: equalities first (interchangeable), then sort, then range.
CREATE NONCLUSTERED INDEX IX_Orders_TenantStatusCreated
    ON Orders (TenantId, Status, CreatedAt DESC)
    INCLUDE (Total, CustomerId);
```

Wrong ordering forces the engine to do extra work:

- `(CreatedAt, TenantId, Status)` — scans every CreatedAt range that matches any tenant, filters in memory.
- `(TenantId, CreatedAt, Status)` — seeks to the tenant, but `Status` filter is post-fetch (stalls the seek-on-second-key).

**Sort direction matters.** `ORDER BY CreatedAt DESC` against an index on `CreatedAt ASC` lets the engine read backward (still a seek) but loses parallelism on some shapes. Match the sort direction in the index for the cleanest plan.

## INCLUDE for covering

A *covering* index is one that contains every column the query needs — predicate, sort, *and* projection — so the engine never has to look up the row. Lookup cost is not negligible: each one is a random I/O against the clustered index leaf.

```sql
-- Query: SELECT Total, CustomerId FROM Orders WHERE TenantId = @t AND Status = @s
-- Without INCLUDE: seek on (TenantId, Status), then key-lookup per row for Total/CustomerId.
-- With INCLUDE:     seek + sequential read at the leaf. No lookup.

CREATE NONCLUSTERED INDEX IX_Orders_TenantStatus
    ON Orders (TenantId, Status)
    INCLUDE (Total, CustomerId);
```

`INCLUDE` columns are stored only at the leaf level, not in the upper B-tree pages — so they don't bloat the key, don't affect seek cost, and don't impose a sort order. The cost is leaf-page storage and the write cost when an included column changes.

PostgreSQL: same concept, same syntax (`INCLUDE` clause added in PG 11).

## Missing-index DMVs (SQL Server)

Every query the optimizer compiles records what indexes it *would have liked*. Surface them:

```sql
SELECT TOP 25
    DB_NAME(d.database_id)        AS db,
    s.avg_total_user_cost * (s.avg_user_impact / 100.0) * (s.user_seeks + s.user_scans) AS improvement,
    d.statement                   AS table_name,
    'CREATE NONCLUSTERED INDEX IX_' + REPLACE(REPLACE(REPLACE(d.statement, '[', ''), ']', ''), '.', '_')
        + ' ON ' + d.statement
        + ' (' + ISNULL(d.equality_columns, '')
        + CASE WHEN d.equality_columns IS NOT NULL AND d.inequality_columns IS NOT NULL
               THEN ',' ELSE '' END
        + ISNULL(d.inequality_columns, '') + ')'
        + CASE WHEN d.included_columns IS NOT NULL THEN ' INCLUDE (' + d.included_columns + ')' ELSE '' END
        AS create_statement
FROM sys.dm_db_missing_index_group_stats s
INNER JOIN sys.dm_db_missing_index_groups g ON s.group_handle = g.index_group_handle
INNER JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
WHERE d.database_id = DB_ID()
ORDER BY improvement DESC;
```

Read the **`improvement`** column as "cost saved per query × number of times the optimizer wanted it." Anything over ~10000 is worth investigating; the top 5 are usually obvious wins.

DMV caveats:

- **Reset on instance restart** — gather data over a representative window (a full business day, ideally a week).
- **Equality columns may be in any order** — the DMV doesn't tell you the right key order for the workload. Apply the ESR rule yourself based on the queries in the plan cache.
- **Won't suggest filtered indexes, columnstore, or partitioned indexes** — those are design decisions, not statistical observations.
- **`included_columns` may be wrong** — the DMV picks up every column the query touched, not only those that actually need to be in the leaf. Trim aggressively.

PostgreSQL has no missing-index DMV; use `pg_qualstats` (extension) or read slow-query logs for the same signal.

## Index usage stats — find writes that nobody reads

Every nonclustered index makes writes slower. Find indexes that are written but never (or rarely) read:

```sql
SELECT
    OBJECT_NAME(i.object_id) AS table_name,
    i.name                   AS index_name,
    s.user_seeks, s.user_scans, s.user_lookups, s.user_updates,
    (s.user_updates - (s.user_seeks + s.user_scans + s.user_lookups)) AS net_writes
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
    ON i.object_id = s.object_id AND i.index_id = s.index_id AND s.database_id = DB_ID()
WHERE i.type_desc = 'NONCLUSTERED'
  AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
ORDER BY net_writes DESC;
```

A row with `user_seeks = 0`, `user_scans = 0`, `user_updates = 1.2M` is dead weight: every insert/update/delete pays to keep it sorted, and nobody reads the result. Drop it.

Also reset on restart, and missing from `dm_db_index_usage_stats` doesn't always mean unused — could mean "never written to" *or* "the cache was reset." Confirm against Query Store before dropping.

PostgreSQL: `pg_stat_user_indexes` exposes `idx_scan`, `idx_tup_read`, `idx_tup_fetch`. Same logic: zero `idx_scan` over a meaningful window = candidate for drop.

## Fragmentation and fill factor

```sql
SELECT
    OBJECT_NAME(ips.object_id)                    AS table_name,
    i.name                                        AS index_name,
    ips.avg_fragmentation_in_percent              AS frag_pct,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 1000
  AND ips.avg_fragmentation_in_percent > 10
ORDER BY frag_pct DESC;
```

Standard thresholds:

| Fragmentation | Action |
|---|---|
| < 10% | Leave alone |
| 10–30% | `ALTER INDEX ... REORGANIZE` (online, light) |
| > 30% | `ALTER INDEX ... REBUILD WITH (ONLINE = ON)` (heavier, refreshes statistics) |

`FILLFACTOR = 80–90` leaves room on each leaf page for in-page inserts, reducing page splits on B-trees with non-monotonic insert order (e.g. random `Guid` clustered keys — which are bad for other reasons; see below). Default `FILLFACTOR = 0` (= 100%) packs pages tight, optimal for read-only / append-only.

PostgreSQL: `pg_stat_user_tables` exposes `n_dead_tup`; the analog of fragmentation is bloat, addressed by `VACUUM` / `pg_repack`. `fillfactor` is a per-table storage parameter.

## Filtered indexes

When 95% of rows have a default value and queries always filter to the 5%:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_PendingByCustomer
    ON Orders (CustomerId, CreatedAt DESC)
    INCLUDE (Total)
    WHERE Status = 'Pending';
```

The index stores only pending rows — perhaps 5% of the table. Storage drops 20×; query cost drops because every page traversed is relevant.

The trap: **the query predicate must exactly match the filter** for the optimizer to pick it. `WHERE Status = 'Pending'` works; `WHERE Status IN ('Pending')` may not (depends on version / cardinality). Inspect the plan to confirm the filtered index is selected.

PostgreSQL equivalent: partial indexes — same syntax (`WHERE` clause on `CREATE INDEX`), same matching requirement.

## Worked example: end-to-end fix

Symptom: an EF query takes 800 ms p95.

```csharp
var page = await db.Orders
    .Where(o => o.TenantId == tenantId && o.Status == OrderStatus.Active)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new { o.Id, o.Total, o.CreatedAt, o.CustomerId })
    .Take(50)
    .ToListAsync(ct);
```

Translated SQL:

```sql
SELECT TOP 50 [o].[Id], [o].[Total], [o].[CreatedAt], [o].[CustomerId]
FROM [Orders] AS [o]
WHERE [o].[TenantId] = @__tenantId AND [o].[Status] = 1
ORDER BY [o].[CreatedAt] DESC;
```

**Step 1: get the plan.** `SET STATISTICS IO, TIME ON;` — see "Index Scan + Sort + Top". Reads = 12000 pages. The existing index is `(TenantId)` only.

**Step 2: missing-index DMV.** Confirms the optimizer wants `(TenantId, Status)` with `INCLUDE (Id, Total, CreatedAt, CustomerId)`.

**Step 3: apply the ESR rule.** Equalities = `TenantId`, `Status`. Sort = `CreatedAt DESC`. Range = none.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_TenantStatusCreated
    ON Orders (TenantId, Status, CreatedAt DESC)
    INCLUDE (Id, Total, CustomerId)
    WITH (ONLINE = ON, FILLFACTOR = 90);
```

**Step 4: re-check the plan.** "Index Seek + Top". Reads = 4 pages. p95 latency = 8 ms.

**Step 5: drop the old single-column `(TenantId)` index** if it's redundant — the new compound index serves any query that filtered on `TenantId` alone, just less efficiently than a single-column would. Inspect `dm_db_index_usage_stats` before dropping.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `Guid` (random) clustered key | Constant page splits on insert; index fragmentation; 80%+ within days |
| One nonclustered index per column "just in case" | Write amplification; pool of indexes the optimizer ignores |
| `INCLUDE` of columns the query doesn't return | Bloat without payoff; writes slower |
| Ignoring missing-index DMV for a year | Compounds: the optimizer reaches the same conclusion 10 million times |
| Rebuilding indexes on a schedule regardless of fragmentation | Wasted CPU + log; small fragmentation costs nothing |
| Filtered index whose predicate doesn't match query syntax | Index never selected; DBA's "I built it but no one uses it" |
| Compound index in alphabetical column order | Never matches ESR; full scan in disguise |
| Columnstore on an OLTP table | Delta-store thrashing; updates 10× slower |
| `INDEX REORGANIZE` on a 70%-fragmented index | Slower than `REBUILD`; spends CPU without resolving the split chains |
| Adding an index without checking write workload | Writes go from 5 ms to 50 ms; you don't notice until the next deploy |

## Senior-level gotchas

- **The clustered key is appended to every nonclustered index automatically.** A wide clustered key (e.g. composite of three columns) bloats every nonclustered index by that amount. Pick a narrow clustered key — `INT IDENTITY` or `BIGINT IDENTITY` — for OLTP tables.
- **`SELECT *` defeats covering indexes.** A query that lists only the columns it needs may be served by a narrow covering index; widen it to `*` and you force a key lookup or a full clustered scan. EF's `Select(o => new { ... })` is the production discipline.
- **Statistics are sampled at 1% by default for tables over 8 MB.** For wildly skewed data (one tenant has 10M rows, others have 100), the sample misses the skew and the plan is built for the mean. `UPDATE STATISTICS ... WITH FULLSCAN` for skewed key columns; `AUTO_UPDATE_STATISTICS_ASYNC = ON` to avoid plan compilation blocking the query.
- **Index intersection happens, but rarely well.** The optimizer can combine two single-column indexes by intersecting their results — but it usually can't beat a single compound index. Don't rely on intersection to avoid the compound index.
- **`HASH WITH (BUCKET_COUNT = N)` on memory-optimized tables wants 1–2× the distinct key count.** Under-sized buckets degrade to chained scans; over-sized waste memory.
- **Columnstore ordered by an `ORDER` clause** (SQL 2022) keeps row groups sorted, which the optimizer can use for segment elimination. Without ordering, columnstore segments are inserted in arrival order and segment elimination only fires when the cardinality estimate aligns by luck.
- **Filtered indexes count rows differently for cardinality estimation** — the optimizer uses the filtered statistics, not the full-table statistics. A predicate that matches the filter exactly gets a tighter estimate; a slightly different predicate may not benefit.
- **`OPTIMIZE FOR UNKNOWN`** in a query hint disables parameter sniffing for that plan — the optimizer estimates cardinality from the column histogram density, not the actual parameter. Useful when the parameter is highly skewed and sniffing ruins one of two common cases.
- **Page splits log every move under `BULK_LOGGED` or `FULL` recovery.** A heavily-fragmenting clustered key can fill the log faster than backups can clear it. The fix is the right key, not bigger logs.
- **`sys.dm_db_index_operational_stats` is the per-IO view** of index usage — number of latch contentions, page-level lock waits, row-level lock escalations. When `dm_db_index_usage_stats` says "this index is used", `dm_db_index_operational_stats` says "and it's hot enough that it's a contention point."
- **PostgreSQL `bloat` is not the same as fragmentation.** Bloat is dead tuples that `VACUUM` hasn't reclaimed; fragmentation in the SQL Server sense doesn't really exist in PG (no clustered indexes). Use `pgstattuple` to measure.
- **A unique index always wins over a non-unique index for the same predicate.** If the column is in fact unique, declare it — the optimizer uses uniqueness for better cardinality estimates and can skip work in many shapes.
- **Composite uniqueness across multiple nullable columns** behaves differently in SQL Server (NULLs are *equal* under a unique constraint by default — only one NULL row allowed) and PostgreSQL (NULLs are *not equal* — many NULL rows allowed). The .NET / EF translation is silent about this.
- **`INCLUDE` columns are *not* updated on read** — but `UPDATE` of an included column does update the index leaf. A counter column updated 1M times/day in three nonclustered indexes is 3M leaf-page writes/day. Audit included columns the same way you audit key columns.
- **A single index can serve queries with prefix subsets of the key**: an index `(A, B, C)` serves `WHERE A = ?`, `WHERE A = ? AND B = ?`, `WHERE A = ? AND B = ? AND C = ?`. It does *not* serve `WHERE B = ?` alone. Build the leftmost index for the most common predicate.
- **Index scans that look like seeks**: a "Seek" with a residual predicate (`Seek Predicate: ... Predicate: ...`) is doing extra row-level filtering after the seek. Often means the index isn't on the right columns. Read the operator's `Predicate` property in the plan.
- **Online index builds in SQL Server use a row-versioning side table** — they're not free. On a 1B-row table, an `ONLINE = ON` rebuild can take hours and consume tempdb. Plan a maintenance window for the heaviest indexes; the smaller ones are fine online.
