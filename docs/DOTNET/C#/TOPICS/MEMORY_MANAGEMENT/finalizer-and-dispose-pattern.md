# Finalizer and Dispose pattern

_Targets .NET 10 / C# 14. See also: [How GC works](./how-gc-works.md), [GC generations](./gc-generation.md)._

The GC reclaims **managed** memory. It does not close files, release sockets, unlock database connections, or decrement COM ref counts. That's your job. The two mechanisms C# offers are:

1. **`IDisposable` / `IAsyncDisposable`** — deterministic cleanup, called by `using` or explicit `Dispose()`.
2. **Finalizers** (`~Class()`) — non-deterministic safety net, called by the GC on a dedicated thread when the object becomes unreachable.

Use both together only when you wrap **unmanaged** state (a raw handle, a pointer) and want deterministic cleanup with a last-resort fallback. For purely managed cleanup (disposing composed `IDisposable` fields), implement `IDisposable` only — no finalizer.

## The one-line rule

> **Prefer `SafeHandle` over a hand-rolled finalizer.** If your class only composes other `IDisposable`s, implement `IDisposable` with no finalizer and no virtual `Dispose(bool)`.

Most of the "classic" Dispose pattern exists because older code held raw `IntPtr`s. Modern code wraps the handle in a `SafeHandle` (or `CriticalHandle`) subclass, which has its own finalizer and thread-safe release. The containing class then implements only `IDisposable`.

## Finalizers: what they actually are

```csharp
public sealed class NativeBuffer
{
    private IntPtr _ptr;

    public NativeBuffer(int size) => _ptr = Marshal.AllocHGlobal(size);

    ~NativeBuffer()                 // the finalizer
    {
        if (_ptr != IntPtr.Zero) Marshal.FreeHGlobal(_ptr);
    }
}
```

Key CLR facts most devs don't know:

- Declaring `~Class()` compiles to `protected override void Finalize()` on `object`. There's no "destructor" — the C# spec just borrows the `~` syntax.
- An object with a finalizer is **registered on the finalization queue** at allocation. When it becomes unreachable, the GC doesn't collect it — it moves it to the **f-reachable queue**. A dedicated finalizer thread drains that queue calling `Finalize()`. Only after that does the object become truly collectable — **a full GC cycle later**.
- Net effect: a finalizable object survives **at least one extra GC generation**. A Gen0 finalizable object gets promoted to Gen1 (or Gen2) before it's freed. This is expensive.
- Finalizers run on a single dedicated thread, in **no guaranteed order** — you cannot touch another managed reference field and assume it hasn't also been finalized. This is the single most dangerous trap.
- If a finalizer throws, the process terminates (since .NET 2.0+). There is no way to observe it with `try/catch` at the call site.
- At process shutdown, the runtime gives finalizers ~2 seconds total, then gives up. Do not rely on them for cleanup that **must** happen.
- Modern .NET has no AppDomains, so the only eager-run API is `GC.WaitForPendingFinalizers()` after a `GC.Collect()`.

## The classic Dispose pattern (when you really need it)

Use this shape only when you own an unmanaged resource directly **and** the type is not sealed. For sealed types, simplify.

```csharp
public class UnmanagedResourceOwner : IDisposable
{
    private IntPtr _handle;           // unmanaged
    private FileStream? _stream;      // managed, disposable
    private bool _disposed;

    public UnmanagedResourceOwner(string path)
    {
        _stream = File.OpenRead(path);
        _handle = Native.OpenThing();
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);    // don't bother the finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Safe to touch managed fields here — we're on the caller's thread.
            _stream?.Dispose();
        }

        // Unmanaged cleanup runs in BOTH paths.
        if (_handle != IntPtr.Zero)
        {
            Native.CloseThing(_handle);
            _handle = IntPtr.Zero;
        }

        _disposed = true;
    }

    ~UnmanagedResourceOwner() => Dispose(disposing: false);
}
```

The `disposing` flag is **the** reason this pattern is shaped this way:
- `disposing: true` → called from `Dispose()` on a normal thread; safe to touch other managed objects.
- `disposing: false` → called from the finalizer; **other finalizable objects may already be finalized**. Touch only unmanaged state.

`GC.SuppressFinalize(this)` in `Dispose()` removes the object from the finalization queue, so it doesn't pay the extra-generation penalty.

## The modern, `SafeHandle`-based version

```csharp
public sealed class ThingHandle : SafeHandle
{
    public ThingHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;
    protected override bool ReleaseHandle() { Native.CloseThing(handle); return true; }
}

public sealed class UnmanagedResourceOwner : IDisposable
{
    private readonly ThingHandle _handle = Native.OpenThing();
    private readonly FileStream _stream;

    public UnmanagedResourceOwner(string path) => _stream = File.OpenRead(path);

    public void Dispose()
    {
        _handle.Dispose();
        _stream.Dispose();
    }
}
```

No finalizer. No `Dispose(bool)`. No `GC.SuppressFinalize`. `SafeHandle` wins here because:
- It's **reference-counted** (`DangerousAddRef` / `Release`), so P/Invoke callers using `[DllImport]` with a `SafeHandle` parameter can't accidentally free the handle while unmanaged code is still using it.
- Its finalizer is a **critical finalizer** (CER), guaranteed to run even on rude aborts.
- The runtime special-cases it for marshaling — the handle is pinned only for the duration of the P/Invoke call.

The CA1063 analyzer (`Implement IDisposable correctly`) is less useful once you move to `SafeHandle`. The `System.Runtime` analyzers will still nag if you finalize and don't `SuppressFinalize`.

## `IAsyncDisposable`

Introduced in C# 8 for resources whose cleanup is genuinely asynchronous — flushing a `Stream`, committing a `DbConnection`, sending a final message.

```csharp
public sealed class BatchPublisher : IAsyncDisposable, IDisposable
{
    private readonly Channel<Message> _channel = Channel.CreateBounded<Message>(1024);
    private readonly Task _pump;

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();
        await _pump.ConfigureAwait(false);   // flush drain
        GC.SuppressFinalize(this);           // only if you have a finalizer (you shouldn't)
    }

    public void Dispose()                    // synchronous fallback
    {
        // Best-effort. Do NOT do DisposeAsync().GetAwaiter().GetResult() — deadlocks.
        _channel.Writer.TryComplete();
    }
}
```

Consumption:

```csharp
await using var p = new BatchPublisher();
await p.PublishAsync(msg, ct);
// DisposeAsync called on scope exit
```

Rules:
- `ValueTask DisposeAsync()` — returns `ValueTask` (not `Task`) to avoid an allocation on the common sync-complete path.
- Do **not** block on `DisposeAsync().GetAwaiter().GetResult()` in a synchronous `Dispose()`. You'll deadlock on a synchronization context (ASP.NET Classic, WPF) and thrash the thread pool otherwise.
- Implement both `IDisposable` and `IAsyncDisposable` **only** if sync callers cannot migrate (legacy `using` blocks). New code should be `await using` throughout.
- `ConfigureAwait(false)` inside `DisposeAsync` — you don't want to re-capture the caller's context for teardown.

## `using` and `using var`

```csharp
// Classic scope — ends at }
using (var s = File.OpenRead(path)) { ... }

// C# 8 declaration form — ends at the enclosing block's }
using var s = File.OpenRead(path);
...

// Async variant (C# 8)
await using var conn = new SqlConnection(cs);
```

Compiler lowering: `using x` becomes `try { ... } finally { if (x != null) x.Dispose(); }`. The `await using` form becomes `finally { if (x != null) await x.DisposeAsync(); }`. Important consequences:
- The `finally` always runs, even on exception — that's the whole point.
- A `using` on a `null` variable is legal (no NRE).
- The disposable is **captured at the `using` point**; reassigning the variable later does not re-target the dispose.

## When to implement `IDisposable`

- You own a handle (`SafeHandle`), native buffer, or unmanaged pointer.
- You subscribe to an event on a longer-lived publisher — the event keeps your subscriber alive. Unsubscribe in `Dispose`.
- You own a disposable field (composition): `HttpClient`, `FileStream`, `Timer`, `CancellationTokenSource`.
- You hold a lock, semaphore, mutex, or enlist in a transaction.

When **not** to implement `IDisposable`:
- You only hold references to GC-managed data. The GC handles it.
- You wrap another `IDisposable` but you don't own its lifetime (classic `HttpMessageHandler` inside `HttpClient` — the outer disposes inner).
- You're tempted to "free memory sooner" — that's what the GC is for. Nulling out fields in `Dispose` almost never helps and can hurt.

## Resurrection (the weird corner)

```csharp
static Zombie? _revived;

sealed class Zombie
{
    ~Zombie() => _revived = this;   // saves itself from the GC
}
```

Assigning `this` to a live reference from inside a finalizer **resurrects** the object. It becomes reachable again and is not collected. `GC.ReRegisterForFinalize(this)` re-arms the finalizer for next time. This is a curiosity, not a pattern — mention it in interviews; never write it in production.

**Senior-level gotchas:**
- **Finalizers cost you a generation.** A finalizable object gets promoted before it's freed. On a hot path, every instance with `~Class()` is a small GC tax. That's why `SafeHandle` is the standard answer — the handle has the finalizer, not the outer object.
- A finalizer runs on the **finalizer thread**. Capturing the logger, accessing `HttpContext.Current`, touching thread-local state — all meaningless there. If you `throw` in a finalizer, the process dies (`Environment.FailFast`-style).
- **Do not assume other managed objects are still alive in your finalizer.** Field order is not finalization order. Call into unmanaged cleanup only.
- `GC.SuppressFinalize(this)` must be called in the **public** `Dispose()`, not in `Dispose(bool)`. Otherwise the finalizer runs redundantly on already-disposed objects.
- `using var x = SomeDisposable();` disposes at **end of enclosing scope**, not end of statement. Inside a loop, the object lives until after the loop — usually not what you want for per-iteration cleanup. Use the block form `using (...) { }` to scope tightly, or extract a method.
- `DisposeAsync()` on a class that doesn't genuinely need async disposal is noise. Don't add it "for future-proofing" — every caller now has to `await using`, and you've broken sync call sites for no gain.
- `CA2000` (dispose objects before losing scope) has false positives with builder patterns (`new X().Configure(...)` where `X` is disposable). Silence with `#pragma` where the lifetime genuinely transfers; don't paper over it with `using` that disposes too early.
- `Dispose()` **must be idempotent**. Users will call it twice — from `using` after a manual call, from double-dispose in a `finally`. Use a `_disposed` flag guard.
- Disposing a shared, captured `CancellationTokenSource` from inside a registered callback causes `ObjectDisposedException` on unregistration. Either don't dispose shared CTS (let them GC — they're cheap) or orchestrate disposal after all linked tokens are known dead.
- Throwing from `Dispose` / `DisposeAsync` during stack unwinding **replaces** the in-flight exception — you lose the original diagnostic. `try/finally` teardown code should swallow (and log) or be infallible.
- `HttpClient` is `IDisposable` but **the idiomatic pattern is `IHttpClientFactory` / singleton `HttpClient` for the process lifetime**. Disposing it per-request exhausts sockets (TIME_WAIT). The rule "always dispose IDisposable in a using" has a famous exception here.
