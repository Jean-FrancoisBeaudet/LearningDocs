# Task Continuation, Cancellation, and Executing Multiple Tasks

_Targets .NET 10 / C# 14. See also: [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [ConfigureAwait](./configureawait-and-synchronizationcontext.md), [CancellationTokenSource and TaskCompletionSource](./cancellationtokensource-and-taskcompletionsource.md)._

Three topics, one file, because they compose. Once you have concurrent tasks, you need to chain work (continuations), cancel them (tokens), and wait on them as a group (`WhenAll`/`WhenAny`).

## Continuations — prefer `await` to `ContinueWith`

Pre-`async/await`, chaining work meant `ContinueWith`:

```csharp
// Legacy — don't write new code like this.
Task.Run(() => Parse(input))
    .ContinueWith(t => Save(t.Result),
        CancellationToken.None,
        TaskContinuationOptions.OnlyOnRanToCompletion,
        TaskScheduler.Default);
```

In modern code, a `try`/`catch` + `await` expresses the same intent with real control flow:

```csharp
try
{
    var parsed = await Task.Run(() => Parse(input), ct);
    await SaveAsync(parsed, ct);
}
catch (OperationCanceledException) { /* cancelled */ }
catch (FormatException ex)         { _log.LogWarning(ex, "bad input"); }
```

Reasons to prefer `await`:
- Exceptions surface as the concrete type, not wrapped in `AggregateException`.
- Control flow lives in the method, not in scheduler options.
- `ContinueWith` defaults to `TaskScheduler.Current` — the same footgun as `StartNew`. You need an explicit scheduler every time.

`ContinueWith` still has niche uses: building primitives, framework-level bookkeeping that must not `await`, or bolting on logging to tasks you don't own. `TaskContinuationOptions` flags worth remembering:

| Flag | Meaning |
|---|---|
| `OnlyOnRanToCompletion` | Fire only on success. |
| `OnlyOnFaulted` | Fire only on exception. |
| `OnlyOnCanceled` | Fire only on OCE. |
| `NotOnRanToCompletion` | Fire on fault/cancel only. |
| `ExecuteSynchronously` | Run inline on the completing thread — dangerous, can deadlock. |
| `DenyChildAttach` | Don't become an attached child of the ambient task. |

## `WhenAll` — run many, await all

```csharp
var tasks = ids.Select(id => LoadAsync(id, ct));
Item[] items = await Task.WhenAll(tasks);
```

Behavior:
- Starts **nothing** on its own — tasks must already be running (they are, if they're hot tasks from `async` methods).
- Returns a `Task<T[]>` that completes when **all** input tasks complete.
- On cancellation or fault, still waits for the others to finish — `WhenAll` never returns early.

### Exception aggregation

```csharp
try
{
    await Task.WhenAll(t1, t2, t3);
}
catch (Exception ex)
{
    // ex is ONE exception — the first one the awaiter saw.
    // To see all of them, inspect the Task's AggregateException:
    var aggregate = await Task.WhenAll(t1, t2, t3)
        .ContinueWith(t => t.Exception, TaskContinuationOptions.OnlyOnFaulted);
    // or simpler:
    Task combined = Task.WhenAll(t1, t2, t3);
    try { await combined; }
    catch { foreach (var inner in combined.Exception!.InnerExceptions) _log.LogError(inner, ""); }
}
```

The asymmetry surprises people: `await` unwraps the `AggregateException` to the first inner exception. The `Task.Exception` property holds the full aggregate. If you need all of them, inspect the completed `Task`.

### The fan-out/fan-in pattern

```csharp
const int MaxConcurrent = 8;
using var throttle = new SemaphoreSlim(MaxConcurrent);

var tasks = ids.Select(async id =>
{
    await throttle.WaitAsync(ct);
    try { return await LoadAsync(id, ct); }
    finally { throttle.Release(); }
});

var results = await Task.WhenAll(tasks);
```

For this exact pattern, `Parallel.ForEachAsync` (see [parallel-class.md](./parallel-class.md)) is cleaner and bounded-by-default.

## `WhenAny` — first to finish wins

```csharp
var finished = await Task.WhenAny(primary, fallback);
if (finished == primary) return primary.Result;
```

Common uses:
- **Timeout:** `await Task.WhenAny(t, Task.Delay(timeout, ct))`. Since .NET 6, prefer `t.WaitAsync(timeout, ct)`.
- **Redundant requests:** fire N providers, take the first response, cancel the rest.
- **Progress loop:** race a work task against a cancellation delay.

Pitfall: `WhenAny` does **not cancel** the losers. You must track them and either await-and-ignore or plumb a cancellation token to cut them off. Leaked tasks with unobserved exceptions end up in `TaskScheduler.UnobservedTaskException`.

```csharp
var winner = await Task.WhenAny(tasks);
var loserCts = ...;
loserCts.Cancel();          // stop the others
foreach (var t in tasks.Where(t => t != winner))
    _ = t.ContinueWith(_ => { }, TaskContinuationOptions.OnlyOnFaulted); // observe
```

## `WaitAsync` (.NET 6+)

```csharp
// Timeout:
var result = await FetchAsync(ct).WaitAsync(TimeSpan.FromSeconds(5), ct);

// Extra cancellation token:
var result = await FetchAsync(ct).WaitAsync(otherCt);
```

`WaitAsync` does **not** cancel the underlying task — it stops **waiting** for it. The underlying task keeps running and must observe its own cancellation to actually stop. This is the correct semantic (you can't abort arbitrary code), but surprises developers expecting `WaitAsync` to kill the work.

## Cancellation — the cooperative contract

`CancellationToken` is a **request**. Code inside the task must poll or throw on it:

```csharp
async Task<int> ComputeAsync(IEnumerable<int> data, CancellationToken ct)
{
    int total = 0;
    foreach (var chunk in data)
    {
        ct.ThrowIfCancellationRequested();
        total += await ProcessAsync(chunk, ct);
    }
    return total;
}
```

Plumb the token through every async call. Analyzers **CA2016** / **CA2024** flag missed propagation.

### `OperationCanceledException` vs `TaskCanceledException`

`TaskCanceledException : OperationCanceledException`. Always catch the base:

```csharp
try
{
    await FetchAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // Distinguish "we were asked to cancel" from "upstream cancelled due to timeout".
}
```

### Linking tokens

```csharp
using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, timeoutCts.Token);
var data = await FetchAsync(linked.Token);
```

Cancellation on *either* source cancels the linked token. `CreateLinkedTokenSource` in .NET 8+ supports up to `N` sources; the 2-source overload is optimized to not allocate the array.

### Cancelling with a timeout the easy way

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
var data = await FetchAsync(cts.Token);
```

Pair with `ct` from the caller via `CancellationTokenSource.CreateLinkedTokenSource` if the method should honor both.

### Unregistering

```csharp
var reg = ct.Register(() => conn.Abort());
try { await DoWorkAsync(ct); }
finally { await reg.DisposeAsync(); }
```

Forgetting to dispose the registration is a quiet memory leak in long-lived tokens (e.g., one `CancellationTokenSource` used across many operations).

## Two tasks in parallel — the common idiom

```csharp
var userTask  = LoadUserAsync(id, ct);
var ordersTask = LoadOrdersAsync(id, ct);
await Task.WhenAll(userTask, ordersTask);
var user = await userTask;
var orders = await ordersTask;
```

Starting both then awaiting together gives you concurrent I/O. Awaiting each in sequence (`var user = await ...; var orders = await ...`) serializes them and is a very common junior-level bug.

**Senior-level gotchas:**
- `Task.WhenAll(tasks)` with a `List<Task>` that's still being mutated is a race. Materialize first: `var arr = tasks.ToArray();`.
- `Task.WhenAny` returns the **first completed** task, which may have faulted. Always check the result before using it — don't assume it succeeded.
- Cancellation is **not instantaneous**. A token fires, the code polls it, throws, the state machine unwinds, finally blocks run. "Cancelled at t=5s" can finish at t=5.2s or later; sensitive tests need to tolerate this.
- `Task.Delay(TimeSpan.FromSeconds(30), ct)` is lighter than a `new Timer` and cancellable. Use it everywhere you'd reach for a timer.
- `ct.Register(callback)` callbacks run **synchronously** on whatever thread cancels the source — if your callback blocks or throws, you block the canceller. Keep them short and safe.
- `Task.Run(() => ..., ct)` only honors `ct` *before* the delegate starts executing. Once it's running, only the delegate's own polling of `ct` will stop it. A common bug: passing `ct` to `Task.Run` and expecting it to abort the running code.
- `ContinueWith` continuations run regardless of whether the antecedent's exception was observed anywhere else. Adding a `ContinueWith(t => log(t.Exception))` is not enough — you need to **access** `t.Exception` for the BCL to mark it observed.
- `WhenAll` over an empty collection returns `Task.CompletedTask` immediately. Don't branch on `tasks.Any()` defensively — the BCL already handles it.
