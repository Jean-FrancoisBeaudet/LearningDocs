# Bulk operations

_Targets .NET 10 / C# 14. See also: [Batch processing](./batch-processing.md), [Relational database query optimization](./relational-database-query-optimization.md), [Document database query optimization](./document-database-query-optimization.md), [Connection pooling](./connection-pooling.md), [GC pressure](./gc-pressure.md), [Aggregation efficiency](./aggregation-efficiency.md), [SQL index analysis](./sql-index-analysis.md)._

**Bulk operations** are *downstream* APIs that accept N records in one call instead of N round-trips. The win is multiplicative: round-trip latency × N collapses to 1 × RTT plus linear server-side processing. For 100k rows on a 0.5 ms-RTT VPC link, that's 50 seconds → ~1.5 seconds — same network, same disk, same CPU, just a different API.

> If your code pattern is `foreach (var x in items) await db.Save(x)`, you've written an N-RTT loop. The fix is a bulk API on the database side, not more concurrency on the client side. Concurrency hides the cost; bulk eliminates it.

This pairs with [batch processing](./batch-processing.md): a batcher *assembles* N items in-process, a bulk operation *ships* them in one call. Use both together.

## What "bulk" means in each store

| Store | Bulk insert | Bulk upsert | Bulk update / delete |
|---|---|---|---|
| SQL Server | `SqlBulkCopy` (minimally-logged) | TVP + `MERGE` | TVP + `UPDATE FROM` / `DELETE` join |
| PostgreSQL | `COPY` via `Begin{Binary,Text}Import` | `INSERT … ON CONFLICT DO UPDATE` (with `unnest`) | `UPDATE … FROM unnest(...)` |
| EF Core 7+ | `AddRange` + `SaveChanges` (still per-row INSERT, batched by `MAX_BATCH_SIZE`) | `ExecuteUpdateAsync` (set-based) | `ExecuteUpdateAsync`, `ExecuteDeleteAsync` |
| MongoDB | `InsertManyAsync(ordered: false)` | `BulkWriteAsync` (mixed ops) | `UpdateManyAsync` |
| RavenDB | `BulkInsert` | `Patch` + `LoadAsync` batches | `PatchByQueryOperation` |
| Azure Service Bus | `SendMessagesAsync(IEnumerable<...>)`, `ServiceBusMessageBatch` | n/a | n/a |
| Event Hubs | `EventDataBatch` | n/a | n/a |
| Cosmos DB | `TransactionalBatch` (per partition key) | `TransactionalBatch` with `Upsert` | `TransactionalBatch` `Replace`/`Delete` |

`SqlBulkCopy` is the only true "wire-protocol bulk" path on this list — it bypasses INSERT statement parsing entirely, talks the BCP protocol, and on a heap with `TABLOCK + BULK_LOGGED` recovery model is the closest you get to raw page writes from .NET.

## SQL Server with `SqlBulkCopy`

```csharp
public async Task LoadAsync(IAsyncEnumerable<Order> orders, CancellationToken ct)
{
    await using var conn = new SqlConnection(_connStr);
    await conn.OpenAsync(ct);
    await using var tx = (SqlTransaction)await conn.BeginTransactionAsync(ct);

    using var bulk = new SqlBulkCopy(conn, SqlBulkCopyOptions.TableLock, tx)
    {
        DestinationTableName = "dbo.Orders_Staging",
        BatchSize = 5_000,                  // server-side commit chunk
        BulkCopyTimeout = 60,
        EnableStreaming = true,
        NotifyAfter = 10_000,
    };
    bulk.SqlRowsCopied += (_, e) => _logger.LogDebug("Copied {Rows}", e.RowsCopied);
    bulk.ColumnMappings.Add(nameof(Order.Id), "Id");
    bulk.ColumnMappings.Add(nameof(Order.CustomerId), "CustomerId");
    bulk.ColumnMappings.Add(nameof(Order.Total), "Total");
    bulk.ColumnMappings.Add(nameof(Order.CreatedAt), "CreatedAt");

    await bulk.WriteToServerAsync(ToDataReader(orders), ct);

    // Now MERGE staging → target with one set-based statement.
    await using var merge = new SqlCommand(@"
        MERGE dbo.Orders AS t
        USING dbo.Orders_Staging AS s ON t.Id = s.Id
        WHEN MATCHED THEN UPDATE SET CustomerId = s.CustomerId, Total = s.Total
        WHEN NOT MATCHED THEN INSERT (Id, CustomerId, Total, CreatedAt)
                          VALUES (s.Id, s.CustomerId, s.Total, s.CreatedAt);
        TRUNCATE TABLE dbo.Orders_Staging;", conn, tx) { CommandTimeout = 60 };
    await merge.ExecuteNonQueryAsync(ct);

    await tx.CommitAsync(ct);
}
```

The pattern: **bulk-load to a staging heap → set-based MERGE → truncate**. Loading directly into the target with constraints/indexes/triggers active drops bulk throughput by 5–10× because every row hits the index trees.

`EnableStreaming = true` matters when the source is `IDataReader` over `IAsyncEnumerable` — without it, `SqlBulkCopy` materializes the whole reader before sending. With it, rows flow through the wire as the reader yields.

## Table-valued parameters (TVPs)

Best for sub-10k-row operations that need a transactional UPSERT alongside other DML in the same call. Cheaper to set up than `SqlBulkCopy` (no staging table), but slower for huge sets.

```csharp
var dt = new DataTable();
dt.Columns.Add("Id", typeof(Guid));
dt.Columns.Add("Total", typeof(decimal));
foreach (var o in orders) dt.Rows.Add(o.Id, o.Total);

await using var cmd = new SqlCommand("dbo.UpsertOrders", conn) { CommandType = CommandType.StoredProcedure };
var p = cmd.Parameters.AddWithValue("@orders", dt);
p.SqlDbType = SqlDbType.Structured;
p.TypeName = "dbo.OrderTvp";
await cmd.ExecuteNonQueryAsync(ct);
```

The stored procedure does the MERGE server-side. TVPs are read-only and cannot be reused across commands without re-binding.

## PostgreSQL with `COPY`

`COPY` is to PostgreSQL what `SqlBulkCopy` is to SQL Server, but cleaner: it's a first-class SQL statement, not a side-channel protocol.

```csharp
await using var conn = new NpgsqlConnection(_connStr);
await conn.OpenAsync(ct);

await using (var writer = await conn.BeginBinaryImportAsync(
    "COPY orders (id, customer_id, total, created_at) FROM STDIN (FORMAT BINARY)", ct))
{
    await foreach (var o in orders.WithCancellation(ct))
    {
        await writer.StartRowAsync(ct);
        await writer.WriteAsync(o.Id, NpgsqlDbType.Uuid, ct);
        await writer.WriteAsync(o.CustomerId, NpgsqlDbType.Uuid, ct);
        await writer.WriteAsync(o.Total, NpgsqlDbType.Numeric, ct);
        await writer.WriteAsync(o.CreatedAt, NpgsqlDbType.TimestampTz, ct);
    }
    await writer.CompleteAsync(ct);
}
```

For upsert, `COPY` into a temp table, then `INSERT INTO orders SELECT ... FROM tmp ON CONFLICT (id) DO UPDATE`. The `unnest` pattern works for sub-1k-row sets and avoids the staging table:

```sql
INSERT INTO orders (id, customer_id, total)
SELECT * FROM unnest(@ids::uuid[], @customers::uuid[], @totals::numeric[])
ON CONFLICT (id) DO UPDATE SET customer_id = EXCLUDED.customer_id, total = EXCLUDED.total;
```

Pass three parallel arrays from .NET (`Guid[]`, `Guid[]`, `decimal[]`); Npgsql binds them as `uuid[]` / `numeric[]`.

## EF Core: what's bulk and what isn't

```csharp
db.Orders.AddRange(orders);
await db.SaveChangesAsync(ct);
```

This is **not** bulk. EF Core sends `INSERT` statements, batched up to `MAX_BATCH_SIZE` (default 42 for SQL Server, 1000 for PostgreSQL) per round-trip, with the change tracker firing for every row. For 100k rows, expect 2400 round-trips on SQL Server and a memory cost from the change tracker that often exceeds the data itself.

The set-based methods *are* bulk-equivalent for update/delete:

```csharp
await db.Orders.Where(o => o.CreatedAt < cutoff)
              .ExecuteDeleteAsync(ct);                       // single DELETE WHERE

await db.Orders.Where(o => o.Status == OrderStatus.Pending)
              .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Cancelled), ct);
```

These compile to one server-side `DELETE` / `UPDATE`, no change tracker, no entity materialization. EF Core 7+ only.

For inserts beyond what `SaveChangesAsync` can handle: use `EFCore.BulkExtensions` or `Z.EntityFramework.Extensions.EFCore`. `BulkInsertAsync` on the former wraps `SqlBulkCopy` with the entity mapping; the trade-off is a third-party dependency and skipping the change tracker, which means navigation properties aren't populated and triggers may not fire.

## Mongo, Service Bus, Event Hubs, Cosmos

```csharp
// MongoDB — unordered = parallel server-side, continues on per-doc errors.
await collection.InsertManyAsync(docs, new InsertManyOptions { IsOrdered = false }, ct);

// Service Bus — batch packing respects 256 KB ceiling.
ServiceBusMessageBatch batch = await sender.CreateMessageBatchAsync(ct);
foreach (var msg in messages)
{
    if (!batch.TryAddMessage(new ServiceBusMessage(msg))) { /* batch full → send + new batch */ }
}
await sender.SendMessagesAsync(batch, ct);

// Event Hubs — same pattern, different size cap (1 MB or partition-tier limit).
EventDataBatch evBatch = await producer.CreateBatchAsync(ct);

// Cosmos DB — TransactionalBatch is per-partition-key only.
TransactionalBatch tb = container.CreateTransactionalBatch(new PartitionKey(customerId));
foreach (var o in ordersForThisCustomer) tb.UpsertItem(o);
TransactionalBatchResponse resp = await tb.ExecuteAsync(ct);
```

Cosmos's `TransactionalBatch` is the only ACID bulk on this list (within one partition key). Service Bus and Event Hubs batch for *throughput* — you still get at-least-once, not atomic.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `foreach (var x in items) await db.Add(x); await db.SaveChangesAsync()` per-iteration | N round-trips, N transactions, ~50× slower than `AddRange` + one `SaveChangesAsync` |
| `SqlBulkCopy` straight into the target table with all indexes & FKs active | 5–10× slower than staging-then-MERGE |
| `BatchSize = 0` on `SqlBulkCopy` (the default = "all in one batch") | Whole load is one transaction; one error rolls back hours; log file fills |
| `IsOrdered = true` on `InsertManyAsync` for huge sets | First doc error stops the whole insert; no per-doc parallelism |
| `AddRange` of 100k entities then `SaveChangesAsync` | Change tracker memory blows up — often more memory than the data itself |
| TVP for >50k rows | Slower than `SqlBulkCopy` because it's still a single command parameter parse |
| Bulk insert inside a foreach loop over batches without holding one connection | Re-acquire connection per batch — see [connection pooling](./connection-pooling.md) |
| Ignoring Service Bus 256 KB batch cap | `MessageSizeExceededException`; must call `TryAddMessage` and check the bool |
| `ExecuteUpdateAsync` on entities you've already loaded into the change tracker | Stale entities in the tracker; subsequent `SaveChangesAsync` overwrites the bulk update |
| Bulk insert that fires application-side triggers via change-tracker events | The triggers don't fire; downstream consumers miss the rows |

## Worked example: 100k inserts with retry on transient

```csharp
public async Task LoadOrdersAsync(IAsyncEnumerable<Order> orders, CancellationToken ct)
{
    var pipeline = new ResiliencePipelineBuilder()
        .AddRetry(new RetryStrategyOptions
        {
            ShouldHandle = new PredicateBuilder().Handle<SqlException>(IsTransient),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
        })
        .Build();

    await pipeline.ExecuteAsync(async token =>
    {
        await using var conn = new SqlConnection(_connStr);
        await conn.OpenAsync(token);
        await using var tx = (SqlTransaction)await conn.BeginTransactionAsync(token);

        using var bulk = new SqlBulkCopy(conn, SqlBulkCopyOptions.TableLock, tx)
        {
            DestinationTableName = "dbo.Orders_Staging",
            BatchSize = 5_000,
            EnableStreaming = true,
            BulkCopyTimeout = 60,
        };
        await bulk.WriteToServerAsync(ToDataReader(orders), token);

        await using var merge = new SqlCommand(MergeSql, conn, tx) { CommandTimeout = 60 };
        await merge.ExecuteNonQueryAsync(token);

        await tx.CommitAsync(token);
    }, ct);
}

static bool IsTransient(SqlException ex) =>
    ex.Number is 4060 or 40197 or 40501 or 40613 or 49918 or 49919 or 49920 or 11001 or -2;
```

A 100k-row load on a 4-core Azure SQL S2 with this pattern: ~3 seconds end-to-end, single transaction, retry on Azure SQL transient throttling.

## Locking & concurrency during bulk

`SqlBulkCopyOptions.TableLock` takes an X lock on the target — fast, but kills concurrent reads. Without it, the engine starts with row locks and *escalates* to table when it crosses ~5000 locks; you get the same end state but with lock-escalation overhead and more chance of deadlock with concurrent writers.

For a target that has live read traffic:

- Bulk-load into a staging table (no concurrency contention).
- Use `MERGE` with read-committed snapshot isolation (RCSI) on the database — readers don't block, writers don't either.
- Or: bulk-load to a side table, then `ALTER TABLE … SWITCH PARTITION` for instant cutover.

PostgreSQL's `COPY` takes a `RowExclusiveLock`, compatible with concurrent readers but not concurrent `COPY` to the same table. For massive loads, partition the table and load partitions in parallel.

## Senior-level gotchas

- **`SqlBulkCopy` skips triggers and check constraints by default.** That's a feature for throughput and a footgun for correctness — if your `INSERT` trigger writes audit rows, bulk inserts won't. Use `SqlBulkCopyOptions.FireTriggers` and accept the cost, or replicate the trigger logic in the post-load `MERGE`.
- **`SqlBulkCopy` does not check identity column constraints unless `KeepIdentity` is set.** Without it, the server generates IDs; with it, you supply them and `IDENTITY_INSERT` is enabled implicitly. Mixing is undefined.
- **Bulk-load throughput is bounded by the transaction log under FULL recovery.** Switch the database to BULK_LOGGED for the load window if you can — the log records minimal information for `SqlBulkCopy` writes and throughput jumps 3–5×. Switch back after.
- **`BatchSize` is a *commit chunk*, not a network chunk.** With `BatchSize = 5000` and 100k rows, the bulk copy commits 20 times. A failure mid-load partially commits — design idempotency accordingly.
- **EF Core's `MAX_BATCH_SIZE` is per-`SaveChangesAsync`, not global.** Lowering it reduces per-batch memory but adds round-trips. Default (42 on SQL Server) is calibrated against TDS packet size; raising it past 100 rarely helps because the packet fragments anyway.
- **`AddRange` populates the change tracker.** For 100k entities, that's 100k tracked entries, ~500 MB of working set, and an `O(N)` scan on every property change. Disable change tracking for pure inserts: `db.ChangeTracker.AutoDetectChangesEnabled = false; … db.SaveChangesAsync(); db.ChangeTracker.Clear();`.
- **`unnest` array binding in Npgsql treats `null` array element as SQL NULL, but a null array as zero rows.** A bug-class waiting to happen if your producer occasionally sends an empty list — check before binding.
- **`InsertManyAsync(ordered: false)` returns first error wrapped in `MongoBulkWriteException` even if 99% of writes succeeded.** Inspect `WriteErrors` to know what failed; don't treat the exception as "nothing was written."
- **Cosmos `TransactionalBatch` is partition-key scoped and capped at 100 ops or 2 MB.** Cross-partition transactions don't exist in Cosmos; if your bulk crosses partitions, you're back to per-partition batches with no atomicity.
- **Service Bus `TryAddMessage` returning `false` is the only signal you've hit the 256 KB cap.** Code that ignores the bool silently drops the message; always loop the pattern of "try → if false, send current batch, start new batch, add."
- **`SqlBulkCopy` deadlocks differently than DML.** It takes locks the engine doesn't expose to `sp_who2` cleanly; the deadlock graph in `system_health` is the source of truth. Don't expect typical deadlock retry semantics — most are not retryable without first re-staging the data.
- **Bulk operations don't compose with EF Core change tracker.** `await db.SaveChangesAsync()` after a `_bulkExtensions.BulkInsertAsync` won't see the inserted rows unless you clear and reload. Treat bulk as a side-channel; don't mix with tracked entities in the same `DbContext` scope.
- **Connection acquisition is sometimes the slowest step.** A bulk that takes 800 ms with a fresh pool can take 50 ms once the pool is warm — see [connection pooling](./connection-pooling.md). Pre-warm the pool at startup if first-request latency matters.
- **`DataTable` is allocation-heavy.** For huge TVPs, use `IEnumerable<SqlDataRecord>` instead — it streams rows without materializing the whole structure into a `DataTable`.
- **`COPY` errors stop the whole import.** PostgreSQL has no equivalent of `ordered: false`; one bad row aborts. Use `COPY ... WITH (ON_ERROR ignore)` (PG 17+) or stage to a `text`-typed table and validate in SQL.
