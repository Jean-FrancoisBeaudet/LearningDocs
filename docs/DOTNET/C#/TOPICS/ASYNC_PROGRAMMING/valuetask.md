# ValueTask

_Targets .NET 10 / C# 14. See also: [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [ConfigureAwait](./configureawait-and-synchronizationcontext.md)._

`ValueTask` / `ValueTask<T>` is a `readonly struct` that **may** wrap a `Task<T>`, a cached sync result, or an `IValueTaskSource<T>`. Its reason to exist: avoid allocating a `Task<T>` on the hot path when the async method frequently completes synchronously.

## The problem it solves

Every `async Task<T>` method that awaits allocates a `Task<T>` (plus a state-machine box on first suspension). For a method called millions of times that returns from cache 95% of the time — `Stream.ReadAsync`, `MemoryCache.GetOrCreateAsync`, `Channel.Reader.ReadAsync` — that's a **lot** of allocations for work that didn't really need a heap object.

```csharp
// Allocates a Task<int> every call, even when 'Length' is reached synchronously.
public async Task<int> ReadAsync(byte[] buf, CancellationToken ct) { ... }

// No allocation on the sync path; only allocates if the read actually yields.
public async ValueTask<int> ReadAsync(byte[] buf, CancellationToken ct) { ... }
```

## Mental model: a discriminated union

A `ValueTask<T>` is one of three things at runtime:
1. A synchronously-completed result (no allocation, just a `T` stored in the struct).
2. A wrapped `Task<T>` (allocated; identical cost to `Task<T>` plus the struct wrapper).
3. A wrapped `IValueTaskSource<T>` with a token — a reusable, pooled async state object.

The struct carries both the reference and a small token (`_token` + `_obj`). You never inspect these; you await the thing.

## The single-await rule

This is the rule most developers get wrong:

> A `ValueTask` / `ValueTask<T>` may be awaited **at most once**, and you must not call `.Result`, `.GetAwaiter().GetResult()`, `.AsTask()`, or any other consuming member more than once on the same instance.

Why: if it's backed by a pooled `IValueTaskSource<T>`, that source may be recycled after the first await. A second await could observe a different operation's result or hit an `InvalidOperationException`.

Safe consumption pattern when you need the result twice:

```csharp
// If you need to share / store the result, materialize to Task<T> immediately.
Task<int> task = _stream.ReadAsync(buf, ct).AsTask();
var a = await task;
var b = await task;   // fine — Task<T> is awaitable multiple times
```

Or call `.Preserve()` to get a non-poolable, multi-await-safe copy without the extra `Task<T>` allocation when the source was synchronous.

## When to return `ValueTask`

Return `ValueTask<T>` **only** when both are true:
1. The API is on a measurably hot path (benchmarked, not guessed).
2. A meaningful fraction of calls complete synchronously.

Otherwise return `Task<T>`. `Task<T>` is free for callers to use idiomatically — `WhenAll`, `WhenAny`, store in a field, await twice, `.ContinueWith`. `ValueTask<T>` is a performance primitive with a usage contract.

Canonical producers of `ValueTask`:
- `Stream.ReadAsync(Memory<byte>, ...)`, `PipeReader.ReadAsync`
- `Channel<T>.Reader.ReadAsync`, `WaitToReadAsync`
- `IAsyncEnumerable<T>.GetAsyncEnumerator().MoveNextAsync`
- `ValueTask<IMemoryOwner<byte>>` style pool rentals

## Consumption rules

```csharp
// ✅ Await it directly.
var n = await _stream.ReadAsync(buf, ct);

// ✅ If you need it twice, store the Task<T>.
var t = _stream.ReadAsync(buf, ct).AsTask();

// ❌ NEVER do this.
var vt = _stream.ReadAsync(buf, ct);
if (vt.IsCompletedSuccessfully) { var x = vt.Result; } // OK — single consume
var y = await vt;                                       // BUG — second consume

// ❌ Do not pass ValueTask into Task.WhenAll.
await Task.WhenAll(vt1.AsTask(), vt2.AsTask());         // materialize first
```

The CA2012 analyzer catches most of these misuses in library code — turn it on.

## Building a custom `IValueTaskSource<T>` (rare, advanced)

For ultra-hot-path scenarios — typically high-throughput network code — you can implement `IValueTaskSource<T>` and pool the sources. This is how `System.Threading.Channels` and `PipeReader` achieve zero allocations on steady-state reads.

```csharp
sealed class PooledResult<T> : IValueTaskSource<T>
{
    private ManualResetValueTaskSourceCore<T> _core;  // struct helper from the BCL

    public ValueTask<T> AsValueTask() => new(this, _core.Version);

    public void SetResult(T value) => _core.SetResult(value);
    public void SetException(Exception ex) => _core.SetException(ex);

    T IValueTaskSource<T>.GetResult(short token)
    {
        try { return _core.GetResult(token); }
        finally { _core.Reset(); /* return to pool */ }
    }
    ValueTaskSourceStatus IValueTaskSource<T>.GetStatus(short token) => _core.GetStatus(token);
    void IValueTaskSource<T>.OnCompleted(Action<object?> continuation, object? state,
        short token, ValueTaskSourceOnCompletedFlags flags)
        => _core.OnCompleted(continuation, state, token, flags);
}
```

99% of user code will never write this. If you're reaching for it, benchmark with BenchmarkDotNet and a `[MemoryDiagnoser]` first — most "hot paths" don't need it.

## Allocation costs — rough back-of-envelope

| Scenario | `Task<int>` allocation | `ValueTask<int>` allocation |
|---|---|---|
| Sync-complete, no state machine box | ~72 B (Task<int>) | 0 B |
| Async yields once, no pooling | ~72 B + box (~64 B) | ~72 B + box (wrapping Task) |
| Async via `IValueTaskSource` pool | n/a | 0 B steady state |

Numbers vary by runtime version; use `[MemoryDiagnoser]` benchmarks for the truth. The rule of thumb: **`ValueTask` pays off when the sync-complete path dominates.**

## `ValueTask` vs `Task` — quick decision

- Returning from an interface that already returns `Task<T>`? **Don't change it** unless you own all callers. You'll break binary compatibility and possibly miscompile `WhenAll` callers.
- Building a new high-throughput primitive with obvious sync-fast-path? Use `ValueTask<T>`.
- Writing a business-logic service method? `Task<T>` every time. The complexity is not worth it.

**Senior-level gotchas:**
- `ValueTask.FromResult` allocates nothing for the sync path; `Task.FromResult` caches common values (`0`, `true`, `false`) but allocates for arbitrary `T`. For a "return 42" async method, both are fine; the difference is only visible at millions of calls/sec.
- `ValueTask` is a `readonly struct`. Copying it around is cheap but **don't copy after the first await** — the token snapshot becomes stale and you'll trip the single-await rule from a seemingly innocent variable assignment.
- `async ValueTask<T>` methods still allocate a state-machine box if they yield. The win is on the sync path only — if your method **always** yields, you're paying for the struct wrapping with zero gain.
- `ValueTask` doesn't have a `WhenAll` equivalent. Call `.AsTask()` on each and use `Task.WhenAll`, or build a hand-rolled awaiter — there is no free lunch for composition.
- An unobserved faulted `ValueTask` does **not** raise `TaskScheduler.UnobservedTaskException` (no Task exists until `.AsTask()` is called). Silently discarded errors are a real risk — always await or `.AsTask()` and observe.
- `ConfigureAwait(false)` exists on `ValueTask` but only as the `bool` overload; the richer `ConfigureAwaitOptions` is `Task`-only.
