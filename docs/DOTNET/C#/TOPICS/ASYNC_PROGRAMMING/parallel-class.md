# Parallel Class

_Targets .NET 10 / C# 14. See also: [Parallel LINQ](./parallel-linq.md), [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [Task continuation, cancellation, and multiple tasks](./task-continuation-cancellation-and-multiple-tasks.md)._

The `System.Threading.Tasks.Parallel` class is for **CPU-bound data parallelism**: the same operation applied independently to many items, using multiple threads. It is **not** a general-purpose "run a bunch of things" API — for async I/O or heterogeneous work, use `Task.WhenAll` (see [task-continuation-cancellation-and-multiple-tasks.md](./task-continuation-cancellation-and-multiple-tasks.md)) or `Parallel.ForEachAsync`.

## The four entry points

| API | Shape | Use case |
|---|---|---|
| `Parallel.For` | Integer index range. | `for (int i = 0; i < n; i++) { Work(i); }` in parallel. |
| `Parallel.ForEach` | `IEnumerable<T>`. | Data-parallel loop over a collection. |
| `Parallel.ForEachAsync` (.NET 6+) | `IEnumerable<T>` / `IAsyncEnumerable<T>` with an async body. | Bounded fan-out of async I/O. |
| `Parallel.Invoke` | N actions. | Run a small fixed set of independent operations. |

All four block the calling thread until every worker has finished, **except** `ForEachAsync` which is naturally `Task`-returning.

## `Parallel.For` and `Parallel.ForEach`

```csharp
Parallel.For(0, rows.Count, i =>
{
    rows[i] = Transform(rows[i]);
});

Parallel.ForEach(files, path =>
{
    var hash = HashFile(path);
    bag.Add((path, hash));
});
```

Under the hood the TPL partitions the range/source across worker tasks (default: up to `Environment.ProcessorCount` of them) and the body runs on pool threads.

### `ParallelOptions`

```csharp
var opts = new ParallelOptions
{
    MaxDegreeOfParallelism = 8,
    CancellationToken = ct,
    TaskScheduler = TaskScheduler.Default,
};
Parallel.ForEach(files, opts, path => HashFile(path));
```

- `MaxDegreeOfParallelism = -1` means "unbounded" (= `ProcessorCount` for `Parallel.For*`). Set an explicit cap for I/O-bounded or resource-limited work; otherwise you can flood the pool.
- `CancellationToken` causes the loop to throw `OperationCanceledException` at the next partition boundary. Bodies must *also* poll the token for fast cancellation — the loop itself won't interrupt a running body.
- `TaskScheduler` lets you run on a custom scheduler; virtually never needed.

### `ParallelLoopState` — Break vs Stop

```csharp
Parallel.For(0, items.Length, (i, state) =>
{
    if (!Validate(items[i]))
    {
        state.Break();   // finish items with index < i, then stop
        return;
    }
    Process(items[i]);
});
```

| Method | Semantics |
|---|---|
| `state.Break()` | Signal "stop ASAP, but still process items with a lower ordinal than this." Useful when you want all the early items finished. |
| `state.Stop()` | Signal "stop ASAP, no ordinal guarantee." Faster exit; no "finish the earlier stuff" promise. |

Check `state.ShouldExitCurrentIteration` or `state.IsStopped` inside long-running iterations to bail early.

### Aggregating with thread-local state

The body runs on arbitrary threads concurrently, so **shared mutable state needs locking** — or better, use the thread-local overload:

```csharp
long total = 0;
Parallel.For(
    0, data.Length,
    localInit: () => 0L,
    body: (i, _, localSum) => localSum + data[i],
    localFinally: localSum => Interlocked.Add(ref total, localSum));
```

Each worker accumulates privately (no contention), then a single `Interlocked.Add` combines the results. This is the canonical parallel-reduce pattern and is much faster than `lock`-ing in the body.

## `Parallel.ForEachAsync` — the modern answer for async fan-out

```csharp
await Parallel.ForEachAsync(
    urls,
    new ParallelOptions { MaxDegreeOfParallelism = 16, CancellationToken = ct },
    async (url, token) =>
    {
        var content = await http.GetStringAsync(url, token);
        await repo.SaveAsync(url, content, token);
    });
```

This replaces the hand-rolled "fan-out + `SemaphoreSlim`" pattern. It bounds concurrency, respects cancellation, and doesn't suffer from the classic `Parallel.ForEach(async () => ...)` trap of firing all items at once (which is what happens because `Parallel.ForEach` doesn't `await` the body).

Key properties:
- Body signature is `(T item, CancellationToken token)` returning `ValueTask`. The token combines the caller's `ct` with an internal one for short-circuiting on first exception.
- Exceptions from any iteration are aggregated into an `AggregateException` (like `Task.WhenAll`).
- Default `MaxDegreeOfParallelism` for async is `Environment.ProcessorCount` — often wrong for I/O. Set it explicitly.

## `Parallel.Invoke`

```csharp
Parallel.Invoke(
    () => LoadCustomers(),
    () => LoadOrders(),
    () => LoadInvoices());
```

A thin wrapper around `Task.WaitAll`. Fine for small, fixed fan-out of synchronous work. For async, use `Task.WhenAll`:

```csharp
await Task.WhenAll(LoadCustomersAsync(ct), LoadOrdersAsync(ct), LoadInvoicesAsync(ct));
```

## When NOT to use `Parallel.*`

| Symptom | What to use instead |
|---|---|
| Body is `async` / does I/O. | `Parallel.ForEachAsync` — **not** `Parallel.ForEach(async () => ...)`. |
| Items arrive over time (stream). | `Channel<T>` consumer + N workers, or `Parallel.ForEachAsync` over `IAsyncEnumerable<T>`. |
| Work is heterogeneous (different sub-operations). | `Task.WhenAll`. |
| You need a result set with LINQ-style operators. | [PLINQ](./parallel-linq.md). |
| Single ASP.NET Core request handler. | Usually nothing — you're already on a pool thread. Adding parallelism multiplies the load on an overloaded pool. |

### The `Parallel.ForEach(async () => ...)` anti-pattern

```csharp
// ❌ This does NOT do what you think.
Parallel.ForEach(urls, async url =>
{
    var content = await http.GetStringAsync(url);
    Save(content);
});
```

`Parallel.ForEach` sees the body returning `void` (it's an `async void` lambda bound to `Action<T>`), so it fires all items as fire-and-forget and returns immediately. Exceptions are unobservable, cancellation doesn't flow, ordering is meaningless. Always reach for `Parallel.ForEachAsync`.

## Partitioning

For uneven workloads, the default range partitioner can under-utilize cores (some threads finish early while others churn). Use `Partitioner.Create` for chunked or dynamic partitioning:

```csharp
var source = Partitioner.Create(items, EnumerablePartitionerOptions.NoBuffering);
Parallel.ForEach(source, item => Work(item));
```

`NoBuffering` gives you fine-grained partitioning — items are fetched one-at-a-time under lock — which helps when items differ wildly in cost. Defaults are fine for uniform CPU work.

## Determining `MaxDegreeOfParallelism`

Rules of thumb:
- **CPU-bound, compute-heavy**: `ProcessorCount` (the default). More threads just increases context switching.
- **Mixed CPU + I/O**: benchmark; typically `ProcessorCount * 2` to `ProcessorCount * 4`.
- **Pure async I/O**: bound by the downstream (DB connection pool, remote rate limit). Often 16–64 regardless of core count.
- **Memory-allocation-heavy**: fewer threads may actually be faster due to GC contention.

Never leave it at `-1` for I/O work. You *will* saturate something.

**Senior-level gotchas:**
- `Parallel.ForEach(async () => ...)` is one of the most common high-impact bugs in .NET. Use `Parallel.ForEachAsync`.
- `Parallel.For` and friends **block the calling thread** until done. Call them from a `Task.Run` if you're on a UI thread (WPF/WinForms) — otherwise you freeze the UI.
- `ConcurrentBag<T>` inside a Parallel body has surprising ordering and high contention on `TryTake`. Prefer a `List<T>` per thread (via `localInit`/`localFinally`) and combine at the end, or `ConcurrentQueue<T>` / `ConcurrentDictionary<K,V>`.
- Exceptions in a parallel loop are wrapped in `AggregateException`. `await Parallel.ForEachAsync` throws the *unwrapped* first exception (like any `async` method) — but the returned `Task`'s `.Exception` is the aggregate.
- `Parallel.For(0, n, i => ...)` inside an already-parallel context (another `Parallel` loop, a PLINQ query) double-fans-out. Each outer iteration spawns `ProcessorCount` workers × `ProcessorCount` workers. Collapse nested parallelism or cap `MaxDegreeOfParallelism` on the inner loop.
- Cancellation is checked at partition boundaries, not inside the body. A long-running body that never checks `ct` will run to completion even after cancel. Plumb the token into the body.
- `Parallel` uses `TaskScheduler.Default` — it does **not** pick up the caller's sync context. Any `ConfigureAwait(true)` inside the body is effectively a no-op because there's no captured context to return to.
- For hot reductions (`sum`, `min`, `max`), a `PLINQ` `Aggregate` with a combiner often beats `Parallel.For` with `Interlocked` — benchmark before assuming.
