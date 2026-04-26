# LINQ performance misuse

_Targets .NET 10 / C# 14. See also: [Closure captures](./closure-captures.md), [Allocations and boxing](./allocations-and-boxing.md), [GC pressure](./gc-pressure.md), [.NET performance anti-patterns](./dotnet-performance-anti-patterns.md)._

LINQ is the easiest way in .NET to write code that *looks* fine and allocates a thousand objects per call. The cost lives in three places: each chained operator allocates an enumerator and a delegate; every captured local lifts to a display class; and on `IQueryable` it can swap a one-row server query for a million-row client materialization. None of this shows up in source. All of it shows up in `% time in gc`, p99 latency, and `dotnet-counters alloc-rate`.

## The cost model in one paragraph

A `foreach` over a chain like `items.Where(...).Select(...).Take(10)` allocates: one enumerator per operator (3), one `Func<>` delegate per lambda unless the lambda captures nothing and the compiler caches it (up to 2), and one display class per capturing lambda. With one captured local in `Where`, every call allocates ~5 short-lived objects before the loop body runs. Over a per-request hot path that's measurable; in BenchmarkDotNet's `[MemoryDiagnoser]` it shows up as a non-zero `Allocated` column on what should be a zero-alloc benchmark.

## Multiple enumeration

`IEnumerable<T>` is lazy. Re-enumerating re-runs the whole pipeline — including any side effect, DB round trip, or HTTP call.

```csharp
IEnumerable<Order> orders = db.Orders.Where(o => o.IsActive);   // not executed yet

if (orders.Any())                                  // round trip 1
    Console.WriteLine($"{orders.Count()} active"); // round trip 2
foreach (var o in orders) { ... }                  // round trip 3
```

Three SQL queries for what looks like one. Materialize once when you need the data more than once:

```csharp
var orders = await db.Orders.Where(o => o.IsActive).ToListAsync(ct);
if (orders.Count > 0) Console.WriteLine($"{orders.Count} active");
foreach (var o in orders) { ... }
```

Roslyn analyzer **CA1851** (.NET 7+) flags this — opt in via `<AnalysisMode>All</AnalysisMode>` or per-rule severity in `.editorconfig`.

## `.Count() > 0` and friends

`Count()` walks the entire sequence on a generic `IEnumerable<T>`. `Any()` short-circuits at element one.

| Pattern | Cost on `IEnumerable<T>` | Cost when source has `Count` (List, Array, ICollection) |
|---|---|---|
| `seq.Count() > 0` | full enumeration | O(1) — `Count()` checks `ICollection.Count` first |
| `seq.Any()` | one `MoveNext` | one `MoveNext` |
| `list.Count > 0` | n/a — direct property | O(1) |
| `seq.TryGetNonEnumeratedCount(out var n)` | O(1) without enumeration | O(1), returns `true` |

`TryGetNonEnumeratedCount` (.NET 6+) is the disciplined version when you only want a count if it's free. Roslyn analyzers **CA1827** (`.Any()` instead of `.Count()`) and **CA1829** (use `Length`/`Count` property) catch the common shapes.

## `OrderBy(...).First()` is `O(n log n)` to find one element

```csharp
var oldest = users.OrderBy(u => u.CreatedAt).First();   // sorts ALL of them
```

`MinBy` / `MaxBy` (.NET 6+) are O(n) and don't allocate intermediate arrays:

```csharp
var oldest = users.MinBy(u => u.CreatedAt);
```

Same trap with `OrderBy(...).Take(k)` for top-k — use a partial sort (`PriorityQueue<T, TPriority>` from `System.Collections.Generic`) when k ≪ n.

## `Where(p).First()` vs `First(p)`

```csharp
items.Where(x => x.Active).First();   // 2 enumerators, 1 delegate (or display class)
items.First(x => x.Active);           // 1 enumerator, 1 delegate
```

Same result, half the operator overhead. `Single`/`Any`/`Count` all have predicate overloads — use them.

## `Single` vs `First` — pick the one you mean

`Single` enumerates **past** the first match to verify uniqueness; `SingleOrDefault` does the same. `First` stops on element one. If "more than one is impossible by construction", use `First` and pay one MoveNext, not n.

## `.ToList()` placement

```csharp
db.Orders.ToList().Where(o => o.IsActive).Select(o => o.Total).Sum();
```

`ToList()` materializes the entire table into memory before filtering. The aggregate happens client-side. Always keep operators on the `IQueryable` until the last call:

```csharp
await db.Orders.Where(o => o.IsActive).SumAsync(o => o.Total, ct);
```

The opposite mistake is materializing **too late** — keeping `IEnumerable<T>` across an enumeration boundary that will iterate twice. Trade allocation for clarity once you know you'll re-iterate.

## IEnumerable vs IQueryable — the silent client-eval bomb

EF Core translates as long as the call chain stays on `IQueryable<T>`. The moment you call something that takes `IEnumerable<T>` (`AsEnumerable()`, custom extension methods, `.Select(...)` over a non-translatable expression), translation stops and the rest runs in process.

```csharp
db.Orders
  .Where(o => o.IsActive)
  .AsEnumerable()                                     // ← server query ends here
  .Where(o => MyHelper(o.CustomerName))               // runs in process, full table client-side
  .ToList();
```

EF Core 3.0+ throws by default when a `Where`/`OrderBy` can't be translated, but `Select` projecting into a method call still triggers client eval. Audit with `db.Database.SetCommandTimeout` + EF logging at `Information` — every `Executed DbCommand` should be the *outer* query, not a row-scan loop.

## Hidden allocations from captures

Every captured local turns the lambda into a display class allocation per call. See [closure-captures.md](./closure-captures.md) for the IL-level detail. Two practical rules:

```csharp
// Bad — `threshold` captured into Where, allocates per invocation.
int threshold = config.Threshold;
items.Where(x => x.Value > threshold);

// Good — pass via state-arg overload (e.g. dict.GetOrAdd), or hoist the predicate to a static field.
private static readonly Func<Item, int, bool> Filter = static (x, t) => x.Value > t;
```

Mark hot-path lambdas `static` (C# 9+) — the compiler errors instead of silently allocating if a future edit introduces capture.

## `Distinct()` on reference types

```csharp
items.Distinct();   // reference equality unless T overrides Equals/GetHashCode
```

`HashSet<T>` under the hood uses `EqualityComparer<T>.Default`. For records, that's by-value (good). For classes with no override, that's by-reference (almost never what you mean). Pass an explicit `IEqualityComparer<T>` or use `DistinctBy(x => x.Id)` (.NET 6+).

## LINQ in hot loops

LINQ in a per-request loop is the highest-volume allocation source in most ASP.NET Core profiles. Drop to `foreach` + `if`, or `Span<T>` for primitives:

```csharp
// Allocates ≥ 4 objects per call.
var sum = numbers.Where(n => n > 0).Sum();

// Zero allocation, often 3–5× faster.
long sum = 0;
foreach (var n in numbers) if (n > 0) sum += n;
```

For numeric reductions, `MemoryExtensions.Sum`, `Vector<T>` SIMD, or `TensorPrimitives` (.NET 8+) crush LINQ on throughput. Reserve LINQ for cold paths and query expressions.

## New operators (and traps)

| Operator | Since | Use |
|---|---|---|
| `MinBy` / `MaxBy` | .NET 6 | O(n) min/max by key — replaces `OrderBy().First()` |
| `DistinctBy` / `UnionBy` / `IntersectBy` / `ExceptBy` | .NET 6 | Set ops by key without ad-hoc comparers |
| `Chunk(size)` | .NET 6 | Batch into arrays — replaces `Select((x,i) => new {x,i}).GroupBy(...)` |
| `TryGetNonEnumeratedCount` | .NET 6 | Cheap-only count |
| `Order()` / `OrderDescending()` | .NET 7 | Cleaner than `OrderBy(x => x)` for natural ordering |
| `CountBy` / `AggregateBy` | .NET 9 | Histogram / fold-by-key without intermediate `GroupBy` |
| `Index()` | .NET 9 | `(int Index, T Item)` pairs — no display class for the counter |

`Chunk` allocates one array per chunk (size `size`); on a 100 M-row stream that's 100 M / `size` allocations — sometimes the fix, sometimes the new bottleneck.

## Tooling

| Tool | What it shows |
|---|---|
| BenchmarkDotNet `[MemoryDiagnoser]` | `Allocated` column — non-zero on a "no-alloc" benchmark = closure or operator |
| `Microsoft.CodeAnalysis.PerformanceSensitiveAnalyzers` | HAA0301 (closure alloc), HAA0302 (display class), HAA0401 (boxing), HAA0501 (delegate alloc), HAA0502 (capture in lambda) |
| Roslyn .NET analyzers | CA1827 (`Any` over `Count() > 0`), CA1829 (`Length`/`Count` property), CA1851 (multiple enumeration), CA1860 (prefer `Length` over `.Count()`) |
| PerfView "GC Heap Alloc Stacks" | Filter by `<>c__DisplayClass*` and `WhereSelectListIterator*` types |
| `dotnet-counters` | `alloc-rate` and `% time in gc` deltas before/after a LINQ refactor |

## Senior-level gotchas

- **`Single` enumerates past the first match.** Always slower than `First` when uniqueness is structural. Reserve for cases where catching duplicate-data bugs is the *point*.
- **Deferred execution + `using` = lifetime bug.** `IEnumerable<T> q = db.Orders.Where(...);` outside a `using var db = ...` block disposes the context before iteration; the LINQ chain throws `ObjectDisposedException` on first `MoveNext`. Materialize inside the `using` block.
- **Captured locals poison the EF Core compiled-query cache key.** A `Where(x => x.TenantId == _tenantId)` from a per-request `DbContext` produces a *new* compiled query per scope unless the captured value is reference-equal across requests. Watch the `Microsoft.EntityFrameworkCore.Query` warnings about query plan cache pollution.
- **`OrderBy(...).First()` is O(n log n).** Even when the consumer takes one element, the sort has already allocated and walked the whole sequence. `MinBy`/`MaxBy` are O(n).
- **`Count()` vs `LongCount()`.** On sequences > `int.MaxValue`, `Count()` overflows silently to a negative number — `LongCount()` is the safe choice for unbounded streams.
- **`.AsEnumerable()` is a query-boundary marker, not a no-op.** Anything after it runs client-side. Use it deliberately when you *want* in-process logic; never as a "shut up the analyzer" fix.
- **`Where(p1).Where(p2)` doesn't fuse.** Two enumerators, two delegate slots. Combine into `Where(x => p1(x) && p2(x))` on hot paths if profiling shows the iterator chain dominates.
- **`Select(...).Select(...)` does fuse on `List<T>`/`T[]`** in modern .NET via specialized iterators (`Select2Iterator` etc.) — measure before "optimizing" by hand. The general iterator path doesn't fuse.
- **`ToHashSet()` defaults to `EqualityComparer<T>.Default`.** Same trap as `Distinct()` for reference types. Specify the comparer explicitly when the values cross a boundary (DTO, JSON, EF projection).
- **Method-group conversions still allocate a delegate.** `items.Where(IsActive)` allocates a `Func<T, bool>` wrapping `IsActive` per call site invocation; the JIT caches it only when the target method is `static` *and* the call site warms enough for tier-1 optimizations. Don't assume "no lambda" means "no allocation".
- **`async`-projecting LINQ is two allocations per element.** `items.Select(async x => await Process(x))` produces a `Task` *and* a state-machine box per element. Use `Task.WhenAll(items.Select(...))` only when degree-of-parallelism is bounded; otherwise `Parallel.ForEachAsync` (.NET 6+) caps the worker count.
- **`Aggregate` without a seed throws on empty.** `Aggregate((a, b) => a + b)` on an empty sequence throws `InvalidOperationException`; the seed-providing overload returns the seed. Production code on user-controlled inputs always uses the seed overload.
