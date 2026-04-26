# Thread contention

_Targets .NET 10 / C# 14. See also: [Synchronization bottlenecks](./synchronization-bottlenecks.md), [async/await performance](./async-await-performance.md), [Channel&lt;T&gt;](./channel-of-t.md), [dotnet-counters](./dotnet-counters.md), [dotnet-trace](./dotnet-trace.md), [Thread / ThreadPool / Task / TaskFactory](../ASYNC_PROGRAMMING/thread-threadpool-task-taskfactory.md)._

**Thread contention** is a *scheduling* problem, not a *CPU* problem.

> Threads are ready to do work but cannot — they are queued behind the CPU scheduler, parked on a lock, or waiting for the thread pool to grow. The application is doing less than the hardware allows because threads are getting in each other's way.

It is regularly mistaken for three adjacent problems, and the fix list does not transfer:

| Problem | Signal | Root |
|---|---|---|
| **Thread contention** | High `cpu-usage` *or* high p99 latency, with `threadpool-queue-length > 0` and rising `threadpool-thread-count` | Threads waiting on the scheduler / on each other |
| **Lock contention** | `monitor-lock-contention-count` > a few hundred/sec | A specific lock serializing threads — see [synchronization bottlenecks](./synchronization-bottlenecks.md) |
| **CPU saturation** | `cpu-usage ≈ 100%` with low queue length | Algorithm or workload genuinely needs more cores |
| **I/O wait** | Latency rises but CPU is low; `current-requests` queueing | See [I/O saturation](./io-saturation.md) |

The signature of contention specifically is *idle CPU + queued work + climbing thread count*. If any of those three are missing, you are looking at a different problem.

## Quantitative signals

Sustained at peak load, you are contention-bound when, roughly:

- **`threadpool-queue-length > 0` for more than a few seconds**, with throughput flat.
- **`threadpool-thread-count` climbs by 1 every ~500 ms** — the hill-climbing algorithm growing the pool because work is queueing.
- **CPU utilization is well below 100%** (often 30–60%) while latency p99 climbs.
- **`monitor-lock-contention-count`** ticks up — at least one hot lock is also at play.
- **`time-in-jit` and `time-in-gc` are not elevated** — neither JIT nor GC is the bottleneck.

If `cpu-usage` is at the ceiling, you are not contention-bound — you are CPU-bound and need to either reduce per-request work or scale out cores.

## Measurement recipe

1. **Live counters** — confirm the shape of the problem:
   ```bash
   dotnet-counters monitor System.Runtime --process-id <pid>
   ```
   Watch `threadpool-thread-count`, `threadpool-queue-length`, `cpu-usage`, `monitor-lock-contention-count`. Hosting counters help too:
   ```bash
   dotnet-counters monitor Microsoft.AspNetCore.Hosting --process-id <pid>
   ```
2. **Trace contention events** — find the offender:
   ```bash
   dotnet-trace collect --process-id <pid> --providers Microsoft-DotNETCore-SampleProfiler,Microsoft-Windows-DotNETRuntime:0x4000:5
   ```
   The `0x4000` keyword (`ContentionKeyword`) emits `ContentionStart`/`Stop` with the blocking stack. View in PerfView (*Thread Time → Lock-Cause Stacks*) or convert to Speedscope.
2. **Sync-over-async detection** — open the trace in PerfView and look for `Wait` / `WaitAny` frames inside ASP.NET request stacks, or `Task.GetAwaiter().GetResult` / `Task.Result` getter — those are the contention generators.
3. **Live dump** — when a starvation incident is in progress:
   ```bash
   dotnet-dump collect --process-id <pid>
   dotnet-dump analyze <dump-file>
   > clrstack -all
   > syncblk
   ```
   Many threads sitting in `Monitor.Wait` / `WaitOne` / `Task.Wait` is the smoking gun.

## The usual suspects

Roughly in order of how often they appear in real services:

- **Sync-over-async.** `task.Result`, `task.Wait()`, `task.GetAwaiter().GetResult()`. Each blocked call burns one worker thread for the duration of the I/O. Under burst, the pool grows by ~2/sec via hill-climbing — far slower than the request rate. Result: queue depth balloons, latency tails explode. Cure: `await` the whole way. There is **no acceptable use of `.Result`** on an async method in a request path.
- **`async void`** outside event handlers. Exceptions tear down the process and the continuation has no `Task` to be observed. Use `async Task` and let the caller decide.
- **`Task.Run` over async.** `await Task.Run(() => SomeAsync())` schedules a worker that immediately yields — pure overhead. Just `await SomeAsync()`.
- **Hot-path locks.** `lock(_state)` inside per-request code paths. Even when the critical section is microseconds, contention scales nonlinearly with concurrency. See [synchronization bottlenecks](./synchronization-bottlenecks.md).
- **`MinThreads` too low for burst load.** The default `MinWorkerThreads` is the processor count; under a 10× burst, hill-climbing takes seconds to grow the pool. Services with predictable burst patterns should call `ThreadPool.SetMinThreads` at startup — but this is a band-aid; the real fix is removing blocking work.
- **Long synchronous startup in singletons.** A DI singleton's constructor or `OnConfiguring` doing 200 ms of synchronous work blocks the first N requests. Use `IHostedService.StartAsync` for startup work, or `Lazy<Task<T>>` for lazy initialization.
- **`Parallel.ForEach` with synchronous lambda performing I/O.** Each iteration grabs a worker; combined with sync-over-async this saturates the pool. Use `Parallel.ForEachAsync` (.NET 6+) and async lambdas.
- **`ConfigureAwait(false)` on a desktop / mobile UI app**, but **omitting it in libraries** that run inside a synchronization context — the continuation hops back to a captured context, serializing through it. Library code should default to `ConfigureAwait(false)`.

## The fix toolkit

| Symptom | Tool | Notes |
|---|---|---|
| Sync wrapper around async API | Refactor caller to async; `IAsyncEnumerable<T>` for streaming | Top-down — never push the wrapper deeper |
| Producer/consumer behind a `lock(_queue)` | `Channel<T>` (bounded, `BoundedChannelFullMode.Wait`) | See [Channel&lt;T&gt;](./channel-of-t.md) |
| Hot counter under `lock` | `Interlocked.Increment` / `Add` | Single word, lock-free |
| Read-mostly map under `lock` | `FrozenDictionary<,>` (read-only) or `ConcurrentDictionary<,>` (read/write) | `FrozenDictionary` is faster on lookup, immutable after build |
| Bursty cold-start latency | `ThreadPool.SetMinThreads(workers, completionPort)` | Treat as a tactical fix while you remove blocking |
| Per-request gate (e.g. concurrency limit on a downstream) | `SemaphoreSlim` with `WaitAsync(ct)` | **Never** `Wait()` — that's sync-over-async |
| Coordinated cancellation across many awaits | One `CancellationTokenSource` plumbed through | Cancellation removes orphaned waits, not just running work |

A worked example — the textbook starvation pattern:

```csharp
// Before: blocks a worker thread per request; pool grows, then queues.
[HttpGet("/orders/{id:guid}")]
public IActionResult GetOrder(Guid id)
{
    var order = _service.GetOrderAsync(id).Result; // sync-over-async
    return Ok(order);
}

// After: zero workers blocked; pool size stays flat under load.
[HttpGet("/orders/{id:guid}")]
public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
{
    var order = await _service.GetOrderAsync(id, ct);
    return Ok(order);
}
```

`dotnet-counters` deltas observed on a service after this single change at 1k rps:

| Counter | Before | After |
|---|---:|---:|
| `threadpool-thread-count` | 220 (climbing) | 18 (flat) |
| `threadpool-queue-length` | 40+ | 0 |
| `cpu-usage` | 35% | 32% |
| `request-duration` p99 | 1100 ms | 38 ms |

Same hardware. Same downstream. The CPU was never the limit.

## Is it actually contention?

Cross-check against three alternatives before optimizing:

- **CPU bound** — flame graph time in user code, `cpu-usage` ≈ 100%. Optimize the algorithm, not the threading.
- **GC bound** — `% time in gc > 20%`. See [GC pressure](./gc-pressure.md).
- **I/O bound** — `current-requests` rising on a downstream call counter while CPU is idle and *no work is queued*. See [I/O saturation](./io-saturation.md).

Contention's signature is the *combination* of idle CPU + queued work + a climbing pool count. Two out of three is usually a different problem.

**Senior-level gotchas:**
- **`ThreadPool.SetMinThreads` is not a fix; it's a delay.** It papers over hill-climbing's slow ramp, but every blocked worker still has zero throughput. Profile and remove the blocking call.
- **`async void` does not throw to the caller.** It crashes the process via the unhandled-exception event. Use it only for true event-handler signatures, never for "fire and forget".
- **`Task.Wait` inside a `lock` is a deadlock waiting to happen.** If the continuation needs the lock you already hold, the worker that runs it will block forever. Don't mix locks with awaits.
- **`ConfigureAwait(false)` does not "speed up" code.** It avoids capturing a synchronization context that doesn't exist on a server anyway. ASP.NET Core has no `SynchronizationContext`, so it's a no-op there — keep using it in *libraries* that may also be hosted in WPF/WinForms/MAUI.
- **`ValueTask`'s thread-pool footprint can be worse than `Task`** if you await the same `ValueTask` twice or await one returned from a state machine that can complete synchronously *and* asynchronously — only use `ValueTask` for hot, mostly-sync code paths.
- **Hill climbing is global.** A starvation event in one tenant's request path drains the pool for every other tenant's. There is no per-endpoint thread pool unless you build one (e.g. dedicated `Channel`-backed worker).
- **`Task.Yield()` does not "release" the thread to do other work** — it queues a continuation. In a fully async pipeline, sprinkling `Yield()` is cargo-cult; if the path needs cooperative yielding, change it to actually await something.
- **Diagnostics tools count differently.** PerfView's "Lock-Cause Stacks" attributes time to the *acquirer*, not the *holder*. If the same lock is held briefly by many threads, the holder is the cause; the acquirers are symptoms.
- **`HttpClient` calls do not block worker threads** when fully awaited — they use the IOCP completion path. Sync-over-async on `HttpClient` is what creates the worker block.
- **A "single slow downstream" can starve the pool indirectly.** Async calls to it queue requests; the requests hold response bodies, captures, state machines — heap pressure rises, GC runs, *that* pause stalls workers. Symptom looks like contention; root is downstream timeout. Plumb a `CancellationToken` and a per-request timeout on the `HttpClient`.
