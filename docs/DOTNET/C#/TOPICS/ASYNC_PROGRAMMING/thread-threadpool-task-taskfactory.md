# Thread, ThreadPool, Task, TaskFactory

_Targets .NET 10 / C# 14. See also: [ConfigureAwait](./configureawait-and-synchronizationcontext.md), [ValueTask](./valuetask.md), [Task continuation, cancellation, and multiple tasks](./task-continuation-cancellation-and-multiple-tasks.md)._

The four APIs form a layered stack. From lowest to highest level:

| API | Abstraction | Typical use in 2026 |
|---|---|---|
| `Thread` | A dedicated OS thread. | Almost never. Interop with thread-affine native code, STA for COM, very long-lived blocking loops you explicitly do *not* want on the pool. |
| `ThreadPool` | A managed pool of worker + IOCP threads, reused for short work items. | The substrate everything else runs on. Rarely queued directly — `Task.Run` is the idiomatic front door. |
| `Task` / `Task.Run` | A future + scheduling unit. | The default answer for "run this off the calling thread" or "represent an async result." |
| `TaskFactory.StartNew` | Legacy, configurable `Task` factory (TPL, pre-2012). | Almost never. The one surviving use is `TaskCreationOptions.LongRunning`. |

## `Thread`

```csharp
var t = new Thread(() => DoWork()) { IsBackground = true, Name = "poller" };
t.Start();
t.Join();   // blocks caller — don't do this from async code
```

Key facts:
- Every `new Thread(...)` allocates ~1 MB of stack on Windows x64 and consumes a kernel thread. Doing this per request is a reliable way to exhaust memory.
- `IsBackground = false` (the default) **will keep your process alive** after `Main` returns. Always set it for fire-and-forget worker threads, or use a `Task`.
- `Thread.Sleep` blocks a real OS thread. In async code, use `await Task.Delay(..., ct)`.
- The `ApartmentState` (STA/MTA) property is effectively Windows-only and matters for COM / old WinForms/WPF interop. Modern service code ignores it.

**When a `Thread` is still correct:** a single, process-lifetime, blocking native loop (e.g., reading from a hardware device in a driver-style SDK that has no async surface). For anything async-capable, prefer `Task` or a hosted `BackgroundService`.

## `ThreadPool`

The pool splits into **worker threads** (CPU-bound work, queued from `Task.Run`, `ThreadPool.QueueUserWorkItem`) and **completion port (IOCP) threads** (callbacks from async I/O). In .NET Core the pool also owns an injection/hill-climbing algorithm that grows the pool *slowly* — roughly one thread per ~500 ms of observed starvation.

```csharp
ThreadPool.GetAvailableThreads(out var worker, out var iocp);
ThreadPool.GetMinThreads(out var minWorker, out var minIocp);
```

**Thread-pool starvation** is the classic `.Result` / `.Wait()` bug: blocked threads don't come back to the pool, the injector adds new ones one at a time, latency spikes for ~500 ms per new thread until the pool catches up. Diagnose with `dotnet-counters monitor System.Runtime` (`ThreadPool Thread Count`, `ThreadPool Queue Length`).

```csharp
// Raise min threads only as a *last-resort* mitigation for cold-start bursts.
ThreadPool.SetMinThreads(workerThreads: 200, completionPortThreads: 200);
```

This is a workaround, not a fix. The fix is to remove the sync-over-async.

**Modern note (.NET 10):** the pool's blocking detection heuristics have improved but the fundamental rule still stands — never block a pool thread on async work. Also note `System.Threading.Tasks.Dataflow` and **`Channel<T>`** are the idiomatic replacements for `ThreadPool.QueueUserWorkItem` producer/consumer patterns.

## `Task` and `Task.Run`

`Task` represents a future (optionally with a `TResult`). `Task.Run` queues a delegate to the default `TaskScheduler` (the thread pool) and returns a `Task` for it.

```csharp
// CPU-bound work off the calling thread:
var result = await Task.Run(() => Hash(bytes), ct);

// Async delegate: Task.Run unwraps Task<Task<T>> → Task<T> automatically.
var count = await Task.Run(async () =>
{
    await using var conn = await pool.RentAsync(ct);
    return await conn.CountAsync(ct);
}, ct);
```

Rules of thumb:
- **Do not** `Task.Run` purely async I/O in library code — you're paying a thread-pool hop to add zero parallelism. Let the `async` method run on the caller's context.
- **Do** `Task.Run` in UI apps to move CPU work off the UI thread.
- In ASP.NET Core, `Task.Run` in a request handler is almost always wrong — the request thread is already a pool thread, and you're just adding a queue hop.

## `TaskFactory.StartNew`

```csharp
// DON'T write this.
var t = Task.Factory.StartNew(async () => await FooAsync(),
                              TaskCreationOptions.None,
                              TaskScheduler.Default);
// t is Task<Task>, not Task — await t only waits for the OUTER task.
```

Why it's a footgun:
- Default scheduler is **`TaskScheduler.Current`**, not `.Default`. Inside a UI dispatcher or a nested task, that can mean you schedule onto the UI thread by accident.
- Does **not** unwrap async delegates → you get `Task<Task<T>>` and silently await only the outer task. `Task.Run` does unwrap.
- Default `TaskCreationOptions` enable attached children, so an inner `StartNew` without `DenyChildAttach` can silently propagate into the parent's completion/exception aggregate.

The one remaining legitimate use is explicit long-running work that should **not** sit on a normal pool thread:

```csharp
Task.Factory.StartNew(
    () => BlockingConsumerLoop(ct),
    ct,
    TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default);
```

`LongRunning` hints the scheduler to spin up a dedicated thread rather than borrow a pool thread for the entire lifetime.

### `Task.Run` vs `TaskFactory.StartNew` at a glance

| Concern | `Task.Run` | `TaskFactory.StartNew` (defaults) |
|---|---|---|
| Scheduler | `TaskScheduler.Default` | `TaskScheduler.Current` |
| Unwraps async delegate | **Yes** | No — returns `Task<Task>` |
| Attached children | `DenyChildAttach` | Attached-if-possible |
| Options | Fixed | Configurable (`LongRunning`, etc.) |
| Idiomatic since | .NET 4.5 | Pre-`async/await` TPL |

Translation: use `Task.Run` always, except the one `LongRunning` case above.

## Modern replacements worth knowing

- `PeriodicTimer` + `BackgroundService` replaces the old "spin a dedicated thread and `Thread.Sleep` in a loop" pattern.
- `Channel<T>` replaces hand-rolled producer/consumer with `ThreadPool.QueueUserWorkItem`.
- `Parallel.ForEachAsync` (see [parallel-class.md](./parallel-class.md)) replaces `Task.WhenAll` over a bounded `Task.Run` fan-out when you need a degree-of-parallelism knob.
- `Task.Run` + `CancellationToken` is the idiomatic "new thread" for 99% of cases — interview answers that still say `new Thread(...)` in 2026 are a red flag.

**Senior-level gotchas:**
- `Task.Run(async () => ...)` unwraps; `Task.Factory.StartNew(async () => ...)` does **not** — the difference has caused production incidents where exceptions from the inner task were silently swallowed.
- A **task is not a thread**. Ten thousand `Task.Delay` tasks cost a few KB, not 10 GB of stacks. Blocking inside those tasks is what turns them back into thread costs.
- `ThreadPool.SetMinThreads` is a foot-gun: bumping it masks real async bugs and increases context-switch pressure. Fix the blocking code, don't raise the floor.
- `Thread.Abort` does not exist on .NET Core / 5+. Cooperative cancellation via `CancellationToken` is the only supported model — there is no "kill this thread" API and that is by design.
- `new Thread` threads do **not** flow `ExecutionContext` the same way as tasks. `AsyncLocal<T>` set on the caller may not be visible inside the thread delegate unless you capture it explicitly.
