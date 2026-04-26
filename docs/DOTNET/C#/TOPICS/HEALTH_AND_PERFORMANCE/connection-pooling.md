# Connection pooling

_Targets .NET 10 / C# 14. See also: [I/O saturation](./io-saturation.md), [Inter-service call optimization](./inter-service-call-optimization.md), [Relational database query optimization](./relational-database-query-optimization.md), [HttpClient and HttpClientFactory](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/httpclient-and-httpclientfactory.md), [dotnet-counters](./dotnet-counters.md)._

**Connection pooling** trades memory for handshake cost: TCP + TLS + auth setup is multi-millisecond and bandwidth-expensive, so most clients keep a small pool of warm connections and hand them out per logical operation. The pool is the most failure-prone piece of infrastructure in a typical service: misconfigured, it either *starves* (callers block waiting for a slot, latency explodes) or *leaks* (connections accumulate until the server-side limit rejects new ones).

> If your service has been "fast in dev, slow in prod" or "fast for an hour, then errors", the pool is the first place to look.

## Pools you actually have

A typical .NET service has at least four connection pools, each with its own knobs and its own failure modes:

| Pool | Owned by | Keyed by | Default size |
|---|---|---|---|
| HTTP outbound | `SocketsHttpHandler` (one per `HttpClient` handler) | Authority (`scheme://host:port`) | `MaxConnectionsPerServer = int.MaxValue` |
| SQL Server / Azure SQL | `Microsoft.Data.SqlClient` | Full connection string (case-sensitive) | 100 |
| PostgreSQL | `Npgsql` | Connection string + `Pooling=true` | 100 |
| EF Core context | `IDbContextFactory<TContext>` (when pooled) | Service registration | 1024 |
| Redis | `StackExchange.Redis` (`ConnectionMultiplexer`) | One multiplexer per "logical Redis" | Multiplexed (single connection, many ops) |
| MongoDB | `MongoClient` | Cluster URI | `MaxConnectionPoolSize = 100` |

The most important property: **two of these are mistaken for the others all the time**.

- `IHttpClientFactory` does *not* pool `HttpClient` — it pools `SocketsHttpHandler`. The `HttpClient` you receive is cheap to allocate; the handler graph behind it is the expensive thing.
- `AddDbContextPool<TContext>` does *not* pool the underlying database connection — ADO.NET already does that. It pools the *`DbContext` instance*: the change tracker, the model, the cached compiled queries. That's a separate problem from connection reuse.

## Symptoms of misconfiguration

| Symptom | Likely cause |
|---|---|
| `Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool` | `Max Pool Size` reached; callers waiting; connections leaked or just under-sized |
| `SocketException: Only one usage of each socket address` | `new HttpClient()` per call, ephemeral port exhaustion |
| Sudden spike of `A network-related or instance-specific error` after exactly N seconds since deploy | Cached DNS in long-lived `HttpClient` without `PooledConnectionLifetime`, downstream IP rotated |
| Server-side `Too many connections` on the database | Pool size × pod count > server's `max_connections` ceiling |
| Latency normal except for the first request after idle | Pool warmed below working size; cold socket + TLS handshake on hot path |
| `Authentication failed` after rotating credentials | Cached connection still authenticated as old principal — depends on driver |
| Connections in `TIME_WAIT` on the server side, low throughput on the client | Client closing connections instead of reusing them |

## The knobs that matter

For each pool, three knobs do most of the work: **max size**, **idle / lifetime**, and **timeout for acquisition**.

### HttpClient / SocketsHttpHandler

```csharp
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    MaxConnectionsPerServer    = 64,                       // bulkhead the upstream
    PooledConnectionLifetime   = TimeSpan.FromMinutes(5),  // DNS rotation
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2), // close idle sockets
    EnableMultipleHttp2Connections = true,                 // for high-rps HTTP/2
})
```

`PooledConnectionLifetime` is the single most-forgotten setting in .NET services. The default is `Timeout.InfiniteTimeSpan` — once a socket is open, it lives until the process exits. After a downstream rolls (Kubernetes pod replaced, Azure App Service slot swap), the cached IP is dead but your handler keeps using it.

### SqlConnection (Microsoft.Data.SqlClient)

```
Server=...;Database=...;
Pooling=True;
Min Pool Size=5;
Max Pool Size=100;
Connection Lifetime=300;       // forces close after 5 min — DNS / cluster failover
Connect Timeout=15;             // give up acquiring/opening
```

- **`Pooling=False` is almost always wrong.** The cases it's right: a one-shot tool with a single connection; an explicit decision because you're hitting a quirky middleware that misbehaves with reused connections.
- **`Connection Lifetime` is misnamed** — it's actually "max age before pool refuses to hand it out again", not "kill the open connection now". Sockets live until they idle out or the lifetime expires *between uses*.
- **Different connection strings = different pools.** `Database=Foo` and `Database=foo` (case difference) are two pools. So are strings differing only by whitespace, capitalization of `Server=`, or the order of keys.

### EF Core (`AddDbContextPool`)

```csharp
builder.Services.AddDbContextPool<AppDbContext>(opts =>
    opts.UseSqlServer(connStr), poolSize: 1024);
```

What it pools: the `DbContext` instance — the change tracker reset, the cached model. What it does *not* pool: the database connection (ADO.NET does that).

Restrictions:

- **No mutable state in `OnConfiguring`** — it runs once per pooled instance, not per scope.
- **No constructor parameters that vary per request** — the pool reuses instances across requests.
- **Reset hook**: implement `IResettableService` if your context holds reset-able state.
- **Don't dispose explicitly** in your code — DI does it; explicit disposal still works but is redundant.

### StackExchange.Redis

One `ConnectionMultiplexer` per logical Redis instance, registered as a singleton. The multiplexer pipelines many operations over a small set of connections; you do *not* get a connection-per-call.

```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(new ConfigurationOptions
    {
        EndPoints = { "redis-host:6379" },
        AbortOnConnectFail = false,
        ConnectRetry = 5,
    }));
```

The most common pool-related Redis failure is creating a multiplexer per request. Always singleton.

## The usual anti-patterns

- **`using var client = new HttpClient()` per request.** Already the canonical mistake. Symptoms in [I/O saturation](./io-saturation.md).
- **Long-lived `HttpClient` without `PooledConnectionLifetime`.** Survives until the next downstream rollout silently breaks you.
- **Holding `SqlConnection` open across `await` boundaries that include unrelated work.** The connection is unavailable to others while your request waits on, e.g., an external HTTP call. Open just-in-time, work, dispose.
- **Different connection strings for the same DB.** Run a quick audit: `git grep "Server=" | sort -u`. Multiple pools cost you handshakes, idle connections, and memory.
- **Pool size set to "the max we ever need" without checking server-side limit.** 50 pods × 100 connections = 5000 — the database may cap at 1500. Plan globally; pool size is a *per-process* number that becomes a *fleet* number.
- **`Pooling=False` because "we had a bug once"**. The bug is almost never the pool; it's a leaked connection, a transaction not committed, or a cached `DbContext` state. Fix the leak, keep pooling on.
- **`OpenAsync()` without cancellation.** A pool starvation event becomes a hang because the wait has no deadline.
- **Disposing a multiplexer** (`ConnectionMultiplexer.Dispose()`) in a request scope. Singleton or process-lifetime, never per-request.

## Worked example — SQL with sized pool and cancellation

```csharp
// Before: synchronous Open, no cancellation; pool starvation → hang.
public Order Get(Guid id)
{
    using var conn = new SqlConnection(_cs);
    conn.Open();
    using var cmd = new SqlCommand("SELECT ... WHERE Id = @id", conn);
    cmd.Parameters.AddWithValue("@id", id);
    using var rdr = cmd.ExecuteReader();
    return rdr.Read() ? Read(rdr) : throw new KeyNotFoundException();
}

// After: async, cancellation-aware, fast-fail on pool exhaustion.
public async Task<Order> GetAsync(Guid id, CancellationToken ct)
{
    await using var conn = new SqlConnection(_cs);
    await conn.OpenAsync(ct);
    await using var cmd = new SqlCommand(
        "SELECT TOP 1 Id, CustomerId, Total FROM dbo.Orders WHERE Id = @id", conn);
    cmd.Parameters.Add("@id", SqlDbType.UniqueIdentifier).Value = id;
    await using var rdr = await cmd.ExecuteReaderAsync(ct);
    return await rdr.ReadAsync(ct) ? Read(rdr) : throw new KeyNotFoundException();
}
```

Connection string for this service:

```
Server=tcp:db.example.com,1433;Database=Orders;
Authentication=Active Directory Default;
Pooling=True;Min Pool Size=10;Max Pool Size=200;
Connect Timeout=15;Connection Lifetime=300;
TrustServerCertificate=False;Encrypt=True;
```

## Worked example — `IHttpClientFactory` with everything wired

```csharp
// Program.cs
builder.Services.AddHttpClient<IOrdersClient, OrdersClient>(c =>
{
    c.BaseAddress = new Uri("https://orders.svc/");
    c.Timeout = TimeSpan.FromSeconds(10);            // outer ceiling
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    MaxConnectionsPerServer    = 64,
    PooledConnectionLifetime   = TimeSpan.FromMinutes(5),
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
})
.AddStandardResilienceHandler(o =>
{
    o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);     // per attempt
    o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(8);// across retries
});
```

The handler is pooled by `IHttpClientFactory` for two minutes by default; after that, a new handler is built and the old one is disposed when its references drop. That's how DNS gets refreshed even without `PooledConnectionLifetime` — but `PooledConnectionLifetime` is more aggressive and predictable.

## Detection recipe

1. **Counters first**:
   ```bash
   dotnet-counters monitor System.Net.Http System.Net.Sockets Microsoft.Data.SqlClient.EventSource --process-id <pid>
   ```
   `connections-current-total`, `requests-queue-duration`, SQL `hard-connects` (rate of *new* connections, not pool reuse).
2. **Server side too**: pool exhaustion has two faces. Client-side is `Timeout expired`; server-side is rejected new connections. Check the database / downstream concurrent connection count.
3. **Detect leaks**: `hard-connects` rate sustained above zero (post warm-up) means the pool is creating new connections constantly — connections are being closed, not returned. Search for missing `await using` / `Dispose`.
4. **Trace open/close**:
   ```bash
   dotnet-trace collect --providers Microsoft-Diagnostics-DiagnosticSource --process-id <pid>
   ```
   `SqlClient` and `HttpClient` emit DiagnosticSource events for connection open / close — useful to identify the offending call site.

**Senior-level gotchas:**
- **Connection pool ≠ thread pool.** A pool exhaustion is independent of thread starvation; you can have plenty of workers and still block on `OpenAsync`.
- **`HttpClient.Timeout` triggers `OperationCanceledException`, not `TimeoutException`.** That's correct behavior — internally it cancels the request — but it confuses code that assumes "cancellation = user cancelled".
- **`PooledConnectionLifetime` is a maximum-age check at hand-out time**, not a kill timer. A request that started at second 299 still finishes — the connection just isn't reused after.
- **`MaxConnectionsPerServer` defaults to `int.MaxValue` for `SocketsHttpHandler`** but `2` for the legacy `HttpClientHandler` on `HttpWebRequest`. If you see strange throughput ceilings, check which handler is in use.
- **DNS TTL ≠ `PooledConnectionLifetime`.** The OS resolver caches DNS independently; a low TTL doesn't refresh the cached IP unless the resolver re-queries. `PooledConnectionLifetime` forces the *connection* to be torn down so the next acquisition triggers a re-resolution.
- **`AddDbContextPool` swallows certain bugs.** State held in property setters (e.g. an `IUserContext` cached in the constructor) leaks across requests because the same context instance is reused. Use `IDbContextFactory<T>` for explicit lifetime control if you need it.
- **TLS resumption is per-handler.** Two `HttpClient` instances pointed at the same host but with different handlers don't share TLS sessions. Use one typed client per logical upstream.
- **`ConnectionMultiplexer.Connect` is synchronous** and can take seconds on first call. Use `ConnectAsync` and warm at startup; never lazy-init in a request.
- **`IDbContextFactory<T>` is not a pool** unless registered via `AddPooledDbContextFactory<T>`. The non-pooled form constructs a new context every call.
- **EF Core `DbConnection` lifetime ≠ `DbContext` lifetime.** EF holds the connection only while a query is executing (or a transaction is open). `Database.OpenConnection()` extends that explicitly — useful for batching, but you own the dispose.
- **HTTP/2 changes pool semantics**: one connection multiplexes many streams. `MaxConnectionsPerServer` is rarely the right knob; consider `MaxResponseHeadersLength` and stream concurrency on the server side.
- **`SqlConnection` `Open` errors don't always indicate the pool.** They can come from auth (token expiry), DNS, network, server-side login throttling — read the inner exception. `pool` is mentioned literally in the message when the pool is the cause.
