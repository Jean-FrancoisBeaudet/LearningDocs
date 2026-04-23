# Concurrent collections

_Targets .NET 10 / C# 14. See also: [Channels](../ASYNC_PROGRAMMING/channels.md), [volatile and Interlocked](../THREAD_SYNCHRONIZATION/volatile-and-interlocked.md), [lock and SemaphoreSlim](../THREAD_SYNCHRONIZATION/lock-statement-and-semaphoreslim.md), [Immutable collections](./immutable-collections.md), [Frozen collections](./frozen-collections.md)._

`System.Collections.Concurrent` exists because the regular collections are **not** safe for any concurrent access — even one writer with multiple readers can corrupt the structure. The concurrent variants give you fine-grained-locked or lock-free alternatives, but each has a usage profile narrower than the API surface suggests.

## The lineup

| Type | Internals | Best when |
|---|---|---|
| `ConcurrentDictionary<K,V>` | Striped locks (`4 × proc count` by default) over bucket subsets | The general-purpose concurrent map |
| `ConcurrentQueue<T>` | Lock-free, segment-based (32-elem segments) | Multi-producer / multi-consumer FIFO without back-pressure |
| `ConcurrentStack<T>` | Lock-free linked list with CAS | Multi-producer LIFO; rarely the right answer |
| `ConcurrentBag<T>` | Per-thread stacks with steal | Producer == consumer (object pools) |
| `BlockingCollection<T>` | Wraps `IProducerConsumerCollection<T>`; threads **block** on `Take` | Legacy. Use [`Channel<T>`](../ASYNC_PROGRAMMING/channels.md) for new code |
| `ImmutableInterlocked` | Static helpers for CAS over an immutable field | Lock-free updates of small immutable state |

## `ConcurrentDictionary<K,V>`

The workhorse. Most ops are lock-free reads (`TryGetValue`, indexer-get) backed by volatile reads of the bucket head. Writes acquire one of N stripe locks based on `key.GetHashCode() % N`.

```csharp
var cache = new ConcurrentDictionary<string, byte[]>(
    concurrencyLevel: Environment.ProcessorCount * 4,
    capacity: 4096,
    StringComparer.Ordinal);
```

### `GetOrAdd` — the factory race

```csharp
// Factory may run multiple times under contention. The CAS winner's value is stored;
// losing values are silently dropped.
var bytes = cache.GetOrAdd(key, k => ExpensiveLoad(k));
```

If the factory is **expensive or has side effects**, this is wrong. Two threads racing on the same missing key both call `ExpensiveLoad`. The fix:

```csharp
var lazyCache = new ConcurrentDictionary<string, Lazy<byte[]>>();

byte[] Get(string key) =>
    lazyCache.GetOrAdd(key, k => new Lazy<byte[]>(() => ExpensiveLoad(k),
                                                  LazyThreadSafetyMode.ExecutionAndPublication))
             .Value;
```

Now multiple threads can race to insert a `Lazy<>`, but `Lazy<>.Value` ensures the *factory* runs exactly once. Same shape works for `AsyncLazy<T>` for async loaders.

`AddOrUpdate` has the same warning for the update factory — it can be invoked multiple times if the CAS loses; the **last successful** value wins. Make the update function pure.

### `TryGetValue` and the indexer

`TryGetValue` is lock-free. So is `dict[key]` (throws on miss). Reads scale linearly with cores. Writes scale up to `concurrencyLevel`, then plateau.

### `Count`, `Keys`, `Values`, `ToArray`

These take **all** stripe locks. Calling `dict.Count` in a hot loop serializes every writer in the system. If you need an approximate count, maintain a separate `Interlocked`-incremented counter. `Keys` / `Values` snapshot — heavyweight; avoid.

### `AlternateLookup<TAlternate>` (.NET 9)

Same trick as `Dictionary<K,V>`: look up a `string`-keyed dictionary by `ReadOnlySpan<char>` with no string allocation.

```csharp
var alt = cache.GetAlternateLookup<ReadOnlySpan<char>>();
if (alt.TryGetValue(spanFromBuffer, out var v)) { ... }
```

## `ConcurrentQueue<T>` / `ConcurrentStack<T>`

Lock-free; both expose `TryDequeue` / `TryPop`, never `.Take()` (no blocking). `ConcurrentQueue<T>` has a `Count` that walks segments — usable but not free. Neither has back-pressure: an unbounded producer will OOM the process. For any real producer/consumer pipeline reach for [`Channel<T>`](../ASYNC_PROGRAMMING/channels.md):

```csharp
// Concurrent queue + manual coordination — verbose, no back-pressure.
var q = new ConcurrentQueue<WorkItem>();
var sema = new SemaphoreSlim(0);

void Produce(WorkItem w) { q.Enqueue(w); sema.Release(); }

async Task Consume(CancellationToken ct)
{
    while (true)
    {
        await sema.WaitAsync(ct);
        if (q.TryDequeue(out var w)) await Handle(w, ct);
    }
}

// Channel — async-aware, optionally bounded, much simpler.
var ch = Channel.CreateBounded<WorkItem>(1024);
await foreach (var w in ch.Reader.ReadAllAsync(ct))
    await Handle(w, ct);
```

`ConcurrentQueue<T>` is still the right pick for **synchronous** throughput-critical paths (background loggers, telemetry buffers) where you don't want async overhead.

## `ConcurrentBag<T>`

Each thread gets its own local stack; `Add` pushes to the local stack, `TryTake` pops local first. If the local is empty, the bag **steals** from another thread's stack — slow path. Net: only good for **work-stealing object pools** where the producing thread is also the consumer.

```csharp
// Reasonable use: thread-local pooling.
private readonly ConcurrentBag<StringBuilder> _pool = new();

StringBuilder Rent() => _pool.TryTake(out var sb) ? sb.Clear() : new StringBuilder();
void Return(StringBuilder sb) => _pool.Add(sb);
```

For real pooling, prefer `ObjectPool<T>` from `Microsoft.Extensions.ObjectPool` — it has lifetime/policy hooks and is what ASP.NET Core uses internally.

If producers and consumers are different threads, use `ConcurrentQueue<T>` instead — `ConcurrentBag<T>`'s steal path will dominate.

## `BlockingCollection<T>` — legacy

Wraps any `IProducerConsumerCollection<T>` (queue/stack) and adds `Take` / `Add` that **block** the calling thread on full/empty. Pre-async pattern. Two problems:

1. `Take` parks a real thread — bad for thread-pool starvation under load.
2. The async story (`GetConsumingEnumerable`) is sync-over-everything.

```csharp
// Don't write new code like this — use Channel<T>.
var bc = new BlockingCollection<int>(boundedCapacity: 100);
Task.Run(() => { foreach (var x in bc.GetConsumingEnumerable()) Process(x); });
bc.Add(1);
bc.CompleteAdding();
```

Replace with [`Channel<T>`](../ASYNC_PROGRAMMING/channels.md) in new code; keep on legacy paths where the thread cost is acceptable.

## `ImmutableInterlocked` — CAS over immutable state

When the shared state is small and read-mostly, store it as a single immutable reference and update via CAS. No locks, no contention on reads.

```csharp
private ImmutableDictionary<string, int> _state = ImmutableDictionary<string, int>.Empty;

void Increment(string key) =>
    ImmutableInterlocked.AddOrUpdate(
        ref _state,
        key,
        addValueFactory:    _      => 1,
        updateValueFactory: (_, v) => v + 1);
```

Internally: spin a `Interlocked.CompareExchange` until the swap from old immutable → new immutable wins. The factories may run multiple times, just like `ConcurrentDictionary.GetOrAdd`. Cost scales with the size of the immutable structure (the "next version" allocation per attempt). Excellent for low-write, high-read configuration / routing tables.

## Decision matrix

| Need | Pick |
|---|---|
| Concurrent map, frequent writes | `ConcurrentDictionary<K,V>` |
| Concurrent map, write-once at startup, read-only after | `FrozenDictionary<K,V>` |
| Concurrent map, infrequent updates, lock-free reads | `ImmutableDictionary` + `ImmutableInterlocked` |
| Producer/consumer pipeline (async) | `Channel<T>` |
| Sync hot-path queue (logger, telemetry) | `ConcurrentQueue<T>` |
| Object pool with lifecycle policies | `ObjectPool<T>` (M.E.ObjectPool) |
| Quick LIFO across threads | `ConcurrentStack<T>` (rarely needed) |
| Coordinating start/stop via thread-blocking | `BlockingCollection<T>` (legacy only) |

## When `lock` + `Dictionary<K,V>` beats concurrent

Counter-intuitive but real: if writes vastly outnumber reads, your access pattern is short, and you don't need the lock-free read scaling, a regular `Dictionary` under a single `lock` can outperform `ConcurrentDictionary` — fewer indirections, no per-op CAS bookkeeping. Profile before deciding. The breakeven is heavily dependent on key type (`int` vs `string`), value size, and contention.

**Senior-level gotchas:**
- **`GetOrAdd` factory is not atomic with insertion** — the most common ConcurrentDictionary bug. If your factory side-effects (allocates a `FileStream`, registers a hook), wrap in `Lazy<T>`.
- `ConcurrentDictionary.Count` is O(n_stripes) **and** locks every stripe. Calling it in a `LogTrace` inside a hot loop is a documented production problem.
- `ConcurrentBag<T>` is **not** "the concurrent List<T>." It's a thread-local stack with stealing. Iteration order is meaningless. If you find yourself enumerating a bag, you've picked the wrong collection.
- `ConcurrentQueue<T>` segments are 32 elements and never shrink — a queue that briefly held 1M items keeps that memory until the GC reclaims drained segments. For long-lived queues, this is fine; for queues that experience one big burst then quiesce, the high-water mark sticks around.
- Calling `ToArray()` / `ToList()` / `Keys` / `Values` on any concurrent collection **is a snapshot at a point in time** — by the time it returns, the live collection has moved on. Don't use these for "current state" decisions; use them for diagnostics.
- `ConcurrentDictionary` does **not** preserve insertion order — and in fact has no defined enumeration order. Don't depend on it.
- The `concurrencyLevel` constructor parameter is the lock-stripe count — once set, can only grow on resize. Setting it absurdly high (1000+) wastes memory and gains nothing past `~4 × proc count`.
- `BlockingCollection<T>.GetConsumingEnumerable()` swallows the `OperationCanceledException` from its `CancellationToken` overload **only** when cancellation happens between items — mid-`Take` it propagates. Don't rely on either; migrate to `Channel<T>` for clean cancellation.
- `ConcurrentDictionary` keys with mutable hash codes corrupt the table just like `Dictionary` does — but the symptom is non-deterministic across cores. Use immutable keys (records, strings, structs).
- For very hot read-mostly state, `ImmutableDictionary` + `ImmutableInterlocked` can outperform `ConcurrentDictionary` because reads are pure pointer dereferences with no volatile barriers per lookup. Benchmark with `BenchmarkDotNet`'s `[ThreadingDiagnoser]`.
