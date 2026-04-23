# Aggregate functions

_Targets .NET 10 / C# 14. See also: [IEnumerable vs IQueryable](./ienumerable-vs-iqueryable.md), [Filtering, Projections, Sorting](./filtering-projections-sorting.md), [Pagination and Grouping](./pagination-and-grouping.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md)._

`Sum`, `Count`, `Min`, `Max`, `Average`, `Aggregate` — and the `*By` family. They look like one-liners and are where most "off by 1", silent overflow, and "Average over 0 rows" bugs live.

## The standard aggregates

```csharp
int    c   = items.Count();                    // O(n) for IEnumerable, O(1) for ICollection
long   l   = items.LongCount();                // counts past int.MaxValue
int    s   = numbers.Sum();                    // unchecked — silent overflow
int    max = numbers.Max();
int    min = numbers.Min();
double avg = numbers.Average();                // returns double for int/long sources
```

Predicate overloads on `Count` and `LongCount` give you "count where ...":

```csharp
int active = users.Count(u => u.IsActive);     // one pass, no intermediate Where allocation
```

There's no `Sum`/`Min`/`Max`/`Average` predicate overload; you do `items.Where(p).Sum(s)` (which fuses, see [Filtering](./filtering-projections-sorting.md)).

### Selector overloads

```csharp
decimal total   = orders.Sum(o => o.Amount);
DateTime latest = events.Max(e => e.Timestamp);
double avgAge   = users.Average(u => u.Age);
```

Selectors avoid building an intermediate `IEnumerable<TResult>` — one pass over source, one accumulator.

### Empty-sequence behavior

| Operator | Empty source | Notes |
|---|---|---|
| `Count` | `0` | Always defined |
| `Sum` | `0` (or `0.0`, `0m`) | Always defined |
| `Min` / `Max` | **throws** `InvalidOperationException` | Use `MinBy`/`MaxBy` and check for null, or `DefaultIfEmpty` |
| `Average` | **throws** `InvalidOperationException` | Same |
| `Aggregate` (no seed) | **throws** `InvalidOperationException` | Use the seeded overload |

```csharp
// Safe pattern: project to nullable and use the nullable overload.
decimal? maxOrZero = orders.Max(o => (decimal?)o.Total) ?? 0m;

// Or DefaultIfEmpty.
decimal max2 = orders.Select(o => o.Total).DefaultIfEmpty(0m).Max();
```

## `MinBy` / `MaxBy` (.NET 6+)

```csharp
Order? mostExpensive = orders.MaxBy(o => o.Total);   // returns the Order, not the decimal
```

Returns the **source element** that has the min/max projected key — the right answer for "which row had the largest X". On an empty source, returns `null` for reference types and `default(T)` for value types (does not throw, unlike `Max(selector)`). Watch the value-type case: `default(int)` is `0`, which can be a valid value — distinguish "empty" via `Any()` first.

## Overflow

```csharp
int[] big = [int.MaxValue, 1];
int s = big.Sum();          // ⚠ silently produces a large negative number — unchecked overflow
```

`Enumerable.Sum<int>` is implemented in an `unchecked` block. To detect overflow:

```csharp
checked
{
    int s2 = big.Aggregate(0, (acc, x) => acc + x);   // throws OverflowException
}
// Or sum into a wider accumulator:
long total = big.Sum(x => (long)x);
```

`Sum<long>` has the same trap. `Sum<decimal>` throws `OverflowException` on overflow; `Sum<double>` follows IEEE and returns `±Infinity`.

## `Aggregate` — the fold

Three overloads:

```csharp
// 1. No seed — first element is the seed. Throws on empty.
int product = numbers.Aggregate((acc, x) => acc * x);

// 2. With seed.
int productSafe = numbers.Aggregate(1, (acc, x) => acc * x);

// 3. Seed + result selector — for accumulator/result type split.
string concat = words.Aggregate(
    seed:           new StringBuilder(),
    func:           (sb, w) => sb.Append(w).Append(", "),
    resultSelector: sb => sb.ToString().TrimEnd(',', ' '));
```

The first overload's "seed-from-source" behavior is a bug magnet: throws on empty, and the seed and elements share a type so you can't fold into an accumulator of a different type. Reach for the seeded overload by default.

`Aggregate` does **not** translate to SQL. EF Core can do `Sum`/`Min`/`Max`/`Average` because they're known reductions; `Aggregate` is opaque and forces client evaluation (which EF Core will refuse — must be after `AsEnumerable()`).

When you find yourself writing `Aggregate`, check whether a `foreach` is clearer:

```csharp
// Aggregate.
var report = events.Aggregate(
    new Report(),
    (r, e) => { r.Apply(e); return r; });

// foreach — same code, easier to debug, no closure allocation.
var report2 = new Report();
foreach (var e in events) report2.Apply(e);
```

## EF Core translation

| Operator | Translates to | Notes |
|---|---|---|
| `Count` / `LongCount` | `COUNT(*)` / `COUNT_BIG(*)` | Predicate becomes `COUNT(CASE WHEN ... THEN 1 END)` |
| `Sum` / `Min` / `Max` / `Average` | `SUM`, `MIN`, `MAX`, `AVG` | Empty-set semantics depend on DB (often `NULL`, mapped to nullable result) |
| `Any` | `EXISTS (SELECT 1 ...)` | Short-circuits, doesn't need a full scan |
| `All` | `NOT EXISTS (SELECT 1 ... WHERE NOT predicate)` |  |
| `Contains` | `IN (...)` (constant list) or `EXISTS` (subquery) |  |
| `MinBy` / `MaxBy` | `ORDER BY ... ASC/DESC FETCH 1 ROW` | EF Core 7+ |
| `Aggregate` | does **not** translate | Force-fetch and aggregate client-side |

### `Count()` vs `Any()`

```csharp
if ((await q.CountAsync(ct)) > 0) ...    // SELECT COUNT(*) — full scan
if (await q.AnyAsync(ct)) ...            // SELECT 1 WHERE EXISTS (...) — short-circuits
```

`Any` is the right operator for "is there at least one". `Count() > 0` forces the database to count every match before returning, which on a 100M-row table is criminally slow. The same applies in LINQ-to-Objects: `Any` short-circuits at the first match; `Count` always walks the full sequence.

## High-throughput aggregates

LINQ aggregates over `IEnumerable<T>` allocate an iterator and dispatch through interfaces; the JIT can't autovectorize. For numeric arrays/lists, the senior pattern is a `for` loop over a span:

```csharp
ReadOnlySpan<int> data = numbers.AsSpan();
long sum = 0;
for (int i = 0; i < data.Length; i++)
    sum += data[i];
```

For larger arithmetic kernels, `System.Numerics.Tensors.TensorPrimitives` (.NET 8+) offers SIMD-vectorized reductions:

```csharp
using System.Numerics.Tensors;

float sum = TensorPrimitives.Sum<float>(values);
float min = TensorPrimitives.Min<float>(values);
float dot = TensorPrimitives.Dot<float>(a, b);
TensorPrimitives.Add<float>(a, b, dest);          // vectorized element-wise add
```

These are typically 4–10× faster than the LINQ equivalent on the same hardware. Trade-off: you lose the operator chaining; the call sites are more verbose.

**Senior-level gotchas:**
- `Sum<int>` overflows silently. Width your accumulator (`Sum(x => (long)x)`) or wrap in `checked { ... Aggregate(...) }`.
- `Average` over an empty source throws. The senior idiom is `items.DefaultIfEmpty(0).Average()` or a `Count == 0` guard. Returning 0 for "no data" is a domain decision — don't assume.
- `MaxBy` / `MinBy` return `default(T)` (not throw) on empty. For value types, `default` may collide with valid data — gate on `Any()` first.
- `Count() > 0` on `IQueryable<T>` is the most common LINQ perf bug in production. Use `Any()` / `AnyAsync()`.
- The seedless `Aggregate(func)` throws on empty *and* forces seed type to equal element type. Always prefer `Aggregate(seed, func)` or `Aggregate(seed, func, resultSelector)`.
- `Count()` on a `string` (which is `IEnumerable<char>`) returns the **char count**, not the grapheme count. For user-facing length, use `StringInfo.LengthInTextElements` or `Rune` enumeration.
- `Count` on an `ICollection<T>` source is O(1) — the BCL special-cases it. On an `IEnumerable<T>` over a generator, `Count` consumes the entire sequence. For "did we get exactly N", `items.Take(N + 1).Count() == N` or `TryGetNonEnumeratedCount` (.NET 6+) avoids surprise enumeration.
- `Min`/`Max` on a sequence containing `double.NaN` returns `NaN` (because NaN compares unordered). Filter out `NaN` first or use `Where(d => !double.IsNaN(d))`.
- `Sum` on `IEnumerable<decimal?>` ignores nulls (treats as 0); same for the other nullable-aware overloads. In SQL, `SUM` over all-NULLs returns `NULL`, not 0 — so an EF Core `SumAsync` with all nulls may surprise you with `null` mapping back to `0` (depending on whether the projection is nullable).
- For "top N by score", use `OrderByDescending(...).Take(N)` — the BCL has a `TopN` optimization for the `OrderBy(...).Take(n)` pattern since .NET 7; you don't need a hand-rolled heap.
- `Aggregate` with a mutable accumulator (e.g., `StringBuilder`) and the resultSelector overload is cleaner than seedless `Aggregate` returning a new immutable object on each step (which allocates O(n) intermediates).
