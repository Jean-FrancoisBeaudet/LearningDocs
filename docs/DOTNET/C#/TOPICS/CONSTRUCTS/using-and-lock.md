# `using` and `lock`

_Targets .NET 10 / C# 14. See also: [Finalizer & Dispose pattern](../MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md), [lock statement & SemaphoreSlim](../THREAD_SYNCHRONIZATION/lock-statement-and-semaphoreslim.md), [async/await](./async-await.md), [Semaphore, Mutex, Monitor](../THREAD_SYNCHRONIZATION/semaphore-mutex-monitor.md)._

Two small keywords that paper over two hard problems: deterministic cleanup of unmanaged/scarce resources (`using`) and mutual exclusion in a shared-memory concurrent program (`lock`). Both compile to `try`/`finally` blocks — knowing what the compiler emits is half the battle.

## `using` statement — deterministic disposal

```csharp
using (var file = File.OpenText(path))
{
    return file.ReadToEnd();
}
// compiler-emitted finally ensures file.Dispose() runs
```

Lowered to:

```csharp
var file = File.OpenText(path);
try
{
    return file.ReadToEnd();
}
finally
{
    if (file is not null) ((IDisposable)file).Dispose();
}
```

Key points:
- The declared variable is **readonly inside the block** — you can't reassign it.
- The null check is real: if an earlier constructor left things half-built, `Dispose` isn't called on a null reference.
- Exceptions from `Dispose` **overwrite** any exception escaping the body unless the body also threw (then the original exception wins only if `Dispose` throws after unwinding — standard CLR behavior).

## `using` declaration (C# 8)

Scope-to-end-of-enclosing-block, no extra nesting:

```csharp
public string Read(string path)
{
    using var file = File.OpenText(path);   // disposed at end of method
    using var gz   = new GZipStream(file.BaseStream, CompressionMode.Decompress);
    return new StreamReader(gz).ReadToEnd();
}
```

Disposal order is **reverse of declaration** — `gz.Dispose()` runs before `file.Dispose()`. That matches nested `using` statements.

Constraints on `using var`:
- Can only declare a local; not valid on fields or in a `switch` section.
- Lives until the end of the enclosing block — don't declare one at method scope if you want it disposed inside a loop.

## `IDisposable` vs `IAsyncDisposable`

```csharp
await using var conn = new SqlConnection(cs);   // await using statement-/declaration-form
await conn.OpenAsync(ct);
```

`IAsyncDisposable` added `DisposeAsync()` returning `ValueTask`. The rules:
- **Both** implemented: the compiler calls `DisposeAsync()` for `await using`, `Dispose()` for `using`. Good types implement both; `DisposeAsync()` does the real work and `Dispose()` forwards (possibly blocking — often a design smell).
- **Only `IAsyncDisposable`**: sync `using` won't compile. Must `await using`.
- **Only `IDisposable`**: `await using` still works, calling the sync `Dispose`.

Never implement `DisposeAsync()` by calling `Dispose().RunSynchronously()`-style wrappers — the point of async disposal is to avoid blocking a thread on I/O (flush-to-disk, send-closing-frame, etc.).

## Pattern-based `Dispose` (no interface needed)

For `ref struct`s pre–C# 13, the compiler looks for a public `Dispose()` method — they couldn't implement `IDisposable` because ref structs couldn't implement any interface. C# 13 lifts that restriction, but the pattern-based pathway still works.

```csharp
ref struct PooledBuffer
{
    byte[] _rented;
    public PooledBuffer(int size) => _rented = ArrayPool<byte>.Shared.Rent(size);
    public void Dispose() => ArrayPool<byte>.Shared.Return(_rented);
}

using var buf = new PooledBuffer(4096);
// scope ends → Dispose called, array returned to pool
```

## Writing `Dispose` correctly

If you hold **only managed IDisposables**: implement `IDisposable`, call `Dispose()` on each field. Don't write a finalizer.

If you hold **unmanaged resources** (raw handles, pointers, pinned memory): use the full dispose pattern + `SafeHandle` instead of a finalizer when possible. See [Finalizer & Dispose pattern](../MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md).

## `lock` — mutual exclusion

```csharp
private readonly object _gate = new();
private int _counter;

public void Increment()
{
    lock (_gate)
    {
        _counter++;
    }
}
```

Lowered to:

```csharp
bool taken = false;
var gate = _gate;
try
{
    Monitor.Enter(gate, ref taken);
    _counter++;
}
finally
{
    if (taken) Monitor.Exit(gate);
}
```

`lock` is **reentrant** — the same thread can re-acquire the monitor without deadlocking (`Monitor` tracks ownership + count).

### The classic lock-object anti-patterns

```csharp
// ❌ lock(this)    — exposes the monitor: any caller with a ref to `this`
//                    can lock on it, causing accidental deadlocks.
// ❌ lock(typeof(T))
//                    — the Type instance is process-global and shared with
//                    reflection code you don't own.
// ❌ lock("literal")
//                    — interned strings are shared across assemblies.
// ❌ lock(boxedStruct) — each boxing is a new object; different calls lock
//                    on different monitors and the lock "works" only once.
```

Rule: lock on a `private readonly object _gate = new();` dedicated to this section of state. One gate per independent invariant.

### The new `System.Threading.Lock` (.NET 9 / C# 13)

```csharp
using System.Threading;

private readonly Lock _gate = new();

public void Increment()
{
    lock (_gate)            // recognized by the compiler — uses EnterScope()
    {
        _counter++;
    }
}
```

If the field is typed as `Lock`, the compiler emits calls to `Lock.EnterScope()` (returning a disposable scope struct) rather than `Monitor.Enter/Exit`. Benefits:
- Slightly cheaper; no monitor sync-block promotion on the target object.
- Misuse-resistant — you can't accidentally monitor-wait on a `Lock`.
- Warns on `lock(myLockCastToObject)` — the compiler catches you routing around the new type.

Keep the type visible as `Lock`; if you pass it around typed as `object`, you fall back to the old monitor behavior.

### `lock` and `await` — never together

```csharp
lock (_gate)
{
    await DoSomethingAsync();   // ❌ CS1996: cannot await in lock statement body
}
```

The compiler rejects it. Even if it didn't, the monitor is **thread-affine**; resuming on a different thread would mean releasing a lock you never acquired.

For async-safe exclusion, use `SemaphoreSlim(1, 1)`:

```csharp
private readonly SemaphoreSlim _sem = new(1, 1);

public async Task DoAsync(CancellationToken ct)
{
    await _sem.WaitAsync(ct).ConfigureAwait(false);
    try
    {
        await WorkAsync(ct).ConfigureAwait(false);
    }
    finally
    {
        _sem.Release();
    }
}
```

Note `SemaphoreSlim` is **not reentrant**. If the same logical flow can re-enter, you need different shaping. See [lock-statement-and-semaphoreslim](../THREAD_SYNCHRONIZATION/lock-statement-and-semaphoreslim.md) for more.

### `Monitor.Enter` with timeout

When you need to time out acquisition (or report lock contention metrics), drop to `Monitor` explicitly:

```csharp
if (!Monitor.TryEnter(_gate, TimeSpan.FromMilliseconds(100)))
    throw new TimeoutException();
try { … } finally { Monitor.Exit(_gate); }
```

With the new `Lock` type:

```csharp
if (_gate.TryEnterScope(TimeSpan.FromMilliseconds(100), out var scope))
{
    using (scope) { … }
}
```

### Deadlock mitigation

- **Consistent lock ordering** across the codebase. If you ever acquire two locks, both code paths must acquire them in the same order.
- **Narrow critical sections.** Never do I/O, user callbacks, or long computations inside a `lock`. Copy what you need, exit the lock, then work.
- **Never call out to unknown code holding a lock.** Events, virtual methods, delegates — any of them can re-enter the same lock from elsewhere.
- **Prefer immutable or `Interlocked`-updated state** for simple counters/flags; locks should be the last resort, not the first tool.

## Mini comparison

| Use case | Tool |
|---|---|
| Release a scarce sync resource | `using` / `using var` |
| Release a scarce async resource (network stream, DB conn) | `await using` |
| Mutual exclusion in sync code | `lock` (prefer new `Lock` type on .NET 9+) |
| Mutual exclusion across `await` | `SemaphoreSlim(1,1)` |
| Named cross-process exclusion | `Mutex` (named) |
| Many readers / few writers | `ReaderWriterLockSlim` |
| Atomic counter | `Interlocked.Increment` |
| Wait for N participants | `SemaphoreSlim(0, N)` / `CountdownEvent` |

## Senior-level gotchas

- **`using var`'s scope is the enclosing block**, not the enclosing statement. Inside a `foreach`, declare `using var` **inside** the body if you need per-iteration disposal.
- **Disposal order is reverse of declaration.** That's important when the inner resource depends on the outer (stream + reader — reader must close first).
- **Throwing from `Dispose` is awful**: if the body also threw, the runtime typically drops the original. Make `Dispose` quiet and idempotent.
- **`DisposeAsync()` returns `ValueTask`, not `Task`.** Don't accidentally `.Result` it; await it or fire-forget via `AsTask()`.
- **`lock` requires a reference-type target.** Locking on a value type doesn't compile; locking on `this`/`typeof(T)`/strings compiles but is wrong.
- **On .NET 9+, type your lock field as `System.Threading.Lock`.** If `lock (field)` still emits Monitor calls, your field type isn't `Lock`.
- **`lock` is reentrant; `SemaphoreSlim` is not.** Code that naturally recurses into its own guarded method will deadlock under `SemaphoreSlim`.
- **You cannot `await` inside `lock`**, and the compiler enforces it. Anyone writing `SemaphoreSlim` + `WaitAsync` + `await` must remember to `Release()` in `finally` — unlike `lock`, there's no compiler-generated unwinder.
- **Exceptions escaping a `lock` body still release the monitor** — the compiler-emitted `finally` handles it. Same for `using`. Same for `await using`.
- **Don't `await` a `Task` inside a section you've guarded with `Monitor.Enter` manually.** `Monitor` ownership is thread-bound; a resumed continuation on a thread-pool thread can't call `Monitor.Exit` on the original thread's monitor.
