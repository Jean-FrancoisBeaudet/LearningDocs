# Dapper

_Targets .NET 10 / C# 14 with Dapper v2.1+. See also: [Controllers](../ASPNET_CORE_BASICS/controllers.md), [Minimal APIs](../ASPNET_CORE_BASICS/minimal-apis.md), [HybridCache](../CACHING/hybrid-cache.md), [GraphQL HotChocolate](../API_COMMUNICATION/graphql-hotchocolate.md)._

Dapper is a **micro-ORM**: extension methods on `IDbConnection` that materialize SQL results into POCOs. There's no `DbContext`, no change tracking, no migrations, no expression-tree translation — you write SQL, Dapper handles parameterization and column-to-property binding. Pick it when EF Core's tracking and translation are overhead (read-heavy reporting endpoints, hand-tuned queries, stored procedures, hot paths where the EF query plan is wrong). Skip it when you need migrations, navigation properties, or when juniors are writing data access — raw SQL has fewer guardrails.

## Connection management

Dapper extends `IDbConnection`, but you provide the connection. Use a factory and `await using`:

```csharp
public interface ISqlConnectionFactory
{
    Task<DbConnection> OpenAsync(CancellationToken ct);
}

public sealed class SqlConnectionFactory(IConfiguration cfg) : ISqlConnectionFactory
{
    private readonly string _cs = cfg.GetConnectionString("Db")!;

    public async Task<DbConnection> OpenAsync(CancellationToken ct)
    {
        var conn = new SqlConnection(_cs);                 // or NpgsqlConnection, MySqlConnector, etc.
        await conn.OpenAsync(ct);
        return conn;
    }
}

builder.Services.AddSingleton<ISqlConnectionFactory, SqlConnectionFactory>();
```

`SqlConnection` is cheap; the .NET pool handles the actual TCP reuse. Don't cache an open connection across requests — open per call, dispose at the end of the scope.

## Core APIs

```csharp
public sealed class OrderRepo(ISqlConnectionFactory factory)
{
    public async Task<Order?> FindAsync(long id, CancellationToken ct)
    {
        await using var conn = await factory.OpenAsync(ct);
        return await conn.QueryFirstOrDefaultAsync<Order>(
            new CommandDefinition(
                "SELECT Id, Sku, Total, CreatedAt FROM Orders WHERE Id = @id",
                new { id },
                cancellationToken: ct));
    }

    public async Task<IReadOnlyList<Order>> RecentAsync(int take, CancellationToken ct)
    {
        await using var conn = await factory.OpenAsync(ct);
        var rows = await conn.QueryAsync<Order>(
            new CommandDefinition(
                "SELECT TOP (@take) Id, Sku, Total, CreatedAt FROM Orders ORDER BY CreatedAt DESC",
                new { take },
                cancellationToken: ct));
        return rows.AsList();
    }

    public async Task<int> MarkShippedAsync(long id, DateTime when, CancellationToken ct)
    {
        await using var conn = await factory.OpenAsync(ct);
        return await conn.ExecuteAsync(
            new CommandDefinition(
                "UPDATE Orders SET ShippedAt = @when WHERE Id = @id AND ShippedAt IS NULL",
                new { id, when },
                cancellationToken: ct));
    }
}
```

**Always** parameterize. Anonymous-object properties become `@name` parameters. Never interpolate user input into the SQL string — Dapper will not save you from SQL injection because that's not its job.

## DynamicParameters when anonymous objects aren't enough

```csharp
var p = new DynamicParameters();
p.Add("status", status);
p.Add("ids", ids);                                  // list — provider expands automatically
p.Add("total", dbType: DbType.Decimal, direction: ParameterDirection.Output);

await conn.ExecuteAsync("dbo.UpdateAndReturnTotal", p, commandType: CommandType.StoredProcedure);

var newTotal = p.Get<decimal>("total");
```

`DynamicParameters` is also where you handle output parameters, table-valued parameters, and explicit `DbType` (see the AnsiString gotcha below).

## Multi-mapping

When a single query joins parent and child rows, supply a `splitOn` and a mapper:

```csharp
const string sql = """
    SELECT  o.Id, o.Sku, o.CustomerId,
            c.Id, c.Name, c.Email
    FROM Orders o JOIN Customers c ON c.Id = o.CustomerId
    WHERE o.CreatedAt >= @since
    """;

var orders = await conn.QueryAsync<Order, Customer, Order>(
    sql,
    map: (o, c) => { o.Customer = c; return o; },
    param: new { since },
    splitOn: "Id");                                  // c.Id starts the second object
```

`splitOn` defaults to `"Id"` — if your join key is `CustomerId` and there's no later column literally named `Id`, the mapper silently sees a `null` second object and you'll spend an afternoon staring at the query plan. Always name the splitting column explicitly.

## QueryMultipleAsync — composite reads in one roundtrip

```csharp
const string sql = """
    SELECT * FROM Orders WHERE Id = @id;
    SELECT * FROM OrderLines WHERE OrderId = @id;
    SELECT TOP 1 * FROM Customers WHERE Id = (SELECT CustomerId FROM Orders WHERE Id = @id);
    """;

await using var grid = await conn.QueryMultipleAsync(new CommandDefinition(sql, new { id }, cancellationToken: ct));
var order    = await grid.ReadFirstOrDefaultAsync<Order>();
var lines    = (await grid.ReadAsync<OrderLine>()).AsList();
var customer = await grid.ReadFirstOrDefaultAsync<Customer>();
```

One TCP roundtrip, one statement plan cache hit per statement, three materialized lists. The right shape for read-model endpoints.

## Streaming large result sets

`QueryAsync` buffers the whole result by default. For exports or large reports, stream:

```csharp
public async IAsyncEnumerable<Order> StreamAllAsync([EnumeratorCancellation] CancellationToken ct)
{
    await using var conn = await factory.OpenAsync(ct);
    var reader = await conn.ExecuteReaderAsync(
        new CommandDefinition("SELECT Id, Sku, Total, CreatedAt FROM Orders", cancellationToken: ct));

    var parser = reader.GetRowParser<Order>();
    while (await reader.ReadAsync(ct))
        yield return parser(reader);
}
```

`GetRowParser<T>` reuses Dapper's compiled row materializer without re-buffering. Pair with `Response.WriteAsJsonAsync` over `IAsyncEnumerable<T>` to stream JSON to the client without ever holding the full result in memory.

## Transactions

```csharp
await using var conn = await factory.OpenAsync(ct);
await using var tx   = await conn.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

try
{
    await conn.ExecuteAsync(new CommandDefinition(
        "INSERT INTO Orders (...) VALUES (...)", new { /* ... */ }, transaction: tx, cancellationToken: ct));

    await conn.ExecuteAsync(new CommandDefinition(
        "INSERT INTO OrderLines (...) VALUES (...)", new { /* ... */ }, transaction: tx, cancellationToken: ct));

    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

Pass `transaction: tx` to **every** command — Dapper does not infer it. Forgetting it on one statement silently runs that statement outside the transaction.

## Type handlers

```csharp
public sealed class JsonDocumentTypeHandler : SqlMapper.TypeHandler<JsonDocument>
{
    public override JsonDocument? Parse(object value)
        => value is string s ? JsonDocument.Parse(s) : null;

    public override void SetValue(IDbDataParameter parameter, JsonDocument? value)
    {
        parameter.DbType = DbType.String;
        parameter.Value  = value?.RootElement.GetRawText() ?? (object)DBNull.Value;
    }
}

SqlMapper.AddTypeHandler(new JsonDocumentTypeHandler());
```

Register once at startup. Use the same pattern for value objects, `DateOnly` on older providers, or domain-specific encodings.

## Sharing the connection with EF Core

When most of an endpoint is EF but one query is hand-tuned SQL, share the connection:

```csharp
var conn = db.Database.GetDbConnection();
await db.Database.OpenConnectionAsync(ct);                  // EF respects already-open

var rows = await conn.QueryAsync<Report>(
    new CommandDefinition("SELECT ... FROM ... WHERE ...", new { /* ... */ }, cancellationToken: ct));
```

If a `DbContext` transaction is in flight, also pass `transaction: db.Database.CurrentTransaction!.GetDbTransaction()`.

## Dapper vs EF Core — the actual decision

| Need | Pick |
|---|---|
| Migrations, change tracking, navigation, model evolution | **EF Core** |
| Hand-written SQL with stored procs / window functions / CTEs | **Dapper** |
| Read-only reporting / read-models / dashboards | **Dapper** (often via `QueryMultipleAsync`) |
| Multi-tenant CRUD with rich graphs | **EF Core** |
| Hot-path query EF is mistranslating | Both — EF for the rest, Dapper for the one query |
| Bulk insert millions of rows | Neither — `SqlBulkCopy` directly, or EFCore.BulkExtensions |

## Senior-level gotchas

- **`AnsiString` vs `String`**: passing a CLR `string` becomes `nvarchar` by default. If the indexed column is `varchar`, SQL Server can't seek the index and silently scans. Use `DynamicParameters` and `Add("name", value, DbType.AnsiString, size: 50)` for `varchar` columns.
- **`splitOn` defaults to `"Id"`** — if the second object's first column isn't literally `Id`, mapping returns nulls with no error.
- **No change tracking** means no `SaveChanges`. Updates are explicit `ExecuteAsync` calls; concurrency control is your responsibility (e.g. `WHERE RowVersion = @rowversion`, check rows-affected).
- **Constructor binding** matches column name to constructor parameter name (case-insensitive). Records work great here, but rename a column without renaming the parameter and the row materializes with defaults.
- **Cancellation** is honored only as far as the underlying provider supports. SqlClient cancels promptly; some legacy providers ignore the token until the current packet returns.
- **MARS** (`MultipleActiveResultSets=true` on SQL Server) lets you nest readers on one connection — useful for streaming + per-row lookups, but it disables connection pool reuse across providers and can mask deadlocks. Default to MARS off; open a second connection when you need to nest.
- **`buffered: false`** with `QueryAsync<T>` keeps the reader open until enumeration finishes. If your loop body does an `await` that takes a connection from the same pool under contention, you can deadlock yourself. Buffer or use a separate connection.
- **`AsList()` vs `ToList()`**: `AsList()` returns the internal `List<T>` Dapper allocated, avoiding a second copy. Cheap win on hot paths.
- **No identity-map**: query the same row twice in one method and you get two different objects. With EF, change-tracker would dedupe. Plan for it when composing aggregates manually.
