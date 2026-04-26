# Synchronization bottlenecks

_Targets .NET 10 / C# 14. See also: [Thread contention](./thread-contention.md), [async/await performance](./async-await-performance.md), [Channel&lt;T&gt;](./channel-of-t.md), [Concurrent collections](../COLLECTIONS/concurrent-collections.md), [Lock statement and SemaphoreSlim](../THREAD_SYNCHRONIZATION/lock-statement-and-semaphoreslim.md), [volatile and Interlocked](../THREAD_SYNCHRONIZATION/volatile-and-interlocked.md), [dotnet-counters](./dotnet-counters.md), [PerfView](./profiling-perfview.md)._

A **synchronization bottleneck** is the subset of [thread contention](./thread-contention.md) caused by the **lock primitive itself** — a single mutex / semaphore / `Monitor` is serializing threads that could otherwise progress in parallel.

> The critical section is correct, but it has become *the* limiting resource: throughput plateaus at the rate one CPU can drain the queue of waiters, regardless of how many cores you add.

The quickest mental model: if you remove the lock, can the work run in parallel? If yes, the lock is a bottleneck — the question is how to redesign so the lock becomes unnecessary, not how to make the lock "faster".

## Pick the right primitive

| Primitive | Cost | Cross-`await`? | When |
|---|---|---|---|
| `lock` (object-based `Monitor`) | ~25 ns uncontended | No (silent footgun) | Legacy synchronization on a non-`Lock` object |
| **`System.Threading.Lock`** (.NET 9, C# 13+) | ~20 ns uncontended; cheaper than `Monitor` | **No (compile-time error)** | Default for new sync code |
| `SemaphoreSlim` (count = 1) | ~80 ns uncontended; allocation per `WaitAsync` | **Yes** (`WaitAsync`) | The only correct lock if you must `await` inside |
| `ReaderWriterLockSlim` | High overhead; only wins with very read-heavy workload | No | Read:write ratio > ~10:1, holds spanning microseconds |
| `Interlocked.*` | A single CPU instruction | N/A | Single-word counters, flags, CAS state machines |
| `Volatile.Read/Write` | Compiler/CPU memory barrier | N/A | Pair with `Interlocked` for non-locking publication |
| `ConcurrentDictionary<,>` | Lock-striped | N/A | Read/write maps with high concurrency |
| `FrozenDictionary<,>` | None (read-only) | N/A | Lookup tables built once, read forever |
| `ImmutableList/Dictionary` | Whole-snapshot allocation on write | N/A | Read-mostly with infrequent atomic swaps |
| `Channel<T>` | Bounded, lock-free fast paths | Yes (`WriteAsync`/`ReadAsync`) | Producer/consumer — usually replaces a `lock(queue)` |

**Rule**: every `lock` in a hot path deserves the question *"could this be `Interlocked` or a concurrent collection instead?"* If the answer is yes, the lock-free option always wins under contention.

## Quantitative signals

You're hitting a synchronization bottleneck when, sustained:

- **`monitor-lock-contention-count`** is rising at hundreds–thousands per second.
- **PerfView's Lock-Cause Stacks** attribute > 10% of wall time to one lock site.
- **Throughput plateaus** as concurrency increases — adding workers does not add work done.
- **Lock convoy** pattern: many threads parked, throughput collapses past a threshold concurrency level (each thread cycles `acquire → release → block → wake → acquire` in lockstep).

A `monitor-lock-contention-count` of single digits per second is not a bottleneck. Don't chase it.

## Measurement recipe

1. **Counters first**:
   ```bash
   dotnet-counters monitor System.Runtime --process-id <pid> --counters monitor-lock-contention-count,threadpool-queue-length
   ```
2. **Find the offending lock**:
   ```bash
   dotnet-trace collect --process-id <pid> --providers Microsoft-Windows-DotNETRuntime:0x4000:5
   ```
   Open in PerfView → *Thread Time → Lock-Cause Stacks*. The blocking stack identifies the *acquirer*; the lock holder is one or two frames up.
3. **Live blocked-thread inspection**:
   ```bash
   dotnet-dump collect --process-id <pid>
   dotnet-dump analyze <dump>
   > syncblk           # which sync blocks have waiters?
   > !mlocks           # who holds what
   > clrstack -all     # what are the blocked threads doing
   ```
4. **Microbenchmark the alternatives**:
   ```csharp
   [MemoryDiagnoser, ThreadingDiagnoser]
   public class CounterBenchmarks
   {
       private long _value;
       private readonly Lock _lock = new();

       [Benchmark(Baseline = true)]
       public long WithLock()
       {
           lock (_lock) return ++_value;
       }

       [Benchmark]
       public long WithInterlocked() => Interlocked.Increment(ref _value);
   }
   ```
   At one thread, `lock` and `Interlocked` are within nanoseconds. At sixteen, `Interlocked` is often 20×+ faster.

## The usual suspects

- **Coarse-grained `lock(this)` or `lock(typeof(X))`.** Both are public — *anything else* with a reference to the instance / type can also acquire the same lock and deadlock or starve you. Use a `private readonly Lock _gate = new();`.
- **`lock` held across `await`.** With `object`-based `Monitor`, this compiles and silently misbehaves: the continuation may resume on a thread that does not hold the lock — undefined behavior. With `System.Threading.Lock` (.NET 9+), the compiler refuses. **If you must coordinate across `await`, use `SemaphoreSlim.WaitAsync` + `try/finally Release()`.**
- **Double-checked locking with non-`volatile` field.** On weak memory models the second read can return a partially constructed object. Use `Lazy<T>(LazyThreadSafetyMode.ExecutionAndPublication)` or `Volatile.Read`.
- **`ConcurrentDictionary.GetOrAdd`** with a value factory that allocates expensively — the factory may run *more than once* under contention; only one result wins. Use the `Lazy<T>` pattern if the factory is expensive or has side effects.
- **`ReaderWriterLockSlim` on a write-heavy workload.** The slim version is still ~3× more expensive than `Monitor` per acquisition; with even 30% writes it loses to a plain `Lock`.
- **`SemaphoreSlim(1, 1)` used as a one-shot signal.** Use `TaskCompletionSource<T>` with `RunContinuationsAsynchronously` instead — a one-shot has no "release count" semantics.
- **Lock convoys from `Thread.Sleep` / blocking inside the critical section.** Holding a lock while doing I/O turns lock contention into pool starvation — see [thread contention](./thread-contention.md).
- **Forgetting `try/finally` around `Release()`.** An exception inside the critical section permanently consumes a semaphore slot — a slow leak that ends in a starvation incident weeks later.

## Reduction strategies

The order of preference is roughly **don't share** > **share immutable** > **lock-free** > **fine-grained locks**.

| Strategy | Pattern | Example |
|---|---|---|
| Don't share | Pass data by value; per-thread / per-request state | `AsyncLocal<T>`, scoped DI |
| Share immutable | Build once, swap atomically | `Volatile.Write(ref _table, newTable)` |
| Lock-free | `Interlocked.CompareExchange` | Counters, flags, append-only lists |
| Striped concurrent | `ConcurrentDictionary` (lock-striped internally) | Cache, registries |
| Fine-grained `Lock` | One lock per *bucket* of state, never one per service | Per-key locking with `LockManager<TKey>` |
| Async coordination | `SemaphoreSlim` (count > 1 = concurrency limiter) | Outbound rate limit, downstream throttle |
| Avoid sharing entirely | `Channel<T>` ownership transfer | Producer/consumer, work scheduling |

## Worked example — hot counter

```csharp
// Before: every increment serializes through the Lock; under 16 threads,
// 90% of CPU time is spent contending on the same word.
public sealed class Counter
{
    private readonly Lock _gate = new();
    private long _value;

    public long Increment()
    {
        lock (_gate) return ++_value;
    }
}

// After: lock-free, ~25× throughput at 16 threads, no allocations.
public sealed class Counter
{
    private long _value;

    public long Increment() => Interlocked.Increment(ref _value);
}
```

## Worked example — read-mostly cache

```csharp
// Before: every reader pays the lock cost; writers also pay; nothing scales.
public sealed class Catalog
{
    private readonly Lock _gate = new();
    private Dictionary<string, Product> _items = new();

    public Product? Get(string sku)
    {
        lock (_gate) return _items.GetValueOrDefault(sku);
    }

    public void Replace(IEnumerable<Product> next)
    {
        lock (_gate) _items = next.ToDictionary(p => p.Sku);
    }
}

// After: reads are lock-free; writers swap an immutable snapshot atomically.
// FrozenDictionary is faster than Dictionary for lookups and immutable.
public sealed class Catalog
{
    private FrozenDictionary<string, Product> _items =
        FrozenDictionary<string, Product>.Empty;

    public Product? Get(string sku) =>
        Volatile.Read(ref _items).GetValueOrDefault(sku);

    public void Replace(IEnumerable<Product> next) =>
        Volatile.Write(ref _items, next.ToFrozenDictionary(p => p.Sku));
}
```

The catalog now scales linearly with reader cores. The cost is one extra `Dictionary → FrozenDictionary` build on each rebuild — usually rare and off the hot path.

**Senior-level gotchas:**
- **`System.Threading.Lock` (the type) and `lock` (the keyword) interact.** With `Lock _gate = new(); lock (_gate) { ... }` the compiler emits `Lock.EnterScope()` rather than `Monitor.Enter` — a different fast path. Mixing `Monitor.Enter(_gate)` and `lock(_gate)` against a `Lock` instance is a bug; pick the keyword form.
- **`Interlocked` is not a free pass.** Under heavy contention on a single cache line, `Interlocked.Increment` ping-pongs the line between cores ("false sharing" if it shares a 64-byte line with another hot field). For per-core counters use `[FieldOffset]` padding or `LongPaddedCounter` patterns.
- **`ConcurrentDictionary` enumeration is a snapshot at an unspecified instant.** It will not throw, but it does not reflect a single point in time — don't compute totals by iterating.
- **`ConcurrentBag<T>` is for thread-local producer/consumer.** It performs poorly when one thread produces and another consumes — use `Channel<T>`.
- **`ImmutableList<T>.Add` is O(log n)`** and allocates, but a tight loop of `Add`s rebuilds the tree per call. Use `ImmutableList<T>.Builder` or `ToImmutableList()` once at the end.
- **`SpinLock` is rarely correct.** It's a tool for *very* short critical sections (< ~1 µs) on a thread that holds no other resources. In application code, the chance you've correctly identified that case is small; use `Lock`.
- **`SemaphoreSlim` does not support recursion.** A thread can `WaitAsync` itself into a deadlock if you forget. `Lock` and `Monitor` are recursive; cross-`await` synchronization is not.
- **`Lazy<T>` with the default mode is `ExecutionAndPublication`** — thread-safe and runs the factory once. The wrong mode (`PublicationOnly`) runs the factory potentially many times. Read the constructor carefully.
- **A "lock-free" algorithm with one `Interlocked.CompareExchange` and a busy retry loop can be worse than a `Lock`** under high contention, because the loops burn CPU instead of parking. Measure under realistic concurrency, not single-threaded.
- **Cross-process synchronization is a different beast.** `Mutex` (named) and `Semaphore` (named) involve kernel transitions and persist beyond your process. They are correct for IPC but a poor choice in-process — `Lock` is two orders of magnitude faster.
