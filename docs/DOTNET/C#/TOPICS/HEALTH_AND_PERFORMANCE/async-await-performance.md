# async/await performance

_Targets .NET 10 / C# 14. See also: [ValueTask](../ASYNC_PROGRAMMING/valuetask.md), [ConfigureAwait and SynchronizationContext](../ASYNC_PROGRAMMING/configureawait-and-synchronizationcontext.md), [Low-allocation patterns in modern .NET](./low-allocation-patterns-modern-dotnet.md), [Thread contention](./thread-contention.md), [Synchronization bottlenecks](./synchronization-bottlenecks.md), [dotnet-counters](./dotnet-counters.md), [Closure captures](./closure-captures.md)._

`async`/`await` is a compile-time rewrite, not a runtime feature. The compiler turns the method body into a state machine struct; the cost model follows from how that struct is moved between the stack and the heap, and from where the continuations end up running. Most "async is slow" complaints are actually **thread-pool starvation** caused by sync-over-async or by accidental work in the synchronous prefix of an async method — not by the await machinery itself.

## What the compiler actually emits

For every `async` method, the C# compiler generates a `MoveNext`-driven state machine. The shape (simplified):

```csharp
[AsyncStateMachine(typeof(<DoWork>d__1))]
public Task<int> DoWork() {
    var sm = new <DoWork>d__1 { <>t__builder = AsyncTaskMethodBuilder<int>.Create(), <>1__state = -1 };
    sm.<>t__builder.Start(ref sm);            // synchronous prefix runs here
    return sm.<>t__builder.Task;
}
```

The state machine is a **struct**. As long as the method completes synchronously (every `await` returns `IsCompleted == true`), it never leaves the stack — **zero heap allocation** for the orchestration. The first time `await` actually has to wait, the runtime boxes the struct into an `AsyncStateMachineBox<T>` on the heap and reuses that box for the rest of the method's lifetime.

So the cost of `await` is bimodal:

- **Sync completion**: a few cycles of struct copy and a returned cached/pooled `Task`.
- **Async completion**: one heap allocation for the box (per call), continuation queued to the thread pool.

The runtime caches the box per-method-invocation, not per-method — every async call that suspends pays once.

## Cached `Task` results: free completions

```csharp
return Task.CompletedTask;        // singleton
return Task.FromResult(true);     // cached for bool
return Task.FromResult(0);        // cached for small ints (-1..8)
return Task.FromResult(value);    // allocates a fresh Task<T>
```

`Task.FromResult` only caches a small set: `Task.CompletedTask`, `Task<bool>` for both values, and small `Task<int>` (`-1..8`) and a few other primitives. For everything else it allocates. If your hot path returns `Task<T>` for a non-cached value, prefer `ValueTask<T>` — `new ValueTask<T>(value)` is **never** an allocation.

## `ValueTask<T>`: the sync-completion fast path

```csharp
public ValueTask<int> ReadAsync(byte[] buf) {
    if (_buffered > 0) return new ValueTask<int>(DrainSync(buf));   // alloc-free
    return new ValueTask<int>(ReadCoreAsync(buf));                  // wraps a real Task
}
```

`ValueTask<T>` is a discriminated union of `(T, Task<T>, IValueTaskSource<T>)`. Use it when:

- The method **frequently** completes synchronously (cache hits, buffered reads, fast-path branches).
- Or the method participates in an interface where one implementation is sync and another is async.

Don't use it when the method always genuinely awaits — the wrapping overhead pays off only on sync completions. And **never await a `ValueTask` twice**: the contract is single-await, and violating it silently corrupts the underlying `IValueTaskSource`. If you need to fan out, call `.AsTask()` and pay the allocation.

For the very hottest async methods, `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<T>))]` pools the state machine box itself. Worthwhile when allocation profiling shows the box dominating; otherwise the pool's coordination cost outweighs the saved alloc.

## `ConfigureAwait(false)` in libraries

Every `await` captures the current `SynchronizationContext` (or `TaskScheduler`) by default and resumes on it. In ASP.NET Core there is **no** SynchronizationContext, so the capture is cheap — but in WinForms, WPF, MAUI, or older ASP.NET (`System.Web`), capture + post is a real cost: a heap allocation for the `ExecutionContext` snapshot plus a context-switch-shaped resume.

```csharp
// Library code: never assume the caller has no SyncCtx.
await stream.ReadAsync(buffer, ct).ConfigureAwait(false);
```

Application code in ASP.NET Core does not need `.ConfigureAwait(false)` — there's nothing to capture. **Library** code that may run anywhere should always use it. The .NET 8+ analyzer **CA2007** flags missing `ConfigureAwait` in library projects.

## Continuations: synchronous vs. queued

`TaskCompletionSource<T>` defaults to **synchronous continuations** — the awaiting code runs on whatever thread called `SetResult`. That has two consequences:

1. The `SetResult` call doesn't return until the continuation finishes — your producer is now doing consumer work.
2. Re-entrant code can deadlock or stack-dive in unexpected ways.

```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
//                                       ^^^ critical for any TCS that crosses code you don't own
```

**Always set `RunContinuationsAsynchronously`** on a TCS unless you know precisely who awaits it and why inline resumption is desired. The default is a footgun preserved for compatibility.

## `Task.WhenAll` and `Task.WhenAny`

```csharp
await Task.WhenAll(tasks);              // tasks: IEnumerable<Task> → array materialized internally
await Task.WhenAll(t1, t2, t3);         // params overload — array allocation per call
await Task.WhenAll([t1, t2, t3]);       // explicit array, identical cost
```

`Task.WhenAll(IEnumerable<Task>)` enumerates and copies into an array internally; that array plus the combined `Task` are the per-call cost. For **two** tasks specifically, prefer `await (t1, t2)` via `Task.WhenAll(t1, t2)` and accept the small alloc, or restructure to await sequentially when ordering is fine.

Avoid `Task.WhenAll(largeList.Select(x => DoAsync(x)))` for unbounded `largeList` — every started task allocates a state machine box plus its captured closure. Use `Parallel.ForEachAsync(largeList, ct, async (x, t) => ...)` (.NET 6+), which uses a worker pool and bounds in-flight concurrency.

## `async void`: never, with two exceptions

`async void` skips the Task allocation but breaks every property the runtime relies on:

- Exceptions are raised on the captured SynchronizationContext (in ASP.NET Core, that's the default `TaskScheduler`) and in practice **terminate the process**.
- The caller cannot await completion or cancellation.

The two legitimate uses:

- **Event handlers** (`button.Click += async (s, e) => ...`) — required by the event signature; wrap the body in `try`/`catch`.
- **Top-level `Main` in some hosted scenarios** — but `async Task Main` exists and is preferred.

Anywhere else, **`async void` is a bug**.

## `await using` and `IAsyncDisposable`

```csharp
await using var conn = await dataSource.OpenConnectionAsync(ct);
await conn.ExecuteAsync(sql, ct);
// dispose runs at scope end — adds one async op
```

The dispose adds one extra `await`. For a pooled resource (Npgsql connection, HttpClient handler) that's the price of correctness. Don't fall back to synchronous `using` over an `IAsyncDisposable` — `Dispose` may block on async work and cause exactly the thread-pool starvation you were trying to avoid.

## Thread-pool starvation: the real "async is slow"

Most production async incidents trace to **the thread pool running out of workers**. Causes, in descending frequency:

| Cause | Symptom | Fix |
|---|---|---|
| `.Result` / `.Wait()` on a `Task` from a request thread | Latency cliff under load; deadlocks in legacy SyncCtx | Make the call chain async to the top |
| Blocking I/O (`SqlConnection.Open`, `File.ReadAllBytes`) inside `async` methods | Threads parked in kernel waits | Use the `Async` variants |
| `Task.Run(() => longRunningCpu)` per request | Pool fills with CPU work; queued I/O waits | Use a dedicated `LongRunning` task or a worker channel |
| `lock`/`Monitor.Enter` over async work | `OperationCanceledException` arrives late | `SemaphoreSlim.WaitAsync` |
| `Parallel.For` over async-flavored work | Doubles the pool footprint | `Parallel.ForEachAsync` |
| Many small tasks fanned out at once | Sudden spike in `ThreadPool.QueueLength` | Bounded `Channel<T>` or `Parallel.ForEachAsync` |

The pool grows by one worker per ~500 ms when starved (`ThreadPool.SetMinThreads` raises the floor; the default is `Environment.ProcessorCount`). Until it does, queued work just waits — and your p99 latency reflects it.

## Measurement

```bash
dotnet-counters monitor System.Runtime --process-id <pid>
# columns to chart:
#   threadpool-thread-count        - active workers
#   threadpool-queue-length        - work waiting for a worker (the starvation signal)
#   threadpool-completed-items-count - throughput
#   monitor-lock-contention-count  - lock fights (often disguised as "async slowness")
#   alloc-rate                     - per-second bytes; tracks state-machine alloc
```

A sustained non-zero `threadpool-queue-length` is the diagnostic for starvation. `monitor-lock-contention-count` rising in step with traffic indicates a `lock` on the async path that should be a `SemaphoreSlim`.

For micro-level allocation analysis:

```csharp
[MemoryDiagnoser]
public class AsyncBenches {
    [Benchmark] public async Task<int> Sync() => await Task.FromResult(42);
    [Benchmark] public async ValueTask<int> SyncVT() => 42;   // alloc-free
}
```

The `Allocated` column is the load-bearing number. `0 B` per op on `SyncVT` is the proof; non-zero on `Sync` is the state-machine box plus the `Task`.

## Senior-level gotchas

- **Async methods that never await still pay for the state machine struct** — the compiler emits the rewrite unconditionally. If a method has a fast path that genuinely returns synchronously every time, drop the `async` keyword and return `Task.CompletedTask` / `new ValueTask<T>(value)` directly.
- **The synchronous prefix of an `async` method runs on the caller's thread.** Code before the first `await` blocks the caller — heavy work there (config parsing, big LINQ materialization) defeats the point. Push it past the first `await` or do it lazily.
- **`Task.Run` inside an `async` method is almost always wrong.** It moves the synchronous prefix to a pool thread, doubling your pool footprint and adding a queue hop. The right move is to make the prefix actually async, or to remove the `async` if it isn't.
- **`ConfigureAwait(false)` does nothing in ASP.NET Core** because there is no `SynchronizationContext`, but it still serves as a documentation contract for code that may be lifted into a desktop or test host. In libraries, use it everywhere.
- **`async void` exceptions are unobservable and crash the process.** The single-most common production async incident is a forgotten event handler that throws under load. Wrap them; or restructure to `Task` returns wired through a launcher that observes the exception.
- **TCS without `RunContinuationsAsynchronously` is a deadlock waiting for a debugger.** Inline continuations in shared infrastructure (channels, custom awaiters) re-enter the producer with arbitrary user code on the call stack. Always set the flag.
- **`ValueTask` used as an `await` target twice is undefined behavior**, not a `Task`-style "second await returns the same result." On debug it throws; on release it can corrupt the `IValueTaskSource`'s pool. Treat the contract as *single-consume*.
- **Pooling `ValueTask` state machines (`PoolingAsyncValueTaskMethodBuilder`) helps only on truly hot methods.** The pool's CAS overhead is a few dozen cycles; if your method is called once per request, the pool wins nothing. Measure, don't guess.
- **`Task.WhenAll` over an unbounded enumerable starts every task immediately.** A million-element select is a million in-flight async ops — a thread-pool DoS by daylight. Use `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`.
- **`AsyncLocal<T>` survives `await` and forks on copy-on-write semantics** — the `ExecutionContext` is captured at every `await` continuation. Setting an `AsyncLocal` deep in a call stack does not propagate back up the stack. This catches everyone exactly once.
- **`HttpClient.Timeout` cancels the request via a `CancellationToken` linked under the hood** — a `using` over the response disposes the linked token before your `await` of the response stream completes. If you stream a large body and see `TaskCanceledException` near the timeout boundary, the timeout was racing your stream consumption, not your initial request.
- **`ThreadPool.SetMinThreads` is not a fix; it's a buffer.** Raising the floor masks starvation by paying for idle threads. The right fix is to remove the blocking call. Use `SetMinThreads` only as a known band-aid while you do the real work.
- **`CancellationToken.None` skips registration overhead**, but a "real" token registers a callback per `await` that observes it. In a tight loop, prefer `cancellationToken.ThrowIfCancellationRequested()` once per logical chunk rather than threading it through every awaitable.
