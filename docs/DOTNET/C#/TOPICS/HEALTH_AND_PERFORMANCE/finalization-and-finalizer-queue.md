# Finalization and the finalizer queue

_Targets .NET 10 / C# 14. See also: [Finalizer and Dispose pattern](../MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md), [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md), [GC generations](../MEMORY_MANAGEMENT/gc-generation.md), [Memory leaks in .NET](./memory-leaks-in-dotnet.md)._

`MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md` covers **how to write** finalizers and `IDisposable`. This note covers **what the runtime does at GC time** — the queues, the dedicated thread, the cost model, and the diagnostics. If you understand the data structures, the rules ("don't touch other managed references in your finalizer", "always `SuppressFinalize`") stop looking arbitrary and start looking like the only option.

## Two queues, not one

The runtime tracks finalization with **two** linked queues:

| Queue | Purpose | Acts as a GC root? |
|---|---|---|
| **Finalization queue** | Every live object whose type has a finalizer is registered here at allocation time. Removed by `GC.SuppressFinalize`. | No |
| **f-reachable queue** | Objects that have *become* unreachable but still need their finalizer to run. | **Yes** — entries on this queue keep their objects (and everything those objects reference) alive. |

The names "finalization queue" and "f-reachable queue" are CLR-internal terms; you'll see them in PerfView events, debugger commands, and CLR source. The shorter way to remember it: the first list is "things that *could* need finalization"; the second is "things that *do* need finalization right now."

## Lifecycle of a finalizable object

Take a class with a finalizer:

```csharp
public sealed class NativeBuffer
{
    private IntPtr _ptr;
    public NativeBuffer(int size) => _ptr = Marshal.AllocHGlobal(size);
    ~NativeBuffer() { if (_ptr != IntPtr.Zero) Marshal.FreeHGlobal(_ptr); }
}
```

Walk through what happens:

1. **Allocation.** `new NativeBuffer(...)` lands on Gen0. Because the type has a `Finalize` override (the C# `~ClassName()` syntax compiles to `protected override void Finalize()`), the runtime **adds the new object's address to the finalization queue**.

2. **Becomes unreachable.** The next GC walks roots, finds the object is no longer reachable from the root set, but **also** finds an entry on the finalization queue pointing at it. The GC moves that entry from the finalization queue to the **f-reachable queue**. The f-reachable queue is itself a GC root, so the object survives this collection. It also gets **promoted** to the next generation (Gen0 → Gen1, etc.).

3. **Finalizer thread runs.** A single dedicated thread drains the f-reachable queue, calling `Finalize()` on each object in turn. When `Finalize()` returns, the entry is removed; the object is now a "regular" unreachable object.

4. **Reclaimed.** On the next GC of the appropriate generation, the object's memory is freed.

The net effect: **a finalizable object survives at least one extra collection, and almost certainly gets promoted to a higher generation than its non-finalizable peers.** That's a real cost, paid per allocation, multiplied by your alloc rate.

## The finalizer thread

- Single thread, dedicated, named `Finalizer` in debuggers (visible as `.NET Finalizer` in dumps).
- Normal priority. Runs in parallel with user code.
- Has **no synchronization context**. `await` will not return to it; thread-local state is empty.
- Drains the f-reachable queue in **arrival order**, but inside a "wave" of finalizable objects discovered by one GC, **the order between siblings is undefined**.
- If a finalizer **throws**, the runtime calls `Environment.FailFast` — process termination. No `try/catch` outside the finalizer can save you.
- A finalizer that blocks (deadlock, synchronous I/O) stalls the entire queue. Every other finalizable object backs up. This is one of the most common production hangs around finalization.

```csharp
~Bad()
{
    Console.WriteLine("disposing");      // surprisingly OK
    _logger.LogError("leaked");          // logger might be finalized — UB
    _semaphore.Wait();                   // can deadlock the finalizer thread
    throw new Exception("oops");         // process dies
}
```

## Cost model

Quick mental formula: a finalizable Gen0 object costs you, on top of a normal Gen0 alloc:

- One survival across the GC that discovered it unreachable (extra mark + promote).
- One promotion into Gen1 (and possibly Gen2 if it survived to Gen1 first).
- One finalizer-thread dispatch (the cost of `Finalize()` itself).
- One more collection cycle to actually reclaim the memory.

A non-finalizable object, by contrast, dies in Gen0 with zero of those overheads.

In a hot service path, this is significant. 100 finalizable allocations per second is fine. 100,000 per second is a measurable Gen1/Gen2 promotion storm — visible as "rising Gen2 size with stable live set" in counters.

## GC.SuppressFinalize: what it does

`GC.SuppressFinalize(this)` removes the object from the **finalization queue** at runtime. After that call, the GC will not put it on the f-reachable queue when it becomes unreachable, and the finalizer thread never sees it.

```csharp
public void Dispose()
{
    if (_disposed) return;
    Native.Close(_handle);
    _disposed = true;
    GC.SuppressFinalize(this);   // we already cleaned up; don't bother the finalizer
}
```

If you implement `IDisposable` on a type that has a finalizer and forget `SuppressFinalize` in `Dispose`, you eat the entire finalization tax even though you cleaned up deterministically. Don't.

The mirror operation, `GC.ReRegisterForFinalize(this)`, puts an object back on the finalization queue. The use case is the same as resurrection (see below) — exotic and rarely correct.

## Critical finalizers and SafeHandle

Regular finalizers are best-effort: at process shutdown, the runtime gives the finalizer thread ~2 seconds to drain, then exits regardless. **Critical finalizers** — types deriving from `CriticalFinalizerObject` — are guaranteed to run, even on rude aborts (`Environment.FailFast`, AppDomain unload in the legacy framework). The two BCL types you'll meet:

- `SafeHandle` (and `SafeHandleZeroOrMinusOneIsInvalid`, `SafeHandleMinusOneIsInvalid`) — wraps an OS handle; reference-counted; finalizer guaranteed to run; native release happens via `ReleaseHandle()`.
- `CriticalFinalizerObject` — base class for any type that absolutely must release something at process death.

A `SafeHandle` subclass is the **right** way to own an OS resource:

```csharp
public sealed class FileHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public FileHandle() : base(ownsHandle: true) { }
    protected override bool ReleaseHandle() => Native.CloseFile(handle);
}
```

The wrapper has its own critical finalizer; **your enclosing class then doesn't need a finalizer at all**. It just disposes the `SafeHandle`. This is the modern shape of the Dispose pattern.

## Process shutdown

At process exit:

- The runtime stops accepting new finalizer registrations.
- It calls `Finalize()` on every object remaining on the finalization queue (modulo `GC.SuppressFinalize`).
- It enforces a soft deadline (~2s by default for managed-host shutdowns; controlled by host policy).
- Critical finalizers (CER-rooted, including `SafeHandle`) are guaranteed to run.

If you have logic that **must** run for correctness — flushing a queue, committing a transaction — do it via `await using` / explicit shutdown hooks (`IHostedService.StopAsync`, `IHostApplicationLifetime`). Finalizers are the wrong place. Treat them as a safety net for unmanaged-handle leaks, nothing more.

## Observing the finalizer queue

| Tool | What it shows |
|---|---|
| `dotnet-counters monitor System.Runtime` | `gc-finalization-queue-length` (pending finalizable objects waiting on the f-reachable queue) |
| PerfView GC events | `FinalizeObject`, `GCFinalizersBegin/End` (per-cycle finalization wall-time) |
| `dotnet-dump analyze` + `!finalizequeue` (SOS) | Both queues, by generation, with type names and instance counts |
| `GC.GetGCMemoryInfo()` | Aggregate memory stats; the GC info struct exposes the finalizer-queue indirectly via promotion counts |
| dotMemory / dotTrace | Visual finalization queue inspection, retention through f-reachable |

A long, growing finalization queue is one of two things:

1. The finalizer thread is **stuck** (blocked on a lock, an exception, an external call). One stuck finalizer halts the queue.
2. Your **allocation rate of finalizable types** outpaces the finalizer thread's ability to drain. Real fix: implement `IDisposable` and call `Dispose` deterministically, or remove the finalizer entirely if the resource is purely managed.

## Anti-patterns

- Adding `~ClassName()` "just in case" to a class that holds only managed `IDisposable`s. Net effect: you pay finalization tax on every instance, and the finalizer can't safely touch those references anyway because field order is not finalization order.
- Calling `GC.WaitForPendingFinalizers()` in tests to "force cleanup". It serializes against the finalizer thread, masks lifetime bugs, and slows the test suite. If you need deterministic cleanup, call `Dispose`.
- Logging or sending telemetry from a finalizer. The logger sink, the HttpClient, the trace listener — any of them may have already been finalized, or the process may be in shutdown.
- Holding non-handle managed references in a finalizable type. They're **rooted** until the finalizer runs, dragging an entire object graph into Gen2.
- Calling `GC.Collect()` followed by `GC.WaitForPendingFinalizers()` "to make sure stuff is cleaned up." This is only valid in test fixtures and benchmark harnesses; in production it's a Gen2 storm.

## Resurrection

```csharp
static List<Zombie> _saved = new();
sealed class Zombie
{
    ~Zombie() => _saved.Add(this);    // become reachable again from inside the finalizer
}
```

Assigning `this` to a live reference inside a finalizer **resurrects** the object — the GC discovers, on the *next* cycle, that it's reachable again, and stops trying to collect it. The finalizer will not run a second time unless you `GC.ReRegisterForFinalize(this)`. Mention it in interviews; never write it in production code. It almost always indicates the wrong design.

**Senior-level gotchas:**

- Finalizable objects keep their **entire reachable graph** rooted until finalized. One finalizable cache entry can pin a megabyte of objects in Gen2 for an extra cycle. The cost isn't just the finalizable object — it's everything it transitively references.
- Field order is **not** finalization order. Inside `Finalize()`, any other managed reference field may already have been finalized. Touch only unmanaged state. (`SafeHandle` subclasses are an exception — they're guaranteed to be alive in their own `ReleaseHandle`.)
- `SafeHandle` is the standard answer for OS handles. Hand-rolled finalizers exist for legacy reasons; new code should always derive from `SafeHandle` (or an existing one like `Microsoft.Win32.SafeHandles.*`).
- Steady-state `gc-finalization-queue-length` above a few hundred items in a service is a smell — either a finalizer is slow/stuck, or a hot type has a finalizer it doesn't need. Investigate immediately; a stuck finalizer thread will eventually exhaust memory.
- A finalizer that throws **terminates the process**. `Environment.FailFast`-style. There is no global handler that can intercept it.
- `GC.SuppressFinalize(this)` belongs in the public `Dispose()`, after the cleanup, not in `Dispose(bool)`. Putting it elsewhere either runs it unnecessarily or omits it on the disposal path.
- Resurrection from a finalizer + `GC.ReRegisterForFinalize` is the only time you'd ever want a finalizer to run twice. If you're considering this, the design is wrong — replace with a proper pool or `IDisposable` lifecycle.
- `GC.WaitForPendingFinalizers()` blocks the calling thread until the finalizer thread drains. Useful in benchmark teardown; in production it serializes against everything finalizable in the process.
- Finalizers run on a thread with **no `AsyncLocal<T>` propagation**, no logging scope, no activity context. Anything that depends on ambient state silently sees defaults.
- Modern .NET has no AppDomains. Old advice that referenced `AppDomain.DomainUnload` for "guaranteed cleanup" is obsolete — use `IHostApplicationLifetime`, `IHostedService.StopAsync`, or `AppDomain.ProcessExit` (the only one that survived).
