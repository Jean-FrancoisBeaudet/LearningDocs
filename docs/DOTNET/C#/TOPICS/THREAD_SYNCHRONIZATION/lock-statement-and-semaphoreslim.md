# `lock` statement and `SemaphoreSlim`

_Targets .NET 10 / C# 14. See also: [`using` & `lock` (basics)](../CONSTRUCTS/using-and-lock.md), [Semaphore, Mutex, Monitor](./semaphore-mutex-monitor.md), [`volatile` & `Interlocked`](./volatile-and-interlocked.md), [Thread, ThreadPool, Task](../ASYNC_PROGRAMMING/thread-threadpool-task-taskfactory.md)._

Two non-interchangeable mutual-exclusion primitives. `lock` is **synchronous, reentrant, thread-affine** and the right answer for guarding shared in-memory state in non-async code. `SemaphoreSlim` is **async-aware, non-reentrant, not thread-affine** and the right answer for serializing across `await` or for bounded concurrency. Picking the wrong one gives you either a compiler error (`lock` body cannot `await`) or a deadlock (recursive `SemaphoreSlim.WaitAsync`).

The basics of the `lock` statement are covered in [`using` & `lock`](../CONSTRUCTS/using-and-lock.md); this note goes deeper on `Monitor` internals, the `System.Threading.Lock` type, and the full `SemaphoreSlim` surface.

## `lock` revisited — Monitor under the hood

```csharp
private readonly Lock _gate = new();   // .NET 9+: System.Threading.Lock
private int _counter;

public void Increment()
{
    lock (_gate) { _counter++; }
}
```

Lowering before C# 13 emits `Monitor.Enter(gate, ref taken)` / `Monitor.Exit(gate)` in a `try/finally`. Each lockable object carries a hidden **sync block index** in its object header; the first `Monitor.Enter` either uses a thin lock (CAS into the header) or, on contention, "inflates" to a full sync block referencing a kernel event. Reentrancy is implemented by tracking the owning thread ID and a recursion count inside that sync block.

In .NET 9+, declaring the field as `System.Threading.Lock` makes the compiler emit `Lock.EnterScope()` calls returning a `Lock.Scope` ref struct. Benefits:
- ~2–3× faster on the uncontended path; no header inflation, no sync-block allocation on the target object.
- Type-safe — you can't accidentally `Monitor.Wait`/`Pulse` on it or use it as a generic `object` lock.
- The compiler warns (`CS9216`) when a `Lock`-typed value is implicitly converted to `object`, which would silently route through the legacy `Monitor` path.

`Lock` is still **reentrant** and **thread-affine**, just like `Monitor`.

## Why `lock` and `await` are incompatible

```csharp
lock (_gate)
{
    await DoAsync();   // CS1996: cannot await in lock statement body
}
```

The compiler rejects it because `Monitor` ownership is bound to the entering OS thread. After the `await`, the continuation may resume on a different pool thread; releasing a `Monitor` from a thread that never acquired it throws `SynchronizationLockException` — and even if it didn't, the critical section was abandoned at the `await` point on the original thread. There is no fix; choose a primitive whose ownership concept is logical, not thread-affine.

## `SemaphoreSlim` — the async-safe alternative

A counting semaphore. Two common shapes:

**Binary mutex** (`new SemaphoreSlim(1, 1)`) — the async equivalent of `lock`:

```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task<T> ReadAsync(CancellationToken ct)
{
    await _gate.WaitAsync(ct).ConfigureAwait(false);
    try
    {
        return await _store.LoadAsync(ct).ConfigureAwait(false);
    }
    finally
    {
        _gate.Release();
    }
}
```

**Counting semaphore** — bounded concurrency for downstream calls:

```csharp
private readonly SemaphoreSlim _slots = new(initialCount: 8, maxCount: 8);

public async Task FetchAsync(Uri url, CancellationToken ct)
{
    await _slots.WaitAsync(ct).ConfigureAwait(false);
    try     { await _http.GetAsync(url, ct).ConfigureAwait(false); }
    finally { _slots.Release(); }
}
```

The `try/finally` is mandatory — unlike `lock`, the compiler does not emit it for you. Forget the `Release` and the next caller waits forever.

## Cancellation, fairness, FIFO

- `WaitAsync(CancellationToken)` registers a continuation; cancellation throws `OperationCanceledException` and **does not** consume a permit.
- `WaitAsync(TimeSpan, CancellationToken)` returns `false` on timeout; again no permit is consumed.
- The implementation **usually** behaves FIFO but the contract does not guarantee it. Don't build fairness-critical schedulers on it; use a `Channel<T>` or roll a queue if order matters.
- `AvailableWaitHandle` exposes a kernel `WaitHandle` for interop with code that expects `WaitHandle.WaitAny` etc. — accessing it lazily allocates the handle, so don't touch it just to inspect availability.

## Reentrancy — the thing that bites everyone

`SemaphoreSlim` has no concept of "the current owner." If the same logical flow re-enters its own guarded method:

```csharp
public async Task DoAsync(CancellationToken ct)
{
    await _gate.WaitAsync(ct);
    try
    {
        await DoAsync(ct);   // ❌ deadlock — waits for the permit it already holds
    }
    finally { _gate.Release(); }
}
```

Two ways out:
1. **Restructure** so the recursion happens *outside* the lock. Extract the protected work into a private method that assumes the permit is already held.
2. **Track ownership yourself** with `AsyncLocal<bool>` for "already inside" — only correct in fully-async flows, and a smell that the design is wrong.

## Performance — `lock` vs `SemaphoreSlim(1,1)`

Rough order-of-magnitude on .NET 9, x64:

| Path | `lock` (`Monitor`) | `Lock` (.NET 9+) | `SemaphoreSlim(1,1)` sync `Wait` | `SemaphoreSlim(1,1)` `WaitAsync` (uncontended) |
|---|---|---|---|---|
| Uncontended | ~20 ns | ~10 ns | ~50 ns | ~80 ns + state-machine alloc |
| Contended | μs (kernel wait) | μs (kernel wait) | μs | μs + Task alloc per waiter |

Use `SemaphoreSlim` for async exclusion and only there; pick `Lock`/`lock` for sync paths.

## Comparison

| | `lock` (`Monitor`) | `Lock` (.NET 9+) | `SemaphoreSlim(1,1)` |
|---|---|---|---|
| Reentrant | Yes | Yes | **No** |
| Async-aware | No | No | **Yes** (`WaitAsync`) |
| Cancellation | No | `TryEnter(timeout)` | **Yes** (`CancellationToken`) |
| Thread-affine | Yes | Yes | No (logical permit) |
| Fairness | None | None | Approx. FIFO (not contractual) |
| Allocation per acquire | None | None (ref-struct scope) | None on uncontended sync; `Task` on contended async |
| Cross-process | No | No | No (use named `Mutex`/`Semaphore`) |

## Senior-level gotchas

- **Forgetting `Release()` in `finally` after `WaitAsync` deadlocks the next caller forever.** No GC reclamation, no timeout, no diagnostic — the API surface puts the burden on you. Static analyzers (`CA2007`, custom Roslyn rules) help; tests with timeouts catch the rest.
- **Disposing a `SemaphoreSlim` while waiters exist** throws `ObjectDisposedException` on the awaiters, not on the disposer. Drain or cancel waiters first.
- **`WaitAsync` resumes on a thread-pool thread by default.** Code inside the critical section must not assume thread-affine context (`HttpContext.Current`, UI dispatcher state captured implicitly, etc.). Use `ConfigureAwait(false)` in libraries.
- **Locking on a `SemaphoreSlim` instance with `lock(...)`** compiles (it's an object) and is a category error — you've taken a `Monitor` on the semaphore, not acquired a permit.
- **Mixing `Wait()` and `WaitAsync()` on the same instance** is allowed but starves async waiters under heavy sync contention because sync waiters spin and grab released permits before async continuations can be scheduled.
- **`SemaphoreSlim.WaitAsync` allocates a `Task<bool>`** for each contended call. For very hot paths, batch work or use a `Channel<T>` with bounded capacity instead.
- **The `Lock` type's compiler-recognized lowering** only kicks in when the field is statically typed as `Lock`. If you store it in an `object` field or pass it around as `object`, you get the slow `Monitor` path with no warning beyond `CS9216` on conversion.
- **`Lock.EnterScope()` returns a ref struct.** Don't try to capture it in an async method, lambda, or field — the compiler will block you, but accidentally widening to `object` defeats the type.
- **Recursive `SemaphoreSlim` deadlocks are silent.** They don't throw; the task simply never completes. In tests, set a timeout on `WaitAsync` to surface them.
- **`SemaphoreSlim.Release(n)` past `maxCount`** throws `SemaphoreFullException`. Constructing with `new SemaphoreSlim(0)` (no max) lets you over-release silently — pass an explicit max in production code.
