# `volatile` and `Interlocked`

_Targets .NET 10 / C# 14. See also: [lock & SemaphoreSlim](./lock-statement-and-semaphoreslim.md), [Semaphore, Mutex, Monitor](./semaphore-mutex-monitor.md), [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md)._

The lowest-cost synchronization tools in .NET. Both work *without* taking a lock — `volatile` constrains how the compiler/JIT/CPU may reorder reads and writes; `Interlocked` exposes hardware atomic primitives. Use them when a `lock` would be overkill (single shared word) or impossible (lock-free data structures). Get them wrong and you ship a heisenbug that only repros on ARM64 under load.

## The .NET memory model in one paragraph

A read or write of a *word-sized* aligned value (`int`, `IntPtr`, `object` reference) is atomic — you will never read a torn half-written value. A read or write of `long`/`double` on 32-bit platforms, or any non-aligned access, is **not** atomic. Beyond atomicity, the compiler, JIT, and CPU are all free to reorder loads and stores as long as the *single-threaded* observable behavior is preserved. The .NET memory model is stronger than ECMA-335 (it forbids store-store reordering, matching x86) but weaker than sequential consistency. ARM64, which the runtime now targets first-class, is much more aggressive: a publish-then-flag pattern that "always works" on x86 can absolutely tear on ARM64.

## `volatile` keyword

```csharp
public class Worker
{
    private volatile bool _stop;

    public void Stop() => _stop = true;

    public void Run()
    {
        while (!_stop) DoUnitOfWork();
    }
}
```

What `volatile` guarantees:
- A read has **acquire** semantics — no later memory access in this thread is reordered before it.
- A write has **release** semantics — no earlier memory access in this thread is reordered after it.
- The JIT will **not** hoist a volatile read out of a loop. Without `volatile`, the JIT is allowed to cache `_stop` in a register and the worker spins forever.

What `volatile` does **not** give you:
- **No atomicity for compound ops.** `_counter++` is `read; +1; write`. Three operations. `volatile int _counter; _counter++;` is still a race.
- **No atomicity for `long`/`double`.** The compiler rejects `volatile long`. Use `Interlocked.Read` or `Volatile.Read(ref _x)`.
- **No happens-before relationship across unrelated fields**, except through the acquire/release fence on the volatile field itself. If you publish a fully-built object via a volatile reference and the reader takes a `Volatile.Read` of that reference, the reader sees a coherent object — that's the only way to get publish-safety.

Use `volatile`:
- Single-writer, multi-reader flag (cancel/stop).
- Publishing an immutable object via reference (double-checked locking, lazy init).

Don't use `volatile`:
- Anything compound (counters, accumulators).
- Anything 64-bit on 32-bit platforms.
- "Just to be safe" — it has a real cost on ARM/ARM64 and obscures intent.

## `Volatile.Read` / `Volatile.Write`

The same fences as `volatile`, applied at the call site instead of the declaration. Useful when:
- The field is `long`/`double` (which can't be marked `volatile`).
- Most accesses don't need fences and you only want the cost on the few that do.
- You're writing a generic helper (`volatile` is not allowed on type parameters).

```csharp
private long _lastSeenTicks;

public long LastSeen => Volatile.Read(ref _lastSeenTicks);

public void Update(long ticks) => Volatile.Write(ref _lastSeenTicks, ticks);
```

## `Interlocked` — atomic compound ops

`Interlocked` operations are read-modify-write executed as a single uninterruptible CPU instruction (`lock cmpxchg` on x86, LL/SC sequences on ARM). They include a full memory fence.

Counter / accumulator:

```csharp
private long _processed;

public void OnItem() => Interlocked.Increment(ref _processed);

public long Total() => Interlocked.Read(ref _processed); // atomic 64-bit read
```

Atomic swap:

```csharp
private string? _current;

public string? Replace(string newValue) =>
    Interlocked.Exchange(ref _current, newValue);   // returns previous value
```

Compare-and-swap (CAS) — the foundation of every lock-free data structure:

```csharp
private Node? _head;

public void Push(Node node)
{
    Node? observed;
    do
    {
        observed = _head;
        node.Next = observed;
    }
    while (Interlocked.CompareExchange(ref _head, node, observed) != observed);
}
```

Lock-free lazy init:

```csharp
private Service? _service;

public Service GetService()
{
    if (_service is { } existing) return existing;

    var created = new Service();
    return Interlocked.CompareExchange(ref _service, created, null) ?? created;
}
```

If two threads race, the loser's `created` instance is discarded — cheap if construction is cheap, wasteful if not. Use `LazyInitializer.EnsureInitialized` or `Lazy<T>` for expensive construction.

Other useful members:
- `Add(ref int, int)` / `Add(ref long, long)` — atomic addition; returns the new value.
- `MemoryBarrier()` — full fence with no associated read/write.
- `MemoryBarrierProcessWide()` (.NET 6+) — issues a process-wide barrier via OS-provided machinery (Windows `FlushProcessWriteBuffers`). Niche; used by GC-like or RCU-style code.
- Generic `Exchange<T>` / `CompareExchange<T>` for ref types (.NET 9+) — eliminates the `(object?)` cast dance the older overloads required.

## Comparison

| Tool | Atomicity | Fence | Cost | Use for |
|---|---|---|---|---|
| `volatile` field | Word-size only | acquire/release per access | ~free on x86; small cost on ARM | Single-writer flags, published refs |
| `Volatile.Read`/`Write` | Word-size only | acquire/release per call | same as `volatile` | Site-specific fences; `long`/`double`; generics |
| `Interlocked.*` | Read-modify-write | full fence | ~10× a non-atomic op | Counters, exchanges, CAS-based lock-free |
| `lock` (`Monitor`) | Arbitrary critical section | full fence on enter/exit | ~25 ns uncontended; > μs contended | Multi-statement invariants, compound state |

Cost figures are rough orders of magnitude — benchmark with BenchmarkDotNet for your hardware.

## Senior-level gotchas

- **`volatile long` does not compile.** The CLR cannot guarantee atomic 64-bit access on 32-bit platforms; use `Interlocked.Read` or `Volatile.Read(ref _long)`.
- **`volatile` only fences accesses to that field.** A volatile write to `_flag` does not order *prior* writes to unrelated fields against *later* reads of those fields by another thread. The two threads must both touch `_flag` for the publish-via-flag idiom to be safe.
- **Double-checked locking with a non-volatile field is broken** on weak memory models. The reader can see the reference assigned before the constructor's writes are visible. Mark the field `volatile`, or use `Lazy<T>`/`LazyInitializer`.
- **ARM64 reordering is real.** Code that "passed all tests for years" on x86 can produce torn observable state on Apple Silicon and Graviton. When in doubt, use `Volatile.Read`/`Write` or `Interlocked`.
- **CAS retry loops can livelock under contention.** If many threads hammer the same CAS site, throughput collapses. Add a backoff (`Thread.SpinWait`, then `Thread.Yield`) or fall back to a real lock.
- **`Interlocked.CompareExchange<T>` for ref types is .NET 9+.** Older runtimes only have the `int`/`long`/`object` overloads — generic algorithms had to launder through `object`.
- **`Interlocked` operations are full fences.** They are not free — a tight loop of `Interlocked.Increment` is ~10× slower than a plain increment. For per-thread counters, accumulate locally and publish in batches.
- **Reading the value `Interlocked.Increment` *returns* is the only race-free way to get "the value after my increment."** A subsequent plain read can see a higher value because another thread incremented in between.
- **`volatile` has no effect on JIT-generated code that reads through `ref`.** Pass the field by `ref` to a helper and you've lost the volatile semantics inside that helper. Use `Volatile.Read(ref x)` at the access site instead.
