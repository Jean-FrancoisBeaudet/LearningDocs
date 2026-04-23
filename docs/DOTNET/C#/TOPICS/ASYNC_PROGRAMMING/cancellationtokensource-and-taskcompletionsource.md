# CancellationTokenSource and TaskCompletionSource

_Targets .NET 10 / C# 14. See also: [Task continuation, cancellation, and multiple tasks](./task-continuation-cancellation-and-multiple-tasks.md), [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [ConfigureAwait](./configureawait-and-synchronizationcontext.md)._

`CancellationTokenSource` (CTS) and `TaskCompletionSource` (TCS) are the two "driver" primitives behind the async surface. `CancellationToken` is read-only; CTS is the thing that actually fires it. `Task` is read-only; TCS is the thing that actually completes it. Both exist so *library code* can produce values (or cancellations) that consumer code awaits.

## CancellationTokenSource

```csharp
using var cts = new CancellationTokenSource();
var token = cts.Token;

_ = Task.Run(() => DoWorkAsync(token));
await Task.Delay(TimeSpan.FromSeconds(3));
cts.Cancel();            // fires callbacks + flips token.IsCancellationRequested
```

The `Cancel` call:
1. Flips `IsCancellationRequested` to `true`.
2. Runs every registered callback **synchronously** on the calling thread, in LIFO order.
3. If any callback throws, the remaining callbacks still run, and `Cancel()` rethrows the first exception (or `AggregateException` if multiple). `Cancel(throwOnFirstException: false)` swallows.

### Timer-based cancellation

```csharp
// Cancel after a deadline, no manual timer.
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
var result = await FetchAsync(cts.Token);

// Extend or reset the deadline later:
cts.CancelAfter(TimeSpan.FromSeconds(5));
```

`CancelAfter` queues a `TimerQueueTimer` — cheaper than a manual `Timer`. Calling it again replaces the pending cancel, not appends.

### Linked tokens

```csharp
using var linked = CancellationTokenSource.CreateLinkedTokenSource(
    callerCt, timeoutCts.Token, shutdownCt);
await DoAsync(linked.Token);
```

Cancellation on **any** source fires the linked token. The .NET 8+ `params`-style overload handles N sources; the 2-source overload is optimized to avoid array allocation. **Always dispose** the linked CTS — otherwise its registration on each parent stays alive for the lifetime of the longest parent.

### Disposing properly

```csharp
// ❌ Leak: if the CTS lives long (e.g., per-connection), every
//    ct.Register(...) kept alive references all its callbacks.
var cts = new CancellationTokenSource();

// ✅ using/await using releases registrations and the internal timer.
using var cts = new CancellationTokenSource(timeout);
```

A CTS holds its registered callbacks' target references. For ambient, process-lifetime tokens (like `IHostApplicationLifetime.ApplicationStopping`) that is fine; for per-request/per-operation CTS, forgetting `Dispose` is a real leak.

### `ct.Register` — running code on cancellation

```csharp
await using var reg = ct.Register(static s => ((SocketConnection)s!).Abort(), connection);
await connection.ReceiveAsync(ct);
```

Rules:
- Callbacks run **synchronously** on the thread that calls `Cancel()`. Keep them short and non-blocking.
- Pass a `static` lambda + `state` object to avoid capturing `this`/closures; otherwise every registration allocates a closure.
- `Register` returns a `CancellationTokenRegistration` — **dispose it** when you no longer need the callback, especially if the token outlives the subscriber.

## TaskCompletionSource

`TaskCompletionSource<T>` is the other side of `Task<T>`: you hold a TCS, hand callers its `.Task`, and later call `SetResult` / `SetException` / `SetCanceled`. The classic use case is adapting a callback-style API into an awaitable.

```csharp
public static Task<string> ReadFromSocketAsync(Socket socket, CancellationToken ct)
{
    var tcs = new TaskCompletionSource<string>(
        TaskCreationOptions.RunContinuationsAsynchronously);

    ct.Register(() => tcs.TrySetCanceled(ct));

    socket.BeginReceive(
        buffer, 0, buffer.Length, SocketFlags.None,
        ar =>
        {
            try
            {
                var count = socket.EndReceive(ar);
                tcs.TrySetResult(Encoding.UTF8.GetString(buffer, 0, count));
            }
            catch (Exception ex) { tcs.TrySetException(ex); }
        },
        state: null);

    return tcs.Task;
}
```

### `RunContinuationsAsynchronously` — the #1 production gotcha

By default, a TCS completion runs its continuations **inline, synchronously, on the thread that called `SetResult`**. If that thread happens to be a network callback thread or a lock-holding thread, the awaiter's continuation runs there too — surprising the consumer and sometimes deadlocking.

```csharp
// ❌ Default (DO NOT do this in library code):
var tcs = new TaskCompletionSource<int>();

// ✅ Always use this flag unless you KNOW you want inline continuations.
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
```

With the flag, continuations queue to the thread pool (or the captured sync context) — the consumer's `await` resumes in the expected place. This has been the default advice since .NET Framework 4.6; it's still surprisingly common to see it omitted.

### `Set*` vs `TrySet*`

| API | Behavior when already completed |
|---|---|
| `SetResult` / `SetException` / `SetCanceled` | **Throws** `InvalidOperationException`. |
| `TrySetResult` / `TrySetException` / `TrySetCanceled` | Returns `false`. |

If there's **any** path where multiple sources could complete (cancellation racing a result, timeout racing success), use the `Try*` variant. Otherwise the first winner completes the task and the second winner throws. Both patterns are valid — pick based on whether a double-complete is a bug or a race.

### `SetCanceled(CancellationToken)`

Passing the token preserves the `OperationCanceledException.CancellationToken` property, so callers can tell *which* cancellation fired. Without the token overload, the property is `default(CancellationToken)`.

### Non-generic `TaskCompletionSource`

Pre-.NET 5, `TaskCompletionSource` was generic only; you'd use `TaskCompletionSource<object?>` for "no result." .NET 5 added a non-generic `TaskCompletionSource` with `Task`-not-`Task<T>` — use it for fire-and-signal scenarios.

```csharp
var ready = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
// ...
ready.TrySetResult();        // no value needed
await ready.Task;
```

## When to use which

| Need | Use |
|---|---|
| Your code needs to *request* cancellation of async operations. | `CancellationTokenSource` |
| You need to combine multiple cancellation sources. | `CancellationTokenSource.CreateLinkedTokenSource` |
| You're adapting a callback/event API to `Task`. | `TaskCompletionSource<T>` |
| You need a hot, `SetResult`-able task for testing. | `TaskCompletionSource<T>` |
| You want to pre-complete a task with a value. | `Task.FromResult` (not TCS) |
| You want a cancelled / faulted task synchronously. | `Task.FromCanceled` / `Task.FromException` (not TCS) |

## Signalling patterns

### One-shot "ready" signal between coroutines

```csharp
private readonly TaskCompletionSource _started =
    new(TaskCreationOptions.RunContinuationsAsynchronously);

public Task WaitForStartAsync(CancellationToken ct) => _started.Task.WaitAsync(ct);
public void SignalStarted() => _started.TrySetResult();
```

For repeated signals, don't reuse a TCS — they're single-use. Use `Channel<T>` or `SemaphoreSlim` instead.

### Bridging events

```csharp
public static Task<EventArgs> WaitForEventAsync(
    object source, string eventName, CancellationToken ct)
{
    var tcs = new TaskCompletionSource<EventArgs>(
        TaskCreationOptions.RunContinuationsAsynchronously);
    EventInfo ev = source.GetType().GetEvent(eventName)!;

    EventHandler handler = null!;
    handler = (_, e) => { ev.RemoveEventHandler(source, handler); tcs.TrySetResult(e); };
    ev.AddEventHandler(source, handler);

    ct.Register(() =>
    {
        ev.RemoveEventHandler(source, handler);
        tcs.TrySetCanceled(ct);
    });

    return tcs.Task;
}
```

`TaskAsyncEnumerableExtensions` and `EventAsTask`-flavored helpers in `Microsoft.Reactive` / `Rx.NET` save you from writing this yourself when the event is well-typed.

**Senior-level gotchas:**
- **Always** pass `TaskCreationOptions.RunContinuationsAsynchronously` to a new TCS in library code. Forgetting it has caused countless "why did my WebSocket callback deadlock" incidents.
- `CancellationTokenSource.Cancel()` runs callbacks **synchronously on the caller**. A slow or deadlock-prone callback registered by any consumer will freeze whoever called `Cancel` — including `CancelAfter`'s timer thread, which is a pool thread.
- `CancellationTokenSource.CancelAsync()` (.NET 8+) queues callbacks to the pool instead of running them inline. Prefer it when you hold locks during cancel or don't trust registered callbacks.
- A disposed CTS cannot cancel. Disposing early silently turns `Cancel()` into a no-op (or `ObjectDisposedException` in .NET 8+). Dispose after you're sure no one still needs to signal.
- `TaskCompletionSource<T>` can be `SetResult`-ed from any thread, but calling it while holding a lock and with sync continuations is a deadlock recipe — the awaiter's continuation can try to re-enter the lock. `RunContinuationsAsynchronously` fixes this by construction.
- `ct.Register` with a closure over `this` pins the registering object for the CTS's lifetime. For long-lived tokens with short-lived subscribers, you get a leak that only shows up under load — dispose the registration.
- A `Task` from `TaskCompletionSource` is hot: any `await` starts observing immediately. Returning `tcs.Task` from an `async` method then never calling `SetResult` leaks an awaitable that may live forever, tying up a continuation and any captured state.
- `TaskCompletionSource<T>.Task` is awaited like any other `Task<T>` — meaning you can `WhenAll`/`WhenAny` / `WaitAsync` it. That's how `Channel<T>`, `SemaphoreSlim.WaitAsync`, etc. are implemented under the hood.
