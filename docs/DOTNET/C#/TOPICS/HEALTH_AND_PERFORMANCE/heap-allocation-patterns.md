# Heap allocation patterns

_Targets .NET 10 / C# 14. See also: [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md), [GC generations](../MEMORY_MANAGEMENT/gc-generation.md), [Large Object Heap](./large-object-heap-loh.md), [Allocations and boxing](./allocations-and-boxing.md), [Low-allocation patterns](./low-allocation-patterns-modern-dotnet.md)._

A field guide to the shapes of code that allocate on the managed heap. Pattern recognition matters more than micro-benchmarks here — once you can scan a method and instantly tag every line that emits `newobj`/`newarr`, you can go fix the actual problem.

## The five sources of heap allocations

Every byte on the managed heap traces back to one of these:

1. **`new` on a class** — anything that `: object` (which is everything that isn't a struct).
2. **Array creation** — `new T[n]`, including the hidden ones inside `params`, `ToArray()`, `List<T>` resize.
3. **Boxing** — value type → `object` / interface. (See [Allocations and boxing](./allocations-and-boxing.md).)
4. **Closure capture** — lambdas that close over locals or `this` allocate a compiler-generated **display class**.
5. **String operations** — concat, format, `Substring`, `Trim`, `Replace`, `Split`, `ToString` all return new strings.

Everything else — `List<T>.Add`, `Dictionary<K,V>` insertion, `async Task<T>` method invocation, LINQ `Where().Select()` — decomposes into one or more of those five.

## SOH allocation pattern

The Small Object Heap is the default. `new C()` for any class smaller than the LOH threshold lands here.

```csharp
var dto = new UserDto { Id = id, Name = name };  // SOH, Gen0
```

Lifecycle:

1. Bump the Gen0 allocation pointer; if it overflows the budget, fire a Gen0 collection.
2. If the object survives that collection → promoted to Gen1. Survives Gen1 → Gen2.
3. Each promotion costs one GC pass that has to mark and copy the object.

**The weak generational hypothesis** says most objects die young. SOH is fast precisely because Gen0 collections are cheap **as long as your objects die before the next Gen0**. The cost of a SOH allocation is dominated by promotion, not by the alloc itself.

## LOH allocation pattern

Any allocation **≥ 85,000 bytes** goes directly to the Large Object Heap, which is itself part of Gen2:

```csharp
byte[] buffer = new byte[100_000];     // LOH (Gen2) — straight to old generation
string giant  = new string('x', 50_000); // 50_000 * 2 + header > 85k → LOH
List<int> list = new(capacity: 25_000);  // 25_000 * 4 = 100k → LOH from the start
```

Consequences you will feel:

- LOH allocations skip the Gen0/Gen1 filter. Every one is effectively a future Gen2 trigger.
- The LOH is **collected with Gen2** — long pauses on workstation GC, parallel pauses on server GC.
- Historically the LOH was not compacted. Modern .NET can compact on demand (`GCSettings.LargeObjectHeapCompactionMode = CompactOnce`), but it still doesn't *automatically* compact. Fragmentation is real (see [LOH fragmentation](./loh-fragmentation.md)).
- A common case: `List<T>` doubles its capacity. If the underlying array crosses 85k bytes during growth, every subsequent resize is on the LOH.

```csharp
//  ints: 4 bytes each → 21,250 ints crosses LOH (85,000 / 4 = 21,250)
var nums = new List<int>();      // starts SOH
for (int i = 0; i < 50_000; i++) nums.Add(i);
// Doubling: 4, 8, 16, ..., 16384, 32768 → 32768 ints * 4 = 131,072 bytes → next backing array on LOH
```

Pre-size with `new List<int>(50_000)` to allocate the LOH array exactly **once**.

## Hidden allocations: a scanning table

| Pattern | What allocates | Senior fix |
|---|---|---|
| `string s = a + b + c;` in a loop | a fresh string per `+` | `StringBuilder` (pool it for hot paths) or `string.Create<TState>(len, state, action)` |
| `obj.ToString()` on a struct via `object` | boxing (see boxing note) | generic constrained API; `ISpanFormattable.TryFormat` |
| `IEnumerable<T>` parameter, caller passes `List<T>` | the enumerator boxes; LINQ chains add iterator+delegate allocations | take `List<T>` / `IReadOnlyList<T>`, or write a generic `where TList : IEnumerable<T>` if you really need polymorphism |
| `Func<T,U>` capturing locals | display class + delegate | static lambda + explicit `state` arg, or method group reference |
| `Task.FromResult(value)` on hot paths | `Task<T>` instance per call | `ValueTask<T>` with sync-completion path; cache `Task.CompletedTask` |
| `params T[] args` calls | a fresh array per call | `ReadOnlySpan<T>`-typed `params` overload (C# 13: `params ReadOnlySpan<T>`) |
| `items.Where(p).Select(s).ToList()` | iterator class per stage + delegate(s) + final list | hand-roll the loop in hot code; or use `Span<T>`-aware helpers |
| `Tuple<int,int>` returns | class allocation | `(int, int)` `ValueTuple` |
| `string.Split(',')` | array of strings | `MemoryExtensions.Split` (`ReadOnlySpan<char>` based, no allocs) |
| `myObj.ToString()` for logging | formatted string even if log level is filtered | `LoggerMessage` source generator; `ILogger`'s interpolated handler |

## List/Dictionary growth

Both grow by **doubling** the backing array when full. The old array becomes garbage; the new one is freshly allocated. For a list reaching N elements, this means ~N total bytes copied across all resizes (a geometric series), and **log₂(N)** discarded backing arrays in Gen0/Gen1.

```csharp
// Pre-size if N is known.
var ids = new List<long>(capacity: expectedCount);

// Or grow once explicitly. EnsureCapacity returns the new (possibly larger) capacity.
ids.EnsureCapacity(expectedCount);
```

`Dictionary<K,V>` is similar but uses prime-sized buckets (next prime after doubled capacity). Pre-sizing eliminates the rehash cost (which copies every entry into new buckets) **and** the Gen0 garbage from the abandoned arrays.

## String allocation patterns

Strings are immutable, so every "modification" is a new string:

```csharp
s.Substring(2, 5)        // alloc
s.Trim()                 // alloc (unless input had no whitespace, which is special-cased)
s.Replace('a', 'b')      // alloc
s.ToUpper()              // alloc
s + "x"                  // alloc
$"value={x}"             // depends — see InterpolatedStringHandler
```

The single-pass primitive is `string.Create<TState>(int length, TState state, SpanAction<char, TState> action)`:

```csharp
string FormatId(long id) => string.Create(11, id, static (span, value) =>
{
    span[0] = 'U';
    value.TryFormat(span[1..], out _, "D10");
});
```

One allocation, exact size, no intermediate strings. Use it for hot-path identifier formatting.

For literal byte sequences (protocols, JSON keys), `"abc"u8` produces a `ReadOnlySpan<byte>` backed by RVA static data — **zero allocation, ever**.

## Async state-machine allocations

`async Task<T>` foo:

- A reference-typed `AsyncTaskMethodBuilder<T>` box holds the state machine.
- The builder allocates a `Task<T>` lazily — only if the method actually awaits something incomplete.
- A method that *always* completes synchronously (cached value, fast path) still pays the box if the method is `async Task<T>` and the JIT can't prove sync completion.

`async ValueTask<T>` is the standard fix when sync-completion is the common case:

```csharp
public ValueTask<User?> GetAsync(long id)
{
    if (_cache.TryGetValue(id, out var u)) return new ValueTask<User?>(u);  // no alloc
    return new ValueTask<User?>(LoadAsync(id));                              // alloc only on miss
}
```

For very hot async methods, opt into pooled state machines:

```csharp
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
public async ValueTask<int> HotPath() { ... }
```

`ValueTask` has a "consume once" contract — never `await` it twice, never `.AsTask()` an already-awaited one.

**Senior-level gotchas:**

- Pre-sizing `List<T>` / `Dictionary<K,V>` / `StringBuilder` is the single highest-leverage allocation fix in most services. It costs nothing, removes garbage, and avoids LOH crossings.
- LOH allocations are **Gen2 from birth**. Treat the 85,000-byte threshold as a tripwire — any allocation that flirts with it deserves a `// LOH` comment and a pool.
- `ToArray()` and `ToList()` at API boundaries materialize the entire sequence — fine for stable APIs, fatal in pipelines. Prefer streaming (`IAsyncEnumerable<T>`, `IEnumerable<T>` with deferred execution).
- A `class` field inside a `struct` defeats half the value-type benefit if the struct ends up boxed, on the heap, or in a `class` array — the reference is still on the heap. Use values for **leaf** data, not as a wrapper around references.
- Pooling small short-lived objects rarely beats Gen0. The GC's bump allocator is faster than your pool's lookup. Pool only when (a) the object is large, (b) construction is expensive, or (c) the object holds a native handle. Measure with `[MemoryDiagnoser]` first.
- `Dictionary<,>` with default capacity allocates entries + buckets on the **first add**, not at construction. Microbenchmarks of "empty dictionary creation" lie about real cost.
- `string.Concat(IEnumerable<string>)` is O(N) when input is a `string[]` (it knows the total length up front) and O(N + reallocs) when input is a generic `IEnumerable<string>` (it has to discover length by iterating into a `StringBuilder`). The overload selected matters.
- `async void` not only cannot be awaited — it has no `Task` to capture exceptions, and the state machine still allocates. Use it only for top-level event handlers.
- `Parallel.ForEach` over a small loop body allocates per-thread state + delegates. For small bodies, a plain `for` is faster and allocation-free.
- A `record class` with primary constructor and value equality **still allocates**. Use `record struct` if you wanted value semantics without the heap.
