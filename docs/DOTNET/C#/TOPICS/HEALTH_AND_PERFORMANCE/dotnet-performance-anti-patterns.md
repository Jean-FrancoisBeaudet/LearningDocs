# .NET performance anti-patterns

_Targets .NET 10 / C# 14. See also: [LINQ performance misuse](./linq-performance-misuse.md), [Closure captures](./closure-captures.md), [async/await performance](./async-await-performance.md), [Thread contention](./thread-contention.md), [Connection pooling](./connection-pooling.md), [Allocations and boxing](./allocations-and-boxing.md)._

A catalogue of the recurring smells that make ASP.NET Core, EF Core, and worker services run hot. Each has a one-line fix and a one-line "why it's worse than it looks". Most of them are **detected by Roslyn analyzers** that ship in the .NET SDK — turning them on (`<AnalysisMode>All</AnalysisMode>`) catches the bulk of this list at build time.

## Concurrency

### `async void`

```csharp
public async void OnButtonClick(...) { await DoWork(); }   // event handler — fine
public async void HandleMessage(...)  { await Process();  } // worker — broken
```

`async void` exceptions cannot be caught by the caller — they go straight to `AppDomain.UnhandledException` and crash the process. Use `async void` **only** for UI event handlers that the framework demands. Everything else returns `Task` or `ValueTask`.

### `.Result` / `.Wait()` / `.GetAwaiter().GetResult()`

```csharp
var result = SomeAsyncMethod().Result;   // sync-over-async
```

Three failure modes: deadlock under a `SynchronizationContext` (UI, ASP.NET pre-Core), thread-pool starvation (worker thread blocks waiting for the same pool to schedule the continuation), and exception wrapping in `AggregateException`. Modern stacks have no `SynchronizationContext` so the deadlock is gone, but starvation is real and tail-latency-shaped — under load you don't crash, you just queue.

```csharp
var result = await SomeAsyncMethod();
```

If you genuinely cannot `await` (e.g. constructor, `Main` in legacy code, `Dispose`), isolate the call to a clearly-named `RunSync` helper and document the constraint.

### Sync-over-async on the request thread

The classic shape is a synchronous controller method that calls `service.LoadAsync().Result`. Every request burns a thread-pool thread blocking on I/O. At ~1000 RPS with 100 ms backend latency, you've consumed 100 threads steady-state for nothing — and the thread pool grows by **one thread per second** by default. The system fails under burst, not steady-state, which is why staging never sees it.

`dotnet-counters monitor System.Runtime` → `threadpool-thread-count` rising while CPU is idle is the unambiguous signal. Cross-link: [thread-contention.md](./thread-contention.md).

### Missing `ConfigureAwait(false)` in library code

ASP.NET Core has no `SynchronizationContext`, so on application code the missing `ConfigureAwait` is invisible. **Library code** is different — your library is consumed by WPF/WinForms apps too, and the absent `ConfigureAwait(false)` leaves them paying a context capture per `await`. Roslyn analyzer **CA2007** flags this. Apply at the library level (e.g. via `<Nullable>enable</Nullable>` + `[assembly: ConfigureAwait(false)]` if using the `ConfigureAwaitAnalyzer`).

### `Task.Run` to fake parallelism

```csharp
var result = await Task.Run(async () => await client.GetAsync(url));
```

Wrapping an already-async call in `Task.Run` adds a thread-pool hop, an extra state-machine allocation, and zero parallelism. The async I/O was already non-blocking. `Task.Run` belongs only when offloading **CPU-bound** work off the request thread (and even then, ASP.NET Core already runs request work on the thread pool — so there's nothing to offload to).

### `lock` over `await`

```csharp
lock (_gate) { await DoAsync(); }   // CS1996 — illegal
```

The compiler blocks this because the continuation could run on a different thread that doesn't own the monitor. The fix is `SemaphoreSlim` (1, 1) and `WaitAsync`/`Release`, or — better — `Channel<T>` to serialize work without a lock at all.

### `IAsyncEnumerable` consumed with `.ToList()`

```csharp
var rows = await db.Orders.AsAsyncEnumerable().ToListAsync(ct);
```

Defeats the entire point. `IAsyncEnumerable<T>` exists to bound memory; materializing it loads the whole stream. If you need the list, query for the list — don't stream-then-collect.

## Allocation

### String concatenation in loops

```csharp
string s = "";
foreach (var item in items) s += item.ToString() + ",";   // O(n²) allocation
```

Each `+=` allocates a new string holding the entire prefix. Use `StringBuilder`, `string.Join`, or — on hot paths — `string.Create(len, state, span => ...)` for zero-intermediate construction. C# 10's `[InterpolatedStringHandler]` makes `$"{a}{b}"` allocation-free in many BCL APIs (`StringBuilder.Append`, `Span.TryWrite`, `ILogger.LogInformation`).

### `List<T>` without capacity

```csharp
var list = new List<T>();
for (int i = 0; i < 10_000; i++) list.Add(...);   // ~14 reallocations + copies
```

`List<T>` doubles its backing array each time it overflows (4 → 8 → 16 → ...). With a known size, pass it: `new List<T>(10_000)`. Same lesson for `Dictionary<K, V>`, `HashSet<T>`, `StringBuilder`. The intermediate arrays go to LOH past 85 KB — see [large-object-heap-loh.md](./large-object-heap-loh.md).

### Boxing on `struct` enumerators / value-type dictionary keys

`foreach` over a `struct` enumerator does **not** box. But casting to `IEnumerable` does:

```csharp
IEnumerable<T> seq = list;          // List<T>.Enumerator (struct) gets boxed on GetEnumerator
foreach (var x in seq) { ... }
```

Same trap on `Dictionary<int, string>`: missing `IEquatable<int>` (it's there for primitives, but check on custom value types) means `Equals(object)` boxes both keys on every lookup. `EqualityComparer<T>.Default` resolves to a no-box implementation **only** when `T` implements `IEquatable<T>`.

### `Enum.HasFlag`

`HasFlag` boxes both operands on .NET Framework. On .NET Core/5+ the JIT special-cases it to a bitwise compare — so it's fine on modern runtimes. The bitwise `(flags & MyFlag) == MyFlag` is still preferred for readability and guarantees zero allocation on every TFM.

### Logging with `$""` interpolation

```csharp
_logger.LogInformation($"Processed {count} items in {sw.ElapsedMilliseconds} ms");
```

The interpolation runs **before** the level filter — the string is built and the args boxed even if `Information` is disabled. Use the message template:

```csharp
_logger.LogInformation("Processed {Count} items in {Elapsed} ms", count, sw.ElapsedMilliseconds);
```

Now formatting is deferred to the sink. Better still: source-generated logging via `[LoggerMessage]` (.NET 6+) — zero-alloc, compile-time-checked templates.

```csharp
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Processed {Count} items in {Elapsed} ms")]
    public static partial void ProcessedItems(this ILogger logger, int count, long elapsed);
}
```

Roslyn analyzers **CA1848** (use `LoggerMessage`) and **CA2254** (template-not-constant) catch the common shapes.

## EF Core

### `Include` chains

```csharp
db.Orders.Include(o => o.Customer).Include(o => o.Items).ThenInclude(i => i.Product).ToList();
```

Each `Include` joins another table, often producing a Cartesian explosion. Use **split queries** (`AsSplitQuery()` since EF Core 5) for one-to-many chains, or project to a DTO and let EF emit a single tight `SELECT`:

```csharp
db.Orders.Select(o => new OrderDto(o.Id, o.Customer.Name, o.Items.Count)).ToList();
```

### N+1

```csharp
var orders = db.Orders.ToList();
foreach (var o in orders) o.Customer = db.Customers.Find(o.CustomerId);   // N round trips
```

`Include` or projection. Profile with EF logging at `Information` — every `Executed DbCommand` is a real round trip; one row per request is N+1.

### `IQueryable` leaving the repository

When `IQueryable<T>` escapes the data layer, callers can append `Where`, `Include`, `OrderBy` from anywhere — and accidentally trigger client evaluation, leak schema, or block the connection until full materialization. Repositories should return `IReadOnlyList<TDto>` (or `IAsyncEnumerable<TDto>` for streams), not `IQueryable<TEntity>`.

### Forgetting `AsNoTracking` on read paths

```csharp
var orders = await db.Orders.Where(...).ToListAsync(ct);   // every entity is tracked
```

Change tracking allocates a tracking entry per entity and snapshots all original values. On a read-only path, `.AsNoTracking()` cuts allocations and time roughly in half:

```csharp
var orders = await db.Orders.AsNoTracking().Where(...).ToListAsync(ct);
```

Or set the default at context level: `optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)`.

## DI / scoping

### Singleton capturing Scoped

```csharp
services.AddSingleton<IOrderService, OrderService>();
services.AddScoped<DbContext, MyDbContext>();
// OrderService injects DbContext — captured for the singleton's lifetime.
```

The captured `DbContext` outlives its scope. EF starts throwing once a request boundary closes the underlying connection. The runtime catches some of this with `ValidateOnBuild`/`ValidateScopes` — turn both on in `Host.ConfigureServices` for dev/staging.

```csharp
.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;
    o.ValidateOnBuild = true;
})
```

### `IServiceProvider` as a service locator

```csharp
public class Foo
{
    private readonly IServiceProvider _sp;
    public Foo(IServiceProvider sp) => _sp = sp;
    public void Run() { _sp.GetRequiredService<IBar>().Do(); }
}
```

Hides dependencies, breaks scope validation, prevents constructor analysis. Inject `IBar` directly. The legitimate uses of `IServiceProvider` are: factory patterns where `T` is decided at runtime, and scope creation in long-running workers (`scope = sp.CreateAsyncScope()`).

## HTTP / networking

### `HttpClient` per request

```csharp
using var client = new HttpClient();
return await client.GetAsync(url);
```

Each `new HttpClient()` opens a new connection pool; the underlying socket lingers in `TIME_WAIT`. Under load, you exhaust ephemeral ports (~28K on Windows by default) and start failing with `SocketException`. Inject `HttpClient` from `IHttpClientFactory` — it pools connections per logical name and rotates handlers. Cross-link: [connection-pooling.md](./connection-pooling.md).

```csharp
services.AddHttpClient<IGitHubClient, GitHubClient>(c => c.BaseAddress = new(...));
```

### Reflection on every call

```csharp
public static T Map<T>(object src) where T : new()
{
    var dst = new T();
    foreach (var p in typeof(T).GetProperties())   // reflection per call
        p.SetValue(dst, src.GetType().GetProperty(p.Name)!.GetValue(src));
    return dst;
}
```

Cache the `PropertyInfo`s once per type, or — better — switch to a source generator (`Mapperly`, `Riok.Mapperly`) that emits the mapping at compile time. Same trap with `Activator.CreateInstance<T>()` in hot paths; cache a compiled `Func<T>` if you must.

## Exceptions

### Exceptions for control flow

```csharp
try { return int.Parse(input); }
catch { return 0; }
```

Throwing is ~1000× slower than returning. First-chance exceptions also pollute crash dumps and confuse debuggers. Use `int.TryParse(input, out var n)`; for collections, `Dictionary.TryGetValue`, `List.IndexOf`, `Span.TryCopyTo`. Reserve `throw` for genuinely exceptional conditions — broken invariants, infrastructure failures.

### Catching `Exception`

```csharp
try { ... } catch (Exception ex) { _log.LogError(ex, "..."); }
```

Swallows `OutOfMemoryException`, `StackOverflowException`, `ThreadAbortException` (legacy), `OperationCanceledException` (which is **not** an error — it's the signal). At minimum re-throw `OperationCanceledException` separately:

```csharp
catch (OperationCanceledException) when (ct.IsCancellationRequested) { throw; }
catch (Exception ex) { _log.LogError(ex, "..."); throw; }
```

## Tooling — what to run on a suspect codebase

| Tool | What it catches |
|---|---|
| `<AnalysisMode>All</AnalysisMode>` in csproj | Most of this list as build-time warnings |
| `Microsoft.CodeAnalysis.NetAnalyzers` | CA1822 (mark static), CA1859 (concrete return), CA1860 (Length over Count()), CA1865-7 (single-char overloads), CA2007 (ConfigureAwait), CA2254 (logger template) |
| BenchmarkDotNet `[MemoryDiagnoser]` | Hidden allocations |
| `dotnet-counters` | Thread-pool starvation, GC pressure, allocation rate |
| PerfView "GC Heap Alloc Stacks" | Top-N allocators by stack |
| EF Core logging (`LogLevel.Information`) | N+1, client eval, missing AsNoTracking |
| `services.UseDefaultServiceProvider(o => o.ValidateScopes = o.ValidateOnBuild = true)` | DI scope mismatches at startup |

## Senior-level gotchas

- **`ValueTask` is not a drop-in `Task`.** Awaiting a `ValueTask` twice, calling `.Result` on a non-completed one, or storing it in a field is undefined behavior. The rule: await it exactly once, immediately. Use `Task` for fields, public APIs that callers might cache, and any `WhenAll`-shaped composition. `ValueTask` is for the synchronous-completion fast path on hot APIs (`ReadAsync`, `MoveNextAsync`).
- **`async` returning `void` from a `Task.Run`** silently drops exceptions — `Task.Run` of an `async void` lambda doesn't capture the failure into the returned `Task`. Always return `Task` from the lambda body.
- **Cancellation is not free.** Every `await` checks the token; pluming it through a 12-deep call chain is right. **Not** plumbing it leads to phantom CPU on a request the client gave up on three timeouts ago — the worst kind of leak because it doesn't trip a watchdog.
- **Double-checked locking on a non-volatile field is broken.** The JIT can reorder the read past the lock entry on weak memory models (ARM, ARM64, which is now the cloud-default). Use `Lazy<T>` (`LazyThreadSafetyMode.ExecutionAndPublication`) or `LazyInitializer.EnsureInitialized` — they get the memory barriers right.
- **`[ThreadStatic]` does not flow across `await`.** Continuations resume on a different thread; the `[ThreadStatic]` value is whatever was on *that* thread last. Use `AsyncLocal<T>` for per-flow state.
- **`HttpClient` reuses sockets but caches DNS until the handler rotates.** `IHttpClientFactory` rotates the handler every 2 minutes by default — long enough that a DNS change to a backend takes up to 2 minutes to take effect. Configure `HandlerLifetime` deliberately for blue-green-aware services.
- **Server GC + 1 vCPU container is bad** (see [server-vs-workstation-gc.md](./server-vs-workstation-gc.md)) — set `DOTNET_GCHeapCount=1` or enable DATAS.
- **`[MemoryPackable]` / `[MessagePackObject]` / source-generated JSON** beat reflection-based serializers by 3–10× on hot endpoints — and are mandatory under NativeAOT, which strips reflection metadata.
- **`StringComparer.OrdinalIgnoreCase` is faster than `string.Equals(..., StringComparison.OrdinalIgnoreCase)`** at the dictionary level — passing the comparer to the dictionary saves the per-lookup branch on culture handling. Roslyn analyzer **CA1862** flags the missed cases.
- **CPU-bound code wrapped in `Task.WhenAll(items.Select(async ...))`** does not parallelize on the thread pool unless each task has an actual `await`. A tight CPU loop inside an async lambda runs sequentially. Use `Parallel.For`, `Parallel.ForEach`, or — for I/O — `Parallel.ForEachAsync` with bounded `MaxDegreeOfParallelism`.
- **`Dispose` on a partially-initialized object** (constructor threw mid-way) is a classic resource leak. Pattern: acquire resources in field initializers when possible; in constructors, wrap acquisition in a try/catch that disposes already-acquired resources before re-throwing.
- **Stopwatch precision is sub-microsecond — `DateTime.UtcNow` is ~15 ms on Windows.** Don't time hot paths with `DateTime`. `Stopwatch.GetTimestamp()` returns ticks at QPC frequency; `Stopwatch.GetElapsedTime(startTs)` (.NET 7+) avoids constructing a `Stopwatch` per measurement.
