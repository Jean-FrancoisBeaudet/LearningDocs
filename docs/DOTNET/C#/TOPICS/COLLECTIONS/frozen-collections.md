# Frozen collections

_Targets .NET 10 / C# 14. See also: [Array, List, Dictionary, HashSet, Stack, Queue](./array-list-dictionary-hashset-stack-queue.md), [Immutable collections](./immutable-collections.md), [Readonly collections](./readonly-collections.md), [Concurrent collections](./concurrent-collections.md)._

`FrozenDictionary<K,V>` and `FrozenSet<T>` (added in .NET 8) are **read-only** collections optimized for the build-once / read-millions-of-times pattern. The construction step pays an upfront analysis cost; in exchange, lookup is faster — sometimes much faster — than `Dictionary<K,V>` / `HashSet<T>`.

## The shape

```csharp
using System.Collections.Frozen;

FrozenDictionary<string, RouteHandler> routes =
    rawDict.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);

FrozenSet<int> allowedStatusCodes = new[] { 200, 204, 301, 302, 304 }.ToFrozenSet();

if (routes.TryGetValue(path, out var handler)) handler.Invoke(ctx);
if (allowedStatusCodes.Contains(code)) { ... }
```

The API surface is just `IReadOnlyDictionary<K,V>` / `IReadOnlySet<T>` plus a few extras (`GetAlternateLookup`). There is **no** `Add`, `Remove`, or `Clear`. To "modify," rebuild from a mutable source.

## What "frozen" buys you

`ToFrozenDictionary` / `ToFrozenSet` inspect the data and pick a specialized representation:

- **Small sets** (≤ a handful of items): linear search beats hashing because hash + index + indirection costs more than 4 compares.
- **Integer keys**: range analysis can produce near-perfect hashing or direct array indexing.
- **String keys**: the builder analyzes lengths and character positions to find a discriminating substring (e.g., "all keys differ at index 3" → hash only that one char). For comparers like `StringComparer.Ordinal[IgnoreCase]`, this often yields a 2–4× lookup speedup.

The construction cost is real — for a 10k-entry dictionary, expect tens of milliseconds. Amortize it over millions of reads. Don't call `ToFrozenDictionary` per request.

## Canonical use cases

- Routing tables (HTTP routes, message handlers, command dispatch).
- Feature flags loaded at app startup.
- Static enum / string lookup tables (country codes, MIME types, allowed values).
- Compiled config snapshots that update on a reload signal — atomic-swap a `volatile` field to the new frozen instance.

```csharp
public sealed class FlagSnapshot
{
    private volatile FrozenDictionary<string, bool> _flags =
        FrozenDictionary<string, bool>.Empty;

    public bool IsEnabled(string key) =>
        _flags.TryGetValue(key, out var v) && v;

    // Called by reload pipeline; readers always see a fully-built dictionary.
    public void Reload(IDictionary<string, bool> newFlags) =>
        _flags = newFlags.ToFrozenDictionary(StringComparer.Ordinal);
}
```

The `volatile` swap is the entire concurrency story — no locks, no `ConcurrentDictionary`. Readers get either the old or the new map atomically.

## When NOT to use frozen

- The data mutates after construction. Period — there's no `Add`.
- The collection has very few items (≤ ~3) **and** is constructed in a tight loop. The build cost will dominate any lookup savings.
- You're going to enumerate it more than look it up. Frozen is tuned for `TryGetValue` / `Contains`; iteration is not its forte.
- You actually need a snapshot but with mutation later — that's `ImmutableDictionary` (see [Immutable collections](./immutable-collections.md)).

## `GetAlternateLookup<TAlternate>` (.NET 9)

Look up a `string`-keyed `FrozenDictionary` by `ReadOnlySpan<char>` with no string allocation:

```csharp
var routes = data.ToFrozenDictionary(StringComparer.Ordinal);
var alt = routes.GetAlternateLookup<ReadOnlySpan<char>>();

ReadOnlySpan<char> path = request.PathSpan;     // slice of the request buffer
if (alt.TryGetValue(path, out var handler)) { ... }
```

This is a hot-path API for parsers, routers, and dispatchers. Same trick exists on `Dictionary<K,V>`, `HashSet<T>`, and `ConcurrentDictionary<K,V>`.

## Comparison: `Dictionary` vs `Frozen` vs `Immutable` vs `ReadOnly`

| Type | Mutation API | Build cost | Lookup speed | Memory | Thread-safety |
|---|---|---|---|---|---|
| `Dictionary<K,V>` | Add/Remove | Cheap | Baseline | Lowest | Single-thread only |
| `FrozenDictionary<K,V>` | None | Expensive (one-time) | Fastest | Slightly higher | Safe (immutable bits) |
| `ImmutableDictionary<K,V>` | Returns new instance | Cheap incremental | Slower (HAMT walk) | Higher (tree nodes) | Safe |
| `ReadOnlyDictionary<K,V>` | None on the wrapper | Free | Same as wrapped Dictionary | + wrapper | As safe as wrapped |

Rule of thumb: **build-once, read-forever → Frozen**. **Need to "mutate" via copy-on-write → Immutable**. **Want a read-only view of a still-mutable Dictionary → ReadOnly**.

## A quick perf intuition

Order-of-magnitude steady-state lookup, 1k entries, modern x64, varies by data distribution:

| Container | `TryGetValue` rough cost |
|---|---|
| `Dictionary<string, V>` (Ordinal) | ~25–35 ns |
| `FrozenDictionary<string, V>` (Ordinal) | ~10–18 ns |
| `ImmutableDictionary<string, V>` | ~80–120 ns |

Numbers are vibes — measure with `BenchmarkDotNet` and `[MemoryDiagnoser]` against your actual key distribution. The Frozen advantage is largest for `string` and `int` keys with `IEqualityComparer<T>.Default` / `StringComparer.Ordinal[IgnoreCase]`.

**Senior-level gotchas:**
- `ToFrozenDictionary()` is **not** cheap. Calling it inside a request handler is a perf bug masquerading as ergonomics — build at startup or at config-change time only.
- Pass the **right comparer** at build time. `StringComparer.OrdinalIgnoreCase` enables specialized fast paths; a custom comparer disables them and falls back to a generic implementation that may be slower than `Dictionary`.
- `FrozenDictionary<K,V>.Empty` is a singleton — use it for "no flags loaded yet" sentinels rather than allocating an empty one each call.
- The build inspects your **specific data**. Add 10 more entries and the chosen representation may switch (linear → hashed → perfect-hashed). Don't assume the same lookup cost across builds.
- Frozen does not protect you from **value-side mutation**. `FrozenDictionary<string, List<int>>` is "frozen" in its key→list mapping; the lists themselves are still freely mutable, including by other threads. If you need value immutability, freeze the values too (`ImmutableArray<int>`, records).
- `FrozenSet<T>.Contains` over a 3-element set is sometimes **slower** than a hand-written `if x == a || x == b || x == c`. The framework's small-set path is good but not magic — for very tiny inline checks, just write the chain.
- There is no `FrozenList<T>` — `IReadOnlyList<T>` over an array (or `ImmutableArray<T>`) is what you want. The "frozen" optimizations target hash lookup; for index-by-int, an array is already optimal.
- Frozen collections are **read-only**, not **immutable** — the distinction matters for documentation. They have no mutation API, but the **values** they hold can mutate. Use `ImmutableDictionary` if you need both axes locked.
- The .NET 8 release notes called Frozen "specialized for cases where reads vastly outnumber writes." Read the constructor cost as a one-time tax, not a per-update one — if your config flips every minute, you're rebuilding constantly and `Dictionary` may win net.
