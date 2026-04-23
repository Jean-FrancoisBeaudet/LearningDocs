# Filtering, Projections, Sorting

_Targets .NET 10 / C# 14. See also: [IEnumerable vs IQueryable](./ienumerable-vs-iqueryable.md), [Pagination and Grouping](./pagination-and-grouping.md), [Aggregate functions](./aggregate-functions.md), [Classes, records, structs](../OOP/classes-records-structs.md), [Member access, type testing, pattern matching, collection expressions](../EXPRESSIONS/member-access-type-testing-pattern-matching-collection-expressions.md)._

The everyday LINQ — `Where`, `Select`, `OrderBy`. Senior view: which overload allocates, which doesn't translate to SQL, and where projection-shape choices have outsized perf and API consequences.

## Filtering — `Where`

```csharp
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate);
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, int, bool> predicate);
```

The indexed overload (`(x, i) => ...`) is **LINQ-to-Objects only** — EF Core has no row-index concept and will throw at translate time. Same applies to `Select`, `TakeWhile`, `SkipWhile`.

### Iterator allocations

`Where` returns one of three internal iterators picked by source shape:

| Source | Iterator | Why it matters |
|---|---|---|
| `T[]` | `WhereArrayIterator<T>` | Skips enumerator state machine, indexes directly |
| `List<T>` | `WhereListIterator<T>` | Reuses the list's struct enumerator |
| `IEnumerable<T>` | `WhereEnumerableIterator<T>` | Generic case, boxes the source enumerator |

Chaining stays cheap: `Where(p1).Where(p2)` is **fused** by the BCL into a single `WhereXxxIterator` whose predicate is `x => p1(x) && p2(x)`. `Where(...).Select(...)` fuses into `WhereSelectXxxIterator`. So a 5-stage chain doesn't allocate 5 iterator objects — it allocates 1 or 2.

### `OfType<T>` vs `Cast<T>`

```csharp
IEnumerable<object> seq = ...;
seq.OfType<string>()    // filters: yields only strings
seq.Cast<string>()      // throws InvalidCastException on first non-string
```

`Cast<T>` over a typed sequence (`IEnumerable<int>` to `IEnumerable<int>`) returns the source unchanged. Useful when you have an untyped `IEnumerable` and need typed LINQ; useless and confusing otherwise.

## Projections — `Select` and `SelectMany`

```csharp
// Map each element.
var names = customers.Select(c => c.Name);

// Map with index — in-memory only.
var indexed = customers.Select((c, i) => $"{i}: {c.Name}");

// Flatten one-to-many.
var allOrders = customers.SelectMany(c => c.Orders);

// Cartesian (rare — usually a bug).
var pairs = list1.SelectMany(_ => list2, (a, b) => (a, b));

// Correlated — keep the parent and the child.
var rows = customers.SelectMany(c => c.Orders, (c, o) => (c.Name, o.Total));
```

### Project to DTOs, not entities

```csharp
// Bad — full SELECT *, change-tracking entries, lazy-load surprises in callers.
var orders = await db.Orders.Where(o => o.CustomerId == id).ToListAsync(ct);

// Good — explicit columns, lightweight, no tracking needed.
record OrderSummary(int Id, decimal Total, DateOnly PlacedAt);
var summaries = await db.Orders
    .Where(o => o.CustomerId == id)
    .Select(o => new OrderSummary(o.Id, o.Total, o.PlacedAt))
    .ToListAsync(ct);
```

EF Core sees the `Select` and emits `SELECT Id, Total, PlacedAt FROM Orders WHERE ...`. A projected query also doesn't produce change-tracker entries (the result is not a tracked entity), so `AsNoTracking()` is implicit.

### Records vs anonymous types

Anonymous types are great for in-method intermediates; they don't escape the assembly. Records are the right choice the moment a shape needs a name, lives across method boundaries, or appears in a public signature. Both have value-equality; both work with EF Core's projection translation.

```csharp
// Inside a method — anonymous is fine.
var rollup = orders.GroupBy(o => o.CustomerId)
                   .Select(g => new { Id = g.Key, Total = g.Sum(o => o.Total) });

// Crossing a boundary — give it a name.
public sealed record CustomerRollup(int CustomerId, decimal Total);
```

## Sorting — `OrderBy`, `ThenBy`, `Reverse`

```csharp
var ordered = items
    .OrderBy(x => x.LastName)
    .ThenBy(x => x.FirstName)
    .ThenByDescending(x => x.JoinedAt);
```

`OrderBy` returns `IOrderedEnumerable<T>` (or `IOrderedQueryable<T>`); only this type exposes `ThenBy`/`ThenByDescending`. A second `OrderBy` after the first **discards** the first ordering — that's a common source of "why is my third sort key being ignored?" bugs.

### Stability

LINQ-to-Objects' `OrderBy` is **stable** (equal keys preserve source order). SQL `ORDER BY` is **not** stable in general — different DB engines give different guarantees. For pagination (`Skip`/`Take`), always order by something that uniquely tiebreaks (e.g., `OrderBy(x => x.CreatedAt).ThenBy(x => x.Id)`) or pages will appear to skip and duplicate rows under churn. See [Pagination and Grouping](./pagination-and-grouping.md).

### `Reverse()`

In-memory only. EF Core has no inverse-of-`ORDER BY`, so flip the direction directly: `OrderByDescending(x => x.CreatedAt)`.

### `Order` and `OrderDescending` (.NET 7+)

```csharp
var sorted     = numbers.Order();              // OrderBy(x => x)
var sortedDesc = words.OrderDescending();      // OrderByDescending(x => x)
```

Sugar for natural-ordering of the source type. Not translated by EF Core; in-memory use only.

## `Distinct` and `DistinctBy`

```csharp
var unique     = list.Distinct();                              // by element equality
var uniqueOrd  = list.Distinct(StringComparer.OrdinalIgnoreCase);
var byProperty = customers.DistinctBy(c => c.Email);           // .NET 6+
```

`DistinctBy` keeps the first element per key (in source order). EF Core translates `Distinct()` over a projection (`SELECT DISTINCT ...`) but rejects `DistinctBy` — write `GroupBy(...).Select(g => g.First())` for the SQL equivalent.

## .NET 9 additions

```csharp
// Index() — pair each element with its position. Replaces Select((x, i) => (i, x)).
foreach (var (index, item) in items.Index())
    Console.WriteLine($"{index}: {item}");

// CountBy — element counts grouped by a key, in one pass, no GroupBy materialization.
Dictionary<string, int> wordCounts =
    words.CountBy(w => w).ToDictionary(kv => kv.Key, kv => kv.Value);

// AggregateBy — like CountBy but with a custom seed and accumulator per key.
var totals = transactions.AggregateBy(
    keySelector: t => t.AccountId,
    seed:        0m,
    func:        (sum, t) => sum + t.Amount);
```

`CountBy`/`AggregateBy` are the modern answer to the `GroupBy(...).Select(g => g.Sum(...))` pattern: same semantics, no intermediate `IGrouping` objects, single pass.

## Hot-path advice

For tight numeric loops, LINQ chains over `IEnumerable<T>` lose to a hand-written `for` over a `Span<T>` by an order of magnitude or more — every operator allocates an iterator and a closure (if it captures), and the JIT can't inline through the interface boundary.

```csharp
// LINQ — clean, allocates ~1-2 iterator objects per call, can't autovectorize.
int sum = numbers.Where(x => x > 0).Select(x => x * x).Sum();

// Hot-path equivalent — zero alloc, autovectorizable.
var span = CollectionsMarshal.AsSpan(numbers);
int total = 0;
for (int i = 0; i < span.Length; i++)
{
    int x = span[i];
    if (x > 0) total += x * x;
}
```

Keep LINQ for clarity at the API surface; drop to spans where a profiler tells you to.

**Senior-level gotchas:**
- Projecting a navigation property (`db.Orders.Select(o => o.Customer.Name)`) issues a JOIN automatically — no `Include` needed, and `Include` would actually pull the whole `Customer` entity, defeating the projection.
- `Where(x => x != null)` on a `Nullable<T>` source does **not** narrow to `T` for the compiler. Follow with `.Select(x => x!.Value)` or write a `WhereNotNull` extension.
- A second `OrderBy` silently replaces the first. Use `ThenBy`/`ThenByDescending` for secondary keys.
- `Select(x => SomeAsyncMethod(x))` returns `IEnumerable<Task<T>>`. You almost always meant `await Task.WhenAll(items.Select(x => SomeAsyncMethod(x)))`. Watch concurrency limits — for many items, throttle with `Parallel.ForEachAsync` or a `SemaphoreSlim`.
- Reusing a stored `IEnumerable<T>` query enumerates the source twice. If the source has side effects (a generator, an HTTP call, a database read), they run twice. Materialize with `ToList()` if the same data is consumed multiple times.
- EF Core projects nullable navigation paths with explicit null-guards; in LINQ-to-Objects, `Select(x => x.Sub.Inner.Value)` will throw `NullReferenceException` if any link is null. Add `?.` operators or filter first.
- A LINQ chain over `IQueryable<T>` followed by `.AsEnumerable().Where(serverUntranslatablePredicate)` is the right hybrid pattern — narrow on the server, refine in memory. The wrong pattern is `ToList().Where(...)`, which materializes everything matching the server predicate before the in-memory filter.
- `OrderBy` allocates a buffer of all elements (it can't yield until it has seen them all). Streaming pipelines past an `OrderBy` are not actually streaming.
- `Select(x => x with { ... })` on a record source is a clean projection; the compiler emits a copy, no manual constructor call. For positional records, the property names you `with` must match the parameter names exactly.
- `OfType<T>` on a `Nullable<T>` source filters out the `null` boxes — handy way to get a typed sequence from `IEnumerable<int?>` without a `Where` + `Select(x => x.Value)` two-step.
