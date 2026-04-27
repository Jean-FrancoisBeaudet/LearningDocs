# Microsoft SQL Server & Azure SQL — Interview Refresher

> Single-document refresher for a broad technical interview where SQL Server / Azure SQL is one of several technologies that may come up — not a deep-dive and not a DP-300 study guide. Targets "experienced practitioner" depth: enough that an interviewer can tell you've actually built and operated systems on the Microsoft data stack and understand the *why* behind the choices. Reflects current Azure SQL semantics and SQL Server 2022. For deeper notes per topic, see the [Further study](#11-further-study) index at the end.

---

## 1. T-SQL fundamentals worth being sharp on

Modern T-SQL has the expressive set-based features you actually need; if you're reaching for a cursor, stop and re-think. The interview-relevant pieces:

### Joins, anti-joins, and `EXISTS`

`INNER` / `LEFT` / `RIGHT` / `FULL OUTER` / `CROSS` are table stakes. The two patterns that come up *constantly* and that interviewers use to separate juniors from seniors:

```sql
-- Anti-join: customers with no orders. Idiomatic, set-based, optimizer-friendly.
SELECT c.CustomerId
FROM dbo.Customers AS c
WHERE NOT EXISTS (
    SELECT 1 FROM dbo.Orders AS o WHERE o.CustomerId = c.CustomerId
);

-- Same logical result with LEFT JOIN ... IS NULL. Plans usually identically;
-- the EXISTS form reads better and cannot accidentally inflate rows on a 1-to-many join.
SELECT c.CustomerId
FROM dbo.Customers AS c
LEFT JOIN dbo.Orders AS o ON o.CustomerId = c.CustomerId
WHERE o.OrderId IS NULL;
```

`EXISTS` is "do any rows match?" — it stops at the first hit and never duplicates the outer row. Prefer it for existence checks and anti-joins. `IN` with a subquery is fine for short lists; `IN` with a large or correlated subquery is a tripwire — most teams use `EXISTS` everywhere for consistency.

### CTEs and recursive CTEs

A CTE is a named subquery scoped to one statement — readability tool, not an optimization barrier. Recursive CTEs (`WITH cte AS (anchor UNION ALL recursive)`) are the go-to for hierarchies, graph walks, and date series:

```sql
-- Walk an org chart from a given manager down. Recursive CTE; MAXRECURSION caps depth.
WITH org AS (
    SELECT EmployeeId, ManagerId, 1 AS Depth
    FROM dbo.Employees WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmployeeId, e.ManagerId, org.Depth + 1
    FROM dbo.Employees AS e
    INNER JOIN org ON org.EmployeeId = e.ManagerId
)
SELECT * FROM org OPTION (MAXRECURSION 100);  -- default is 100; 0 = unlimited
```

### Window functions

The single biggest force-multiplier in modern T-SQL. `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG/LEAD`, and aggregate windows (`SUM(...) OVER (...)`) replace huge classes of self-joins.

| Function | Behavior on ties |
|---|---|
| `ROW_NUMBER` | Always increments — picks one row arbitrarily on a tie. |
| `RANK` | Same rank for ties, then *skips* — 1, 2, 2, 4. |
| `DENSE_RANK` | Same rank for ties, no skip — 1, 2, 2, 3. |

Frame clauses (`ROWS BETWEEN ... AND ...` vs the default `RANGE`) matter — `RANGE` includes peers and is rarely what you want for running totals. Default frame for `SUM(...) OVER (ORDER BY ...)` is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which produces an unwanted "all peers included" effect when the order-by has duplicates. Be explicit with `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`:

```sql
-- Running daily revenue per customer.
SELECT CustomerId, OrderDate, OrderTotal,
       SUM(OrderTotal) OVER (
           PARTITION BY CustomerId
           ORDER BY OrderDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- why: avoid RANGE peer surprise
       ) AS RunningTotal
FROM dbo.Orders;
```

### `MERGE`

Looks elegant — one statement that does `INSERT` / `UPDATE` / `DELETE`. In practice `MERGE` has a long history of correctness bugs under concurrency (lost updates, duplicate-key violations) and the standard senior-DBA take is: avoid it, or if you must use it, take an explicit `HOLDLOCK`/`SERIALIZABLE` hint on the target. Many teams have a "no `MERGE` in production" rule and use separate idempotent `UPDATE` + `INSERT` blocks (or `INSERT ... WHERE NOT EXISTS`) instead. Worth knowing the syntax; worth knowing it has a reputation.

### `CROSS APPLY` / `OUTER APPLY`

Like a join, but the right side can reference columns from the left and can be a table-valued function or a correlated `TOP N`. The textbook use case is "top N child rows per parent":

```sql
-- 3 most recent orders for each customer. CROSS APPLY beats a correlated ROW_NUMBER subquery here.
SELECT c.CustomerId, o.OrderId, o.OrderDate
FROM dbo.Customers AS c
CROSS APPLY (
    SELECT TOP (3) OrderId, OrderDate
    FROM dbo.Orders AS o2
    WHERE o2.CustomerId = c.CustomerId
    ORDER BY o2.OrderDate DESC
) AS o;
```

`CROSS APPLY` ≈ `INNER JOIN`, `OUTER APPLY` ≈ `LEFT JOIN` — they preserve / drop the outer row when the right side is empty.

### Pivoting

`PIVOT` exists; almost everyone in production uses the conditional-aggregation pattern instead — easier to read, easier to maintain, no quoting gymnastics:

```sql
-- Conditional aggregation. Beats PIVOT for almost every real case.
SELECT CustomerId,
       SUM(CASE WHEN Status = 'Shipped'   THEN 1 ELSE 0 END) AS Shipped,
       SUM(CASE WHEN Status = 'Pending'   THEN 1 ELSE 0 END) AS Pending,
       SUM(CASE WHEN Status = 'Cancelled' THEN 1 ELSE 0 END) AS Cancelled
FROM dbo.Orders
GROUP BY CustomerId;
```

### Gaps and islands

A classic interview chestnut. The trick: assign a stable group key to consecutive runs by subtracting `ROW_NUMBER()` from a sequence. Rows in the same "island" share the result.

```sql
-- Find runs of consecutive days with at least one event per day.
SELECT MIN(EventDate) AS RunStart, MAX(EventDate) AS RunEnd, COUNT(*) AS Days
FROM (
    SELECT EventDate,
           DATEADD(DAY,
               -ROW_NUMBER() OVER (ORDER BY EventDate),
               EventDate) AS GroupKey  -- why: consecutive days produce identical GroupKey
    FROM dbo.DailyEvents
) AS x
GROUP BY GroupKey;
```

If you can write that without notes, you're past the bar an interviewer is checking for.

---

## 2. Schema and data modeling

### Normalization vs denormalization

Normalize for **write integrity** (one fact in one place — updates can't go inconsistent), denormalize for **read shape** (the query you run a million times shouldn't re-derive the same join). 3NF is the default; deviate consciously when a workload measurably needs it. The pathologies that come up in interviews:

- Pre-mature denormalization "for performance" before you have query data — usually creates update anomalies and saves nothing.
- Over-normalized schemas where reading one logical entity touches eight tables and every page is a join — usually a sign of EAV creep or modeling concepts as rows that should be columns.

### Primary and foreign keys

A primary key is a constraint plus, by default, the clustered index. A foreign key enforces referential integrity, helps the optimizer (it knows the relationship is trustable for join elimination), and gates cascade behavior. The trap senior practitioners flag: `WITH NOCHECK ... NOT FOR REPLICATION` or otherwise *non-trusted* foreign keys. The optimizer treats untrusted constraints as if they didn't exist, and your "I have FKs" plan is doing redundant work.

```sql
-- Check whether your FKs are actually trusted (is_not_trusted = 0 means trusted).
SELECT name, is_not_trusted
FROM sys.foreign_keys
WHERE is_not_trusted = 1;
```

### Clustered vs nonclustered indexes

The single most important storage concept in SQL Server.

- **Clustered index = the table.** The leaf level of the clustered index *is* the row. One per table.
- **Nonclustered indexes** are separate B-trees sorted on their key columns; the leaf level holds the *clustering key* (or a `RID` if the table is a heap), which the engine uses to look up the actual row when needed.
- Every nonclustered index implicitly contains the clustering key. That's why a wide clustering key (a 16-byte GUID, a multi-column natural key) bloats every nonclustered index on the table — pick a narrow, ever-increasing clustering key when you can.

### Included columns and covering indexes

A nonclustered index can carry extra columns *only at the leaf level* via `INCLUDE`. They aren't sorted on, so they don't widen the B-tree's intermediate pages, but they let the index "cover" a query — every column the query needs is in the index, no key lookup required.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Date
ON dbo.Orders (CustomerId, OrderDate)
INCLUDE (OrderTotal, Status);  -- why: covers SELECT OrderDate, OrderTotal, Status without lookup
```

Reading the plan: a **Key Lookup** (or RID Lookup) means the index found the row but had to go back to the clustered index for missing columns. If that lookup runs once per row across thousands of rows, you've found a candidate for `INCLUDE` columns — *if* the write cost is acceptable.

### Filtered indexes

Index a subset of the table. Smaller, cheaper to maintain, only useful if the filter matches your queries' predicates literally.

```sql
-- Most queries hit only active customers — index just those.
CREATE NONCLUSTERED INDEX IX_Customers_Active_Email
ON dbo.Customers (Email)
WHERE IsActive = 1;  -- query MUST contain WHERE IsActive = 1 (literal) to use this
```

Sharp-edge: parameterized queries (`WHERE IsActive = @flag`) won't use a filtered index unless `@flag` is inlined or `OPTION (RECOMPILE)` is used. This catches people.

### Index design feeds plan choice

The optimizer picks plans based on what indexes exist and what the statistics say is selective. Three things to internalize:

1. The **leading column** of an index is what makes it usable for an equality or range seek. An index on `(A, B)` helps `WHERE A = ?` and `WHERE A = ? AND B = ?` — it does *not* help `WHERE B = ?`.
2. **Order matters**: put the equality predicates first, then the range predicate (high-selectivity equality before lower-selectivity range), then `ORDER BY` columns if they match.
3. **Indexes have a write tax.** Every nonclustered index has to be maintained on `INSERT` / `UPDATE` (of indexed columns) / `DELETE`. Adding "just one more index" is never free.

---

## 3. Engine internals at a practical level

### Query optimizer in one paragraph

The optimizer is **cost-based**: it generates candidate plans, costs each one using statistics about row distribution, and picks the cheapest within a time budget. It does *not* search the full plan space — it stops when it finds a "good enough" plan ("trivial plan" → quick search → full search). Plans land in the **plan cache** keyed by the statement's hash; subsequent calls reuse the cached plan rather than re-optimizing. This caching is the source of *parameter sniffing* (below).

### Statistics

Statistics are histograms (and density vectors) describing column value distributions. The optimizer uses them to estimate row counts at every step of the plan. If statistics are stale — meaning the data has changed substantially since they were last sampled — estimates drift and plans go bad. SQL Server auto-creates and auto-updates statistics by default; the auto-update threshold on modern versions is roughly "20% of rows changed, with a square-root-based dampener for large tables." When somebody says "I updated stats and the query got fast again," they usually mean estimates were wildly wrong and a fresh sample fixed the plan choice.

```sql
-- Manually refresh stats on a table (full scan is expensive but accurate).
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
```

### Parameter sniffing

The canonical SQL Server performance pitfall. The optimizer compiles a plan based on the *first* parameter values it sees and caches it. If those values were atypical (an account with 10 rows when the average has 10 million, or vice-versa), every subsequent execution gets that bad plan. Symptoms: "the same proc is fast for me and slow for them," or "it was fast yesterday and slow today," after a server restart or stats update.

The fix menu, ranked by "how often I'd actually use it":

1. **Fix the query / index first.** Most parameter-sniffing pain is amplified by missing or wrong indexes; a well-designed index gives a single good plan that's robust across parameter values.
2. **Query Store forced plan.** When you know the good plan, force it. Lowest-friction fix in modern versions.
3. **`OPTION (RECOMPILE)`** on the offending statement. Trades CPU for stability — recompiles every call. Fine for low-frequency procs.
4. **`OPTION (OPTIMIZE FOR @p = literal)`** — pin to a representative value.
5. **`OPTION (OPTIMIZE FOR UNKNOWN)`** — use density (average distribution) instead of histogram. A blunt instrument; usually worse than (2) or (3).

### Reading execution plans

Read **right to left, top to bottom**. The thing on the far right is what runs first; the leftmost root is the final result. What to focus on:

- **Fat arrows** = lots of rows. A fat arrow into a Nested Loops join is bad news.
- **Estimated rows vs actual rows.** Big mismatch = the optimizer was lied to, usually by stale stats, parameter sniffing, or non-SARGable predicates.
- **Operators that pop up in trouble plans:** `Key Lookup` (missing covering index), `Hash Match` (often fine, sometimes a sign of missing index), `Sort` (the optimizer needed sorted input — could often be served by index ordering), `Spool` (intermediate materialization — sometimes fine, sometimes a smoking gun).
- **Estimated vs actual plan.** Estimated never executed; actual ran. *Always* prefer actual when you have it. `SET STATISTICS IO, TIME ON` adds the IO and CPU detail.

### Locking and blocking

Locks are the engine's mechanism for serializing access. Granularity escalates: row → page → object. **Lock escalation** to a table lock kicks in around 5,000 locks held by a single statement — after which everyone else trying to touch that table queues. The interview-grade picture:

| Lock mode | Conflicts with |
|---|---|
| `S` (Shared) | `X` |
| `U` (Update) | `U`, `X` |
| `X` (Exclusive) | everything |
| `IS` / `IX` / `IU` (Intent) | The intent variant of a conflicting mode |
| `Sch-M` (Schema mod) | everything |

Live blocking lookup:

```sql
SELECT session_id, blocking_session_id, wait_type, wait_time, wait_resource,
       command, status
FROM sys.dm_exec_requests
WHERE blocking_session_id <> 0;
```

### Isolation levels

| Level | Dirty reads | Non-repeatable | Phantoms | Mechanism |
|---|---|---|---|---|
| `READ UNCOMMITTED` (`NOLOCK`) | yes | yes | yes | reads ignore X locks — also missed/duplicated rows on page splits |
| `READ COMMITTED` (default) | no | yes | yes | S locks released as rows are read |
| `READ COMMITTED SNAPSHOT` (RCSI) | no | yes | yes | reads from row version in tempdb instead of taking S locks |
| `REPEATABLE READ` | no | no | yes | S locks held to end of transaction |
| `SNAPSHOT` | no | no | no | entire transaction sees the row versions as of its start |
| `SERIALIZABLE` (`HOLDLOCK`) | no | no | no | range locks |

The two snapshot variants both use the **version store in tempdb** and use no read locks. RCSI is statement-scoped (each statement sees a consistent snapshot of *that statement's* start); SNAPSHOT is transaction-scoped (the whole transaction sees a snapshot of *its* start, and you can hit "update conflict" errors when committing).

### Tempdb

The shared scratch space for the entire instance. Used for: temporary tables (`#temp`), table variables (sometimes), sort/hash spills, the row-version store for RCSI/SNAPSHOT, internal worktables. Heavy reliance + a single tempdb data file = PFS/GAM/SGAM page contention under load. The standard fix: multiple equally-sized tempdb data files (8 is the conventional starting point on dedicated boxes), trace flag 1118 on older versions (default behavior on 2016+). On Azure SQL the platform handles this for you, which is one of several reasons "Azure SQL is hard to misconfigure" is a real thing.

---

## 4. Performance tuning

### Diagnostic order

Don't guess. The standard sequence:

1. **Wait stats** (`sys.dm_os_wait_stats`) — what is the engine actually *waiting on*? CPU, IO, locks, latches, network. Compare deltas, not absolutes.
2. **Query Store** — top resource consumers, top regressions over the last hour/day, plan changes that correlate with regressions. This is the first thing to open in 2026.
3. **DMVs** for specifics: `sys.dm_exec_query_stats` for cached plan stats, `sys.dm_db_index_usage_stats` for dead indexes, `sys.dm_exec_requests` for live blocking.
4. **Actual execution plan** with `SET STATISTICS IO, TIME ON` — for the suspect query.
5. **Extended Events** session — for repeating but low-frequency issues you can't catch live (deadlocks, occasional long-running statements, schema changes).

If you skip (1) and go straight to plans, you'll often tune a query that wasn't the bottleneck.

### Common anti-patterns

- **Implicit conversions.** A `WHERE Email = @email` where `Email` is `VARCHAR(255)` and `@email` is `NVARCHAR(255)` (the .NET / Entity Framework default) forces an `nvarchar` conversion on every row in the index — index seek becomes an index scan. Either fix the column type or the parameter type.
- **Function-wrapped columns.** `WHERE YEAR(OrderDate) = 2025` or `WHERE LOWER(Email) = @x` make the predicate non-SARGable — the index can't be seeked. Rewrite as a range:
  ```sql
  WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'
  ```
- **Leading-wildcard `LIKE`.** `LIKE '%foo'` cannot seek a B-tree. Either fix the search to be a prefix, build a reverse-string column, or use full-text search.
- **`OR` across separate indexes.** The optimizer often gives up and scans. Rewrite as `UNION ALL` over two seeks when the predicates target different indexes and de-dup is unnecessary.
- **`WHERE column = ISNULL(@p, column)`** — also non-SARGable. Use `WHERE @p IS NULL OR column = @p` with `OPTION (RECOMPILE)`, or branch the query.

### When to add an index vs. rewrite the query

The missing-index suggestions in SSMS / DMVs are *hints based on this plan*, not a recommendation. Three checks before you create one:

1. Does this column set already have an existing index that's *almost* covering? Add an `INCLUDE` instead of a new index.
2. What's the write cost? Indexes on hot-write columns are expensive; check `sys.dm_db_index_operational_stats`.
3. Is the query non-SARGable? An index doesn't help if the predicate prevents a seek — rewrite the predicate first, then re-evaluate.

If five of your top-ten regressed queries all want overlapping indexes, you have an index-design problem, not five missing-index problems.

---

## 5. Transactions and concurrency

### ACID, briefly

- **Atomicity** — all or nothing.
- **Consistency** — constraints hold before and after.
- **Isolation** — degree to which concurrent transactions interfere (the isolation-level table above).
- **Durability** — committed = on durable storage. Implementation: **Write-Ahead Logging**. The transaction log is flushed to disk *before* the data pages it describes. After a crash, recovery replays committed transactions from the log and rolls back uncommitted ones.

### RCSI vs SNAPSHOT — the practical difference

Both eliminate reader-writer blocking by reading row versions from tempdb. Where they differ:

- **RCSI** — set once at the database level (`ALTER DATABASE … SET READ_COMMITTED_SNAPSHOT ON`). It's a drop-in replacement for the default `READ COMMITTED` — every query just stops taking S locks. Each *statement* sees a consistent snapshot of its own start.
- **SNAPSHOT** — set at the database level *and* requested per-transaction (`SET TRANSACTION ISOLATION LEVEL SNAPSHOT`). The whole *transaction* sees a snapshot of its start, all the way through. If you try to update a row that was changed by another transaction since you started, you get **error 3960** ("update conflict") and have to retry.

Most OLTP shops turn on RCSI as the default — it solves the most common reader-writer blocking complaint with no application changes — and use SNAPSHOT only for specific reporting transactions that need the strict snapshot-isolation guarantee.

### Deadlocks

Two transactions, opposite lock order. Engine detects the cycle, picks a "victim" by least-cost rollback, rolls it back with error 1205. Mitigations:

1. **Order access consistently.** If every transaction touches `Customers` before `Orders`, they can't deadlock on those two.
2. **Shorten transactions.** Long-running transactions hold locks longer and widen the deadlock window. Don't open a transaction, then read user input, then commit — that's a real production pattern.
3. **Use the right isolation level.** RCSI eliminates many reader-writer deadlocks for free.
4. **Retry pattern.** Deadlocks under contention are normal; the application should catch error 1205 and retry the transaction with a small backoff.
5. **Capture and analyze the deadlock graph.** Extended Events `xml_deadlock_report` is the modern way; it tells you exactly which two statements and which two resources collided.

### Optimistic vs pessimistic concurrency

- **Pessimistic** — take locks and hold them. The engine's default. Safe; potentially blocks a lot under contention.
- **Optimistic** — don't lock; verify at write time that nobody changed the row. Two flavors:
  - **Engine-level optimistic** = SNAPSHOT isolation (above). The engine tracks versions; you handle the conflict error.
  - **App-level optimistic** = a `rowversion` (formerly `timestamp`) column on the table. Read it with the row, send it back on update, and the `WHERE` clause filters on it — if the row changed since you read it, the update affects 0 rows and you handle it as a conflict.

```sql
-- App-level optimistic pattern.
UPDATE dbo.Orders
SET Status = @newStatus
WHERE OrderId = @id
  AND RowVersion = @rvFromClient;  -- why: 0 rows affected = somebody else won
```

EF Core, Dapper, and other ORMs build this in if you map a `rowversion` column as a concurrency token.

---

## 6. Azure SQL specifics

### The three deployment options

| | Azure SQL Database | Azure SQL Managed Instance | SQL Server on Azure VM |
|---|---|---|---|
| Surface area | Single database (or elastic pool) | Whole-instance — multi-DB, SQL Agent, cross-DB queries | Whole VM — full SQL Server |
| Surface compatibility | Subset of SQL Server | Near-100% feature parity | Identical |
| Networking | Public endpoint or private endpoint | VNet-injected by default | Whatever you build |
| Patching | Microsoft, automatic | Microsoft, automatic | You |
| HA / backup | Built in, automatic | Built in, automatic | You build it |
| Use it when… | Greenfield app, single DB scope | Lift-and-shift from on-prem (especially with linked servers, SQL Agent jobs, cross-DB) | You need CLR / FILESTREAM / specific OS access / strict vendor support requirements |

### DTU vs vCore

Two purchasing models for Azure SQL Database. **DTU** bundles compute, memory, and IO into a single opaque metric — simple, but you can't tune what's actually constraining you. **vCore** unbundles them — pick CPU count, pick memory, pick storage tier separately. Almost everyone picks vCore now; DTU is mostly residual on small workloads. vCore also unlocks Azure Hybrid Benefit (BYOL of existing SQL Server licenses) and the higher service tiers.

### Service tiers (vCore model)

| Tier | Architecture | Why pick it |
|---|---|---|
| **General Purpose** | Compute + remote (Azure Storage) data files | Cheap default, decent for most workloads, IO latency higher because storage is remote |
| **Business Critical** | Compute + local SSD + Always On AG with sync replicas | Lowest latency (local SSD, single-digit-ms log), built-in read-only replica, fastest failover; pricier |
| **Hyperscale** | Decoupled compute, log service, page servers | Decouples compute and storage; near-instant database restore via storage snapshots; scales to multi-TB; readable secondaries; the place for very large databases |

The mental model: GP is "remote disks," BC is "local disks + always-on replicas," Hyperscale is "decoupled architecture, scales sideways." Pick BC for low-latency OLTP, Hyperscale for very large DBs or fast-restore requirements, GP otherwise.

### Built-in HA, backups, geo-replication

What Azure SQL does for you and you don't have to configure:

- **HA** — every database has redundant replicas. BC and Hyperscale have synchronous replicas; GP has redundant storage. Failover is automatic; from your app's perspective it's a connection drop and a retry.
- **Backups** — automatic FULL/DIFF/LOG, retained for **point-in-time restore** out to (configurable) several weeks; **Long-Term Retention** for compliance-grade multi-year retention. You don't manage them; you initiate restore from the portal / API.
- **Active geo-replication** — DB-level, async readable secondary in another region. Manual failover, single DB.
- **Auto-failover groups** — group-level (one or more DBs as a unit) with a stable listener endpoint. Manual or automatic failover, the listener swings to the new primary so the app's connection string stays the same. This is what you use for app-level DR.

### What's not on Azure SQL Database (vs the box product)

The list shrinks every year, but the perennial gaps:

- **No SQL Agent** on Azure SQL Database (but you have it on Managed Instance and on VM). For DB you use Elastic Jobs, Azure Automation, Logic Apps, or run it from outside.
- **No CLR** — SQL CLR assemblies aren't supported. MI does support it.
- **No FILESTREAM / FILETABLE.** MI doesn't either. Use blob storage instead.
- **No cross-database queries** on Azure SQL Database (MI yes). Single-DB is the unit.
- **No `xp_cmdshell` / no host-OS access.** Goes without saying.
- **No native backup-to-URL / RESTORE FROM URL** semantics on Azure SQL Database — backups are managed, restore is a different API. MI does support backup-to-URL.

---

## 7. Security basics

### Authentication

- **SQL authentication** — username/password stored in the engine. Works everywhere. Avoid for human users; defensible for service principals when nothing better is available.
- **Windows authentication** — domain-integrated; box product / SQL on VM with a domain.
- **Microsoft Entra ID authentication** — the default for humans on Azure SQL. Supports **Entra-only** mode (disable SQL auth entirely) for "no shared passwords ever." Apps should use **Managed Identity** so there's no secret to leak.

```sql
-- Create a contained Entra user mapped to a group; permissions, no password to manage.
CREATE USER [GroupName] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [GroupName];
```

### Roles and permissions

- **Server-level**: fixed server roles (`sysadmin`, `securityadmin`, etc.) and user-defined server roles. Almost everyone over-grants `sysadmin` and pays for it later.
- **Database-level**: fixed database roles (`db_owner`, `db_datareader`, `db_datawriter`, `db_ddladmin`, …) and user-defined database roles.
- **Principle of least privilege** is the only sane stance. Apps connect as a database role with `SELECT/INSERT/UPDATE/DELETE` on specific schemas, never `db_owner`. Schema-scoped permissions are the easiest way to keep this sane (`GRANT SELECT ON SCHEMA::[reporting] TO [reader_role]`).

### Row-Level Security (RLS)

A security predicate function attached to a table that the engine applies on every read (and optionally every write). The function returns a row that says "this principal can see this row." Used for multi-tenant tables where every tenant shares the schema:

```sql
CREATE FUNCTION dbo.fn_TenantPredicate(@TenantId INT)
    RETURNS TABLE WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS res WHERE @TenantId = CAST(SESSION_CONTEXT(N'tenant_id') AS INT);

CREATE SECURITY POLICY dbo.TenantFilter
    ADD FILTER PREDICATE dbo.fn_TenantPredicate(TenantId) ON dbo.Orders
WITH (STATE = ON);
```

The app sets `SESSION_CONTEXT` at connection start; the engine enforces it. The catch: a sysadmin or `db_owner` bypasses RLS unless you're careful with the predicate definition.

### Always Encrypted

**Column-level encryption where the engine never has the keys.** The client driver encrypts on the way in and decrypts on the way out using a key stored outside the engine (Key Vault, Windows certificate store). Two flavors:

- **Deterministic** — same plaintext always produces same ciphertext. Allows equality lookups; weakens secrecy somewhat (frequency analysis).
- **Randomized** — different ciphertext every time. Strongest; can't be searched at all (without secure enclaves).
- **Secure enclaves** extend Always Encrypted to support range queries on randomized columns by performing comparisons inside an attested trusted execution environment.

Use case: PII / payment data where even DBAs must not be able to read plaintext. Trade-off: query patterns are constrained; performance is lower; key management is now your problem.

### TDE (Transparent Data Encryption)

Encrypts the data files, log files, and backups **at rest, on disk**. Transparent to applications — the engine encrypts/decrypts pages as they move between disk and memory. **What it does not protect against**: an attacker who reads memory, an attacker on the network without TLS, or a sysadmin querying the data. TDE is "stolen disk / stolen backup" protection, full stop. Pair with Always Encrypted if you need protection from privileged DBAs.

On Azure SQL, TDE is on by default with a service-managed key; you can switch to a customer-managed key (CMK) backed by Key Vault for "bring your own key."

### Dynamic Data Masking (DDM)

A presentation-layer mask. The data sits in the table unmodified; the engine returns masked output to non-privileged users. **It is not encryption** and not a real security control — anyone who can write a `WHERE` clause can usually unmask it (`WHERE SSN LIKE '123-45-%'`). Use it to keep CSRs from seeing full credit-card numbers in a UI; do not rely on it as a defense against motivated attackers.

### When to use which

| Goal | Use |
|---|---|
| Stop disk-level / backup-level theft | TDE |
| Stop the DBA from reading PII | Always Encrypted (randomized; with enclaves if you need range queries) |
| Multi-tenant table, isolate tenants | Row-Level Security |
| Keep partial data out of UI for low-trust users | Dynamic Data Masking |
| Replace passwords with identity | Entra auth + Managed Identity |

---

## 8. Operational topics

### Backup / restore strategy

- **FULL** — entire database. Baseline of any restore chain.
- **DIFFERENTIAL** — changes since last FULL. Restore order: latest FULL + latest DIFF.
- **LOG** — every transaction since the last LOG (or DIFF/FULL when in `FULL` recovery model). Required for **point-in-time recovery**.

The recovery model determines what's possible:

- **SIMPLE** — log auto-truncates; no log backups; no PITR.
- **FULL** — log grows until backed up; PITR available.
- **BULK_LOGGED** — like FULL but minimal logging for some bulk operations; trade-off on PITR coverage.

A typical OLTP shop on box runs FULL recovery, with weekly FULL + daily DIFF + log backups every 5–15 minutes. The restore chain: latest FULL → latest DIFF → all subsequent LOG backups in order, with `STOPAT = '...'` for point-in-time.

On Azure SQL all of this is automatic — you initiate **point-in-time restore** to any second within your retention window, and the platform handles the chain.

### Maintenance — index and statistics

Two separate jobs that get conflated:

- **Index reorganize / rebuild** — handles physical fragmentation. Reorganize is online and incremental; rebuild is a more expensive, fuller operation (online if the edition supports it). Modern guidance: don't rebuild every night. Fragmentation matters less than people think on SSD; what *really* matters is statistics.
- **Statistics update** — refreshes histograms. Auto-update covers most cases. For volatile or skewed columns, schedule manual updates with `FULLSCAN` if the auto-sample is misleading.

The de-facto standard maintenance solution on box SQL Server is **Ola Hallengren's scripts** — every shop uses them. On Azure SQL, automatic tuning and managed maintenance handle most of this. If you're "scheduling nightly DBCC INDEXDEFRAG" in 2026 you're carrying patterns from 2009 forward.

### Monitoring

- **Query Store** is the single most useful feature for ongoing performance work. Turn it on. Top regressed queries, top resource consumers, plan history for any query, force good plans when bad ones get cached.
- **DMVs** for ad-hoc investigation: `sys.dm_exec_query_stats`, `sys.dm_db_index_usage_stats` (find unused indexes), `sys.dm_os_wait_stats` (what's the engine waiting on), `sys.dm_exec_requests` (live activity).
- **Extended Events** for repeating but elusive issues (deadlocks, occasional timeouts) — the modern replacement for SQL Trace / Profiler. Profiler is deprecated; don't use it.
- On Azure SQL: **Intelligent Insights** (auto-detected performance regressions with explanations), **automatic tuning** (engine creates/drops indexes and forces last-good plans on regressions), **Defender for SQL** (vulnerability assessment + threat detection).

---

## 9. Trade-offs and things learned the hard way

The kind of practical commentary that signals real production experience.

- **Parameter sniffing is the #1 operational pain point.** A stored proc that's been fast for years suddenly tanks after a server restart — because the first call after restart had unusual parameters and that plan got cached. Fix the index first; then `OPTION (RECOMPILE)` on the offending statement; then a Query Store forced plan; only then reach for `OPTIMIZE FOR UNKNOWN`. Don't sprinkle hints prophylactically.

- **`NOLOCK` (a.k.a. `READ UNCOMMITTED`) is not a "go faster" knob.** It allows dirty reads, but worse, on an active table it can return a row twice or skip a row entirely as page splits move data around mid-scan. Reports that need to be "approximately right and definitely not blocking" are the only legitimate use. The right answer 95% of the time is to enable RCSI database-wide.

- **GUID clustered keys fragment the B-tree on every insert.** If you must use a GUID PK, either (a) use `NEWSEQUENTIALID()` so values are monotonically increasing, or (b) cluster on a separate `IDENTITY` surrogate and keep the GUID as a nonclustered unique key. "Random GUID is the clustered key" is one of the top-five reasons a SQL Server schema rots over time.

- **`SELECT *` in views and ORMs hides covering-index opportunities.** EF Core's "select the entire entity by default" behavior is convenient and forecloses the option of tight covering indexes for hot read paths. Project explicitly when the query matters.

- **`MERGE` has had correctness bugs under concurrency for years.** Many shops just don't use it. If you do, take `WITH (HOLDLOCK)` on the target. Or write idempotent `INSERT ... WHERE NOT EXISTS` plus a separate `UPDATE`.

- **Untested backups are not backups.** Schedule periodic restore drills. The first time you find out your backup chain is broken should never be during an outage.

- **TDE is not "data is encrypted everywhere."** It protects data at rest, on disk. In memory? Decrypted. In motion? Whatever your TLS posture is. Against a privileged DBA reading the table? Useless. Pair with Always Encrypted when the threat model requires it.

- **"Just trust the green missing-index hint" is a trap.** SSMS suggests indexes based on this single plan with no awareness of write cost, existing-index overlap, or whether you already have eight indexes that almost cover the same query. Validate with `sys.dm_db_index_usage_stats` and `sys.dm_db_index_operational_stats`.

- **Implicit `nvarchar`-to-`varchar` conversions silently scan indexes.** ORMs default to `NVARCHAR` parameters; if your column is `VARCHAR`, every query does a row-by-row conversion and the seek becomes a scan. Either type your parameters to match the column or change the column. This single issue accounts for a lot of "it's slow in production but fine in my LINQPad query" stories.

- **Passwords in App Service connection strings are a smell.** Use **Managed Identity** + Entra-authenticated database users. There's no secret to rotate, leak, or commit.

- **One Hyperscale database used as a "dump everything" data warehouse misses the point of columnstore + a real analytics platform.** Hyperscale is excellent OLTP that scales to large data; it is not a replacement for Synapse / Fabric for heavy analytical queries. Use the right tool.

---

## 10. Likely interview questions

Concise, strong answers — say enough to demonstrate depth, then stop.

**Q: What's the difference between a clustered and a nonclustered index?**
A: A clustered index *is* the table — the leaf level holds the actual rows in clustering-key order, so there's only one per table. A nonclustered index is a separate B-tree sorted on its own keys; its leaf level holds the clustering key (or RID for a heap), which the engine uses to look up the row when a needed column isn't in the index. Two implications: every nonclustered index implicitly carries the clustering key, so a wide clustering key bloats every index; and a nonclustered index that covers a query (key columns plus `INCLUDE`d columns) avoids the lookup entirely.

**Q: Walk me through what happens when SQL Server runs a query.**
A: Parse (syntax check), bind (resolve names against the catalog), optimize (the optimizer enumerates plan candidates, costs them with statistics, and picks the cheapest within a time budget — caching the result keyed by the statement's hash), and execute. Subsequent calls reuse the cached plan; this is the source of parameter sniffing, where the plan was compiled for atypical parameter values and is wrong for typical ones. The engine reads pages into the buffer pool, takes locks per the isolation level, writes changes to the log first (write-ahead logging), and only later writes the dirty pages to data files at checkpoint.

**Q: What is parameter sniffing and how do you fix it?**
A: SQL Server compiles a plan based on the *first* parameter values it sees and caches it. If those first values were atypical — a tiny tenant when the average is huge, or vice versa — the cached plan is wrong for everyone else, and you get the symptom "the same proc is fast for me and slow for them." Fix order: (1) make sure the index is right, because a great index gives a single robust plan; (2) force the good plan via Query Store; (3) `OPTION (RECOMPILE)` on the offending statement; (4) `OPTIMIZE FOR @p = literal` to pin to a representative value. `OPTIMIZE FOR UNKNOWN` is a last resort and usually worse than the alternatives.

**Q: Difference between READ COMMITTED SNAPSHOT and SNAPSHOT isolation?**
A: Both eliminate reader-writer blocking by reading row versions from tempdb instead of taking shared locks. RCSI is statement-scoped — each statement sees a consistent snapshot of *its own start* — and is set once at the database level as a drop-in replacement for `READ COMMITTED`. SNAPSHOT is transaction-scoped — the entire transaction sees a snapshot of *its* start — and you opt in per session. SNAPSHOT can throw "update conflict" (error 3960) when two transactions touch the same row; RCSI cannot. Most OLTP workloads enable RCSI as the default and reserve SNAPSHOT for specific reporting transactions that need the strict guarantee.

**Q: How do you investigate a slow query in production?**
A: Wait stats first — what's the engine actually waiting on (CPU, IO, locks, latches)? Then Query Store: top resource consumers and recent plan regressions. Then `sys.dm_exec_query_stats` for the specific cached statement and `sys.dm_exec_requests` if it's running right now. Then capture the actual plan with `SET STATISTICS IO, TIME ON` and look for fat arrows, big estimated-vs-actual mismatches, key lookups, and unexpected scans. Only then start hypothesizing — usually it's a missing or wrong index, a non-SARGable predicate (function on a column, leading-wildcard `LIKE`, implicit conversion), or stale statistics that gave the optimizer bad row estimates.

**Q: How do deadlocks happen and how do you mitigate them?**
A: Two transactions, opposite lock order: T1 locks A then asks for B; T2 locks B then asks for A. Engine detects the cycle, picks a victim, rolls it back with error 1205. Mitigations in priority order: enforce consistent access order across the codebase; shorten transactions (don't open one and then call out to a service); enable RCSI to eliminate reader-writer deadlocks; have the application catch error 1205 and retry. Capture the deadlock graph via Extended Events to know which two statements actually collided, because the answer is almost never obvious from the error message.

**Q: When would you denormalize?**
A: When read patterns are stable and frequent enough that the cost of joining live every time exceeds the cost of maintaining a redundant copy. Common cases: a counter or aggregate that's read on every page load (denormalize and update on write or via a trigger / async job); a read model in a separate table maintained by a background process; flattened reporting tables. The trade-off is always update anomalies and data staleness — you're trading write complexity for read speed. Don't denormalize until you have evidence; you almost certainly know less about your access patterns than you think.

**Q: Azure SQL Database vs Managed Instance vs SQL on a VM — how do you choose?**
A: Start at the right end. **VM** when you need something the platform doesn't allow — CLR, FILESTREAM, host-OS access, a specific OS / SQL build, or strict vendor support requirements. **Managed Instance** for lift-and-shift from on-prem when you have multi-DB queries, SQL Agent jobs, linked servers, or a fairly heavy SQL Server feature surface — MI is near-100% feature parity. **Azure SQL Database** for new apps, single-database scope, when you want minimum operational overhead. The deciding factors are operational responsibility (more managed = less surface area you control), feature surface area, and networking (MI is VNet-injected by default; Database is public endpoint or private endpoint).

**Q: What does TDE actually protect against?**
A: Data at rest, on disk. Stolen drives, stolen backup tapes, somebody copying the .mdf file off the server. That's it. TDE does *not* protect data in memory (decrypted), data in motion (whatever your TLS does), or data from a privileged DBA who can `SELECT *`. People often install TDE and call the data "encrypted," which is misleading. For protection from privileged users, layer **Always Encrypted** on the sensitive columns so the keys never leave the client.

**Q: Tell me about a `MERGE` statement.**
A: It's syntactically tidy — one statement that does INSERT, UPDATE, and DELETE based on a join condition — but it has a long history of correctness issues under concurrency: lost updates, primary-key violations from concurrent inserts, certain trigger interactions. The defensive answer is "we use `WITH (HOLDLOCK)` on the target if we use it at all," and many production teams just don't use it — they write separate idempotent `UPDATE` and `INSERT ... WHERE NOT EXISTS` statements instead. Knowing the syntax is fine; knowing why senior practitioners are wary of it is what an interviewer is checking for.

---

## 11. Further study

Cross-links to deeper notes elsewhere in this repo, plus topic folders to fill out as needed:

- [`../../DOTNET/C#/TOPICS/HEALTH_AND_PERFORMANCE/sql-index-analysis.md`](../../DOTNET/C%23/TOPICS/HEALTH_AND_PERFORMANCE/sql-index-analysis.md) — practitioner-level index analysis from the .NET notes.
- [`../../DOTNET/C#/TOPICS/HEALTH_AND_PERFORMANCE/sql-query-plan-analysis.md`](../../DOTNET/C%23/TOPICS/HEALTH_AND_PERFORMANCE/sql-query-plan-analysis.md) — reading execution plans, from the .NET notes.
- [`../../AZURE/ENTRAID/04-managed-identities.md`](../../AZURE/ENTRAID/04-managed-identities.md) — Managed Identity for app-to-Azure-SQL auth (don't restate; cross-link).
- [`../../AZURE/AZ_CLI/12-databases-commands.md`](../../AZURE/AZ_CLI/12-databases-commands.md) — `az sql` reference for the command-line side.

Topic sub-folders to fill in lazily as deeper notes get written: `T_SQL/`, `INDEXING/`, `PERFORMANCE/`, `CONCURRENCY/`, `HA_DR/`, `SECURITY/`, `AZURE_SQL/`. One concept per file when each gets created — this overview stays as the single document.
