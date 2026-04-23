# `async` / `await`

_Targets .NET 10 / C# 14. See also: [ConfigureAwait & SynchronizationContext](../ASYNC_PROGRAMMING/configureawait-and-synchronizationcontext.md), [ValueTask](../ASYNC_PROGRAMMING/valuetask.md), [Thread, ThreadPool, Task, TaskFactory](../ASYNC_PROGRAMMING/thread-threadpool-task-taskfactory.md), [Task continuation, cancellation, multiple tasks](../ASYNC_PROGRAMMING/task-continuation-cancellation-and-multiple-tasks.md), [using and lock](./using-and-lock.md)._

`async`/`await` is a **compiler transform**, not a runtime feature. Marking a method `async` rewrites it into a state machine, and `await` yields control via an awaiter. This note covers the shape of that transform, the awaiter pattern, and the per-call rules. Deeper-dive topics (synchronization context, ValueTask pooling, parallel patterns) live in `ASYNC_PROGRAMMING/` — links above.

## The shape of an `async` method

```csharp
public async Task<int> FetchLengthAsync(HttpClient http, string url, CancellationToken ct)
{
    var body = await http.GetStringAsync(url, ct).ConfigureAwait(false);
    return body.Length;
}
```

What the compiler produces (simplified):

1. A generated `struct` `<FetchLengthAsync>d__N` implementing `IAsyncStateMachine`. Locals become fields; `state` tracks which suspension point we're at.
2. An `AsyncTaskMethodBuilder<int>` field on the state machine — this is what drives the returned `Task<int>`.
3. The original method body is moved into `MoveNext()`, with `await` sites split into "schedule continuation, return" / "resume here" pairs.
4. The public method body becomes: allocate builder, start state machine (first synchronous chunk runs inline), return `builder.Task`.

So "async runs on a background thread" is **false**. By default, an async method runs synchronously on the calling thread until the first `await` of something not-yet-complete — at that point it returns to the caller and scheduling takes over.

### Fast path — already-completed awaitable

When the awaited thing is already complete at the `await` site (cached result, already-finished I/O), the state machine **doesn't suspend**: no continuation is scheduled, no box allocation, no thread-pool hop. This is why `ValueTask<T>` and the builder's "sync completion" optimization exist — to make that fast path allocation-free end-to-end.

## The awaiter pattern — duck-typed, not interface-based

Any type you can `await` needs to satisfy:

```csharp
// on the awaitable type:
TAwaiter GetAwaiter();

// on the awaiter:
bool IsCompleted { get; }
void OnCompleted(Action continuation);               // or UnsafeOnCompleted (ICriticalNotifyCompletion)
TResult GetResult();                                 // void or T
```

`Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `YieldAwaitable` (from `Task.Yield()`), `ConfigureAwaitable` (from `.ConfigureAwait(false)`) — all via this pattern. You can make your own type awaitable by providing a `GetAwaiter()` extension method.

```csharp
// Await a TimeSpan — because why not:
public static TaskAwaiter GetAwaiter(this TimeSpan ts) => Task.Delay(ts).GetAwaiter();

await TimeSpan.FromMilliseconds(50);     // works
```

## Return types

| Return type | Use |
|---|---|
| `Task` | async method with no result |
| `Task<T>` | async method returning `T` |
| `ValueTask` / `ValueTask<T>` | the awaited result is **usually** already available; avoids a `Task` allocation. See [ValueTask](../ASYNC_PROGRAMMING/valuetask.md). |
| `async void` | event handlers only — see below |
| Custom task-like types | with `[AsyncMethodBuilder(typeof(MyBuilder))]` — advanced |
| `IAsyncEnumerable<T>` | async streams — see [Async streams](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md) |

## `async void` — only for event handlers

```csharp
// ❌ fire-and-forget on a library method
async void Upload() { await DoUploadAsync(); }

// ✅ WinForms / WPF event handler — the framework expects a void signature
private async void Button_Click(object? sender, EventArgs e)
{
    try { await DoUploadAsync(); }
    catch (Exception ex) { ShowError(ex); }
}
```

Why the restriction:
- `async void` has no `Task` to await — caller can't know when it finished.
- Unobserved exceptions propagate to the **current `SynchronizationContext`** — on ASP.NET Core (no context) they can crash the process.
- No `await` in tests, no retry on failure, no cancellation propagation.

If you need fire-and-forget in a non-event context, return `Task` and explicitly `_ = DoAsync();` — at least the exceptions land on `TaskScheduler.UnobservedTaskException` where you can log them. Even better: use a background queue or `ChannelWriter<T>`.

## Exception behavior

```csharp
async Task Run()
{
    throw new InvalidOperationException("boom");
}

try { await Run(); }             catch (InvalidOperationException) { }   // ✅ caught unwrapped
try { Run().Wait(); }            catch (AggregateException ae)     { }   // wrapped
try { var r = Run().Result; }    catch (AggregateException ae)     { }   // wrapped
```

`await` unwraps the first exception; `.Result` / `.Wait()` give you `AggregateException`. This is one reason to never mix the two styles — either `await` everything or synchronously handle both wrappings.

If an async method throws **before its first `await`**, the returned `Task` is already-faulted — the caller still observes the exception via `await`, not on the outer `try` around the call. This is the source of the "why didn't my `ArgumentNullException` fire synchronously?" confusion.

### Early validation idiom

Same as iterators — wrap the async method:

```csharp
public Task<Account> GetAccountAsync(string id, CancellationToken ct)
{
    ArgumentException.ThrowIfNullOrEmpty(id);
    return GetAccountCore(id, ct);

    async Task<Account> GetAccountCore(string id, CancellationToken ct) { … await … ; return … ; }
}
```

Validation fires synchronously; the async work happens behind the awaited inner task.

## `ConfigureAwait` — libraries say `false`, app code usually doesn't care

```csharp
// Library code — decouples from caller's sync context:
await _http.GetAsync(url, ct).ConfigureAwait(false);
```

`ConfigureAwait(false)` tells the awaiter not to try to resume on the captured `SynchronizationContext`/`TaskScheduler`. Two consequences:

- In classic UI frameworks (WinForms, WPF, MAUI) the continuation may resume on the thread pool instead of the UI thread.
- In classic ASP.NET (`System.Web`) it avoided a deadlock + a small overhead.
- In **ASP.NET Core**, no `SynchronizationContext` is installed — `ConfigureAwait(false)` is a no-op. You can still add it in library code for the libraries' sake.

Full story: [ConfigureAwait & SynchronizationContext](../ASYNC_PROGRAMMING/configureawait-and-synchronizationcontext.md).

## Cancellation is part of the contract

```csharp
public async Task<Report> BuildAsync(CancellationToken ct)
{
    var data = await _repo.LoadAsync(ct).ConfigureAwait(false);
    ct.ThrowIfCancellationRequested();
    return Shape(data);
}
```

Rules:
- **Take `CancellationToken` as the last parameter**, named `cancellationToken` (or `ct` in this codebase).
- **Flow it through** every async API you call. Don't swallow it — don't `CancellationToken.None` it at a boundary.
- `OperationCanceledException` is the expected exit — don't swallow silently in top-level handlers; log-and-observe so you notice when a *non-requested* cancellation fires.
- Cancel with `CancellationTokenSource.CancelAsync()` (.NET 8+) when the cancellation callbacks themselves might await — it lets them run to completion.

## Composition patterns

```csharp
// Fan-out + fan-in: fail-fast on any failure
var results = await Task.WhenAll(urls.Select(u => http.GetStringAsync(u, ct)));

// First-complete: careful — losers keep running
var first = await Task.WhenAny(probes);

// Limit concurrency
using var sem = new SemaphoreSlim(4);
async Task Limited(string url)
{
    await sem.WaitAsync(ct);
    try { await http.GetStringAsync(url, ct); }
    finally { sem.Release(); }
}
```

`Task.WhenAll` aggregates exceptions into an `AggregateException` but `await` still unwraps to the first one — to see all, use `task.Exception.InnerExceptions` after the `await`.

More in [Task continuation, cancellation, and multiple tasks](../ASYNC_PROGRAMMING/task-continuation-cancellation-and-multiple-tasks.md).

## Performance notes

- **Every `await` that suspends** boxes the state machine (once) and allocates a `Task` (unless already cached, a singleton like `Task.CompletedTask`, or a `ValueTask` fast-path). Recent .NET versions have aggressive tiered compilation and pooling — measure before optimizing.
- **Avoid `async` when no awaits suspend**: if a method only ever returns synchronously, return a `Task` directly without `async` — saves the state-machine overhead:

    ```csharp
    public Task<int> AlwaysZeroAsync() => Task.FromResult(0);
    ```

- **`ValueTask<T>`** avoids the `Task` allocation when the result is already available — but can be awaited only once, so don't cache it. Covered in [ValueTask](../ASYNC_PROGRAMMING/valuetask.md).
- **Don't `.Result`/`.Wait()` on async code.** Best case you lose `ConfigureAwait` context; worst case you deadlock on a captured sync context (classic ASP.NET, UI).

## `await using` — async disposal

```csharp
await using var conn = new SqlConnection(cs);
await conn.OpenAsync(ct);
// conn.DisposeAsync() awaited on scope exit
```

See [using and lock](./using-and-lock.md). Don't mix sync `using` on a resource that only correctly closes via `DisposeAsync` (some network/crypto handles flush buffers in `DisposeAsync`).

## `await foreach` — async iteration

```csharp
await foreach (var line in ReadLinesAsync(path, ct)
                  .WithCancellation(ct)
                  .ConfigureAwait(false))
    Handle(line);
```

See [Async streams & `await foreach`](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md).

## Senior-level gotchas

- **Sync validation in an `async` method surfaces *on await*, not at call time.** If you need eager throws, wrap with a non-async outer method (iterator pattern).
- **`await` unwraps one exception; `.Wait()`/`.Result` don't.** If a `Task.WhenAll` faults with two exceptions, `await` gives you the first; inspect `task.Exception.InnerExceptions` to see all.
- **`async void` eats exceptions into the `SynchronizationContext`.** On ASP.NET Core (none), that usually means process crash. Never use outside event handlers.
- **Locks don't cross `await` boundaries.** `lock` forbids it outright; `Monitor.Enter` + `await` + `Monitor.Exit` deadlocks or errors. Use `SemaphoreSlim` for async mutual exclusion.
- **`ConfigureAwait(false)` is a no-op in ASP.NET Core**, but still write it in shared libraries — they may be consumed from UI apps.
- **Don't return `async Task` and then await internally for a value you could just pass through**: `public async Task<int> X() => await Y();` adds a state machine for no reason — `public Task<int> X() => Y();` is equivalent when there's no post-processing.
- **`Task.Run(() => …).Result` in a UI event handler is the canonical deadlock.** Solution: make the method `async` top to bottom; `async` is contagious for a reason.
- **`CancellationToken`s are value types** — cheap to pass, cheap to store. Propagate them; don't default-initialize a fresh one at a boundary.
- **`await`-ing a faulted/cancelled `ValueTask<T>` twice is undefined.** Await it once or convert to `Task` via `.AsTask()` if you must share it.
- **The state machine is a struct on the hot path; it moves to the heap on first suspension.** Minimize captures to shrink it (every captured local = another field).
