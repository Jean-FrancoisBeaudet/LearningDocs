# `ManualResetEvent` and `AutoResetEvent`

_Targets .NET 10 / C# 14. See also: [Semaphore, Mutex, Monitor](./semaphore-mutex-monitor.md), [TaskCompletionSource & CancellationTokenSource](../ASYNC_PROGRAMMING/cancellationtokensource-and-taskcompletionsource.md), [Channels](../ASYNC_PROGRAMMING/channels.md)._

Both are kernel-mode `WaitHandle`s — they signal *threads*, they don't guard data. Use them when one thread needs to tell others "go" or "stop blocking and check," especially across managed/unmanaged or named cross-process boundaries. In modern async code they are almost always the wrong primitive: `TaskCompletionSource<T>`, `Channel<T>`, or `CancellationToken` express the same patterns without a kernel transition per wait.

## `ManualResetEvent` — the gate

`Set()` opens the gate; **all** waiters pass through. The gate stays open until `Reset()`.

```csharp
private readonly ManualResetEvent _initialized = new(initialState: false);

public void Initialize()
{
    LoadConfig();
    _initialized.Set();         // every WaitOne caller proceeds, now and later
}

public void DoWork()
{
    _initialized.WaitOne();     // blocks until Set
    Process();
}
```

Pattern: one-shot initialization broadcast, "service is ready" flag, run-to-completion gates.

## `AutoResetEvent` — the turnstile

`Set()` releases **exactly one** waiter and auto-resets. If no thread is waiting, the next single `WaitOne` passes through and resets immediately. A second `Set()` before any waiter arrives is *not* counted — the signal is dropped.

```csharp
private readonly AutoResetEvent _itemReady = new(initialState: false);
private readonly Queue<Item> _queue = new();
private readonly Lock _qGate = new();

public void Enqueue(Item it)
{
    lock (_qGate) _queue.Enqueue(it);
    _itemReady.Set();           // wake one consumer
}

public void Consume(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        _itemReady.WaitOne();
        Item? it;
        lock (_qGate) it = _queue.TryDequeue(out var x) ? x : null;
        if (it is not null) Process(it);
    }
}
```

Pattern: hand-rolled producer/consumer, single-item handoff. In modern code, prefer `Channel<T>` — it gives you cancellation, async wait, and bounded backpressure for free.

## The "slim" variants

`ManualResetEventSlim` spins for a configurable count, then falls back to a kernel handle:

```csharp
var mre = new ManualResetEventSlim(initialState: false, spinCount: 100);
```

It exposes a lazily-allocated `WaitHandle` for interop, but if you only ever call `Wait`/`Set`/`Reset` no kernel handle is created — fully user-mode for short waits. There is **no `AutoResetEventSlim`**; use `SemaphoreSlim(0, 1)` (which has `WaitAsync` too) or a `Channel<T>` of capacity 1.

## Async equivalents — there is no `AsyncManualResetEvent` in the BCL

Build it from `TaskCompletionSource`:

```csharp
public sealed class AsyncManualResetEvent
{
    private TaskCompletionSource _tcs =
        new(TaskCreationOptions.RunContinuationsAsynchronously);

    public Task WaitAsync(CancellationToken ct = default) =>
        ct.CanBeCanceled ? _tcs.Task.WaitAsync(ct) : _tcs.Task;

    public void Set() => _tcs.TrySetResult();

    public void Reset()
    {
        while (true)
        {
            var current = _tcs;
            if (!current.Task.IsCompleted) return;

            var fresh = new TaskCompletionSource(
                TaskCreationOptions.RunContinuationsAsynchronously);
            if (Interlocked.CompareExchange(ref _tcs, fresh, current) == current)
                return;
        }
    }
}
```

`RunContinuationsAsynchronously` is **not optional** — without it, `TrySetResult` runs awaiting continuations inline on the calling thread, which can deadlock the setter or smuggle work onto threads that shouldn't run it.

For an async auto-reset, `SemaphoreSlim(0, 1)` is the standard answer.

`Nito.AsyncEx` (third-party) ships polished `AsyncManualResetEvent` / `AsyncAutoResetEvent` if you don't want to roll your own.

## Family members

- `CountdownEvent` — wait for **N** signals. Useful for fork-join with a known count when `Task.WhenAll` doesn't fit (e.g., signals from non-task code):

  ```csharp
  using var done = new CountdownEvent(workers);
  foreach (var w in pool) w.OnFinish += () => done.Signal();
  done.Wait();
  ```

- `Barrier` — phase synchronization for parallel algorithms; threads rendezvous at the end of each phase before any starts the next.
- `EventWaitHandle` — base class; supports **named** events for cross-process signaling:

  ```csharp
  using var ready = new EventWaitHandle(
      false, EventResetMode.ManualReset, @"Global\MyAppReady");
  ```

## `WaitHandle.WaitAll` / `WaitAny`

Static methods that block on multiple handles. Notes:
- `WaitAll` requires an **MTA** thread on Windows; calling it from STA throws `NotSupportedException`. Most service code is MTA, but desktop UI threads are STA.
- `WaitAny` returns the index of the signaled handle; if the signaled handle is an `AutoResetEvent`, only that one is consumed.
- All inputs must derive from `WaitHandle`; `ManualResetEventSlim.WaitHandle` lazily creates the kernel handle on first access.

## Comparison

| | `ManualResetEvent` | `AutoResetEvent` | `ManualResetEventSlim` | `SemaphoreSlim(0,1)` | TCS-based async event |
|---|---|---|---|---|---|
| Mode | gate (broadcast) | turnstile (one) | gate (broadcast) | turnstile (one) | gate, custom |
| Async API | No | No | No (`Wait` is sync) | **Yes** (`WaitAsync`) | **Yes** |
| Allocates kernel handle | Always | Always | Lazy | Lazy | Never |
| Cross-process | Via named `EventWaitHandle` | Via named `EventWaitHandle` | No | No | No |
| Disposable | Yes (handle) | Yes (handle) | Yes | Yes | n/a |
| Drops signals | No | **Yes** if no waiter | No | No (count clamps to 1) | No |

## Senior-level gotchas

- **`AutoResetEvent.Set()` called twice before any `WaitOne` releases only one waiter.** The second `Set` is a no-op. If you need "exactly N pending signals," use `Semaphore`/`SemaphoreSlim` or a `Channel<T>`.
- **`ManualResetEvent` and `AutoResetEvent` hold OS handles.** Long-lived ones are fine; per-request ones must be `Dispose()`d or you leak handles until the finalizer runs.
- **`TaskCompletionSource` without `RunContinuationsAsynchronously`** runs `await` continuations inline on the thread that calls `TrySetResult`. That thread may now execute arbitrary user code under whatever lock the setter holds. This is a classic deadlock cause; always set the flag.
- **`WaitHandle.WaitAll` requires MTA on Windows.** Desktop UI threads are STA — call `WaitAny` in a loop or marshal the wait to a worker thread.
- **`Slim` variants are in-process only.** `ManualResetEventSlim.WaitHandle` *creates* a kernel handle on demand for interop, but the slim object itself does not synchronize across processes.
- **Don't use kernel events as async signals.** Bridging `WaitOne` to `Task` via `ThreadPool.RegisterWaitForSingleObject` works but burns a thread-pool thread per registration callback — `TaskCompletionSource` or `Channel<T>` is strictly better.
- **`CountdownEvent.Signal()` past zero throws `InvalidOperationException`.** Reset it explicitly with `Reset(N)` if you reuse the instance, and don't share it across phases without coordination.
- **Named events are session-scoped by default.** Prefix with `Global\\` to make a name visible across user sessions; permissions and antivirus policies on Windows can block this.
- **`AutoResetEvent` does not guarantee FIFO ordering** of released waiters — it's whichever the kernel scheduler picks. Don't build fairness-critical pipelines on it.
- **The race "set right before wait" is fine on both event types** — `Set` updates handle state, and `WaitOne` checks state before blocking. The race "wait right before set" is fine too. The race that bites is "two `Set`s, one waiter" on `AutoResetEvent` — the second `Set` is silently dropped.
