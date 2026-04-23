# IEnumerable vs IQueryable

_Targets .NET 10 / C# 14. See also: [Filtering, Projections, Sorting](./filtering-projections-sorting.md), [Aggregate functions](./aggregate-functions.md), [Async streams and await foreach](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md), [Iterators and IAsyncDisposable](../GENERAL_TOPICS/iterators-and-iasyncdisposable.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md)._

The single most important LINQ distinction. Two interfaces with the same method names, two completely different execution models, and the boundary between them is where most LINQ performance bugs live.

## Two interfaces, two worlds

```csharp
namespace System.Collections.Generic;
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}

namespace System.Linq;
public interface IQueryable<out T> : IEnumerable<T>, IQueryable
{
    Type           ElementType { get; }
    Expression     Expression  { get; }
    IQueryProvider Provider    { get; }
}
```

`IEnumerable<T>` is **delegate-based** in-process iteration: methods take `Func<T, bool>`, the runtime executes them on local data. `IQueryable<T>` is **expression-tree-based**: methods take `Expression<Func<T, bool>>`, and the provider (EF Core, MongoDB driver, OData client, ...) walks the tree and translates it into something foreign — SQL, KQL, an HTTP query.

Two extension method sets share the same names:

```csharp
namespace System.Linq;
public static class Enumerable
{
    public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate);
}
public static class Queryable
{
    public static IQueryable<T> Where<T>(this IQueryable<T> source, Expression<Func<T, bool>> predicate);
}
```

The compiler picks the overload based on the **static** type of the source. Cast a `DbSet<Order>` to `IEnumerable<Order>` and you've silently switched from "translates to SQL" to "pull all rows, filter in C#".

## Variance

Both are covariant in `T` (`out T`):

```csharp
IEnumerable<string> strings = new[] { "a" };
IEnumerable<object> objects = strings;     // OK — covariant
```

Useful when you publish a sequence-returning API and want callers to consume it as a more general type. See [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md).

## Deferred execution

Building a query allocates an enumerator/expression chain but **does not execute** anything:

```csharp
var query = source.Where(x => x.IsActive)
                  .Select(x => x.Name);
// Nothing has run yet.

foreach (var name in query) { ... }   // ← executes here
var list  = query.ToList();           // ← or here
var first = query.First();            // ← or here
```

Triggers of execution:
- `foreach` (calls `GetEnumerator`)
- Materializing operators: `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`, `ToFrozenSet`/`ToFrozenDictionary`
- Aggregates: `Count`, `Sum`, `Min`, `Max`, `Average`, `Aggregate`
- Element accessors: `First`, `FirstOrDefault`, `Single`, `Last`, `ElementAt`
- Predicate checks: `Any`, `All`, `Contains`

A query enumerated twice **runs twice**. If the source is a database, that's two round trips; if the source is an `IEnumerable<T>` over a generator, the side effects run twice. Materialize once with `ToList()` and reuse the list.

## Crossing the boundary

```csharp
IQueryable<Order> q = db.Orders.Where(o => o.Total > 100);

var bad = q.AsEnumerable()             // forces in-memory mode
           .Where(o => Compute(o));    // C# delegate — runs after full SQL fetch

var worse = q.ToList()                 // round-trips ALL matching orders
             .Where(o => Compute(o));  // then filters in memory
```

`AsEnumerable()` does not execute; it only changes which `Where` overload the compiler picks downstream. Once you've called it, every operator after the call runs locally — over whatever the provider streams to you. Use it deliberately when you need a function the provider can't translate (e.g., a regex or a method with no SQL equivalent), and put as much filtering/projection as possible **before** the boundary.

`AsQueryable()` is the inverse and almost always wrong. On an in-memory `IEnumerable<T>` it wraps you in `EnumerableQuery<T>`, which compiles the expression tree at run time and invokes it. You pay an expression-tree allocation tax for zero benefit. The only legitimate use is in tests where you mock a `DbSet<T>` against a `List<T>` — and even there, `AsAsyncEnumerable()` plus a fake context is usually cleaner.

## EF Core translation

EF Core translates `IQueryable<T>` operators to SQL. Roughly: `Where`, `Select`, `OrderBy`/`ThenBy`, `Skip`, `Take`, `Join`, `GroupJoin`, `GroupBy`, `Distinct`, `Union`/`Intersect`/`Except`, `Any`, `All`, `Count`, `Sum`/`Min`/`Max`/`Average`, `First*`/`Single*`. Indexed overloads (`Where((x, i) => ...)`) and `Aggregate` do **not** translate.

Since EF Core 3.0 the default is **strict**: anything the provider can't translate throws `InvalidOperationException` at query-execution time rather than silently switching to client evaluation. You can scope the relaxation via `AsEnumerable()` at a deliberate boundary; you cannot opt back into the EF Core 2.x "guess what I meant" behavior.

```csharp
// Translates: WHERE o.Total > @p0 AND o.CustomerId = @p1
var q = db.Orders.Where(o => o.Total > 100m && o.CustomerId == id);

// Throws — Math.Round has no SQL mapping the provider knows about for this overload.
var bad = db.Orders.Where(o => Math.Round(o.Total, 2) == 100m);

// Fix: do the rounding server-side via a known function, or AsEnumerable() after narrowing.
var ok = db.Orders.Where(o => o.Total >= 99.995m && o.Total < 100.005m);
```

## Async iteration

`IAsyncEnumerable<T>` is the async sibling of `IEnumerable<T>` (`MoveNextAsync`, `await foreach`). EF Core implements it on its query results so you can stream rows without materializing:

```csharp
await foreach (var order in db.Orders
    .Where(o => o.Total > 100m)
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await Process(order, ct);
}
```

Materializing operators have async variants on `IQueryable<T>` provided by EF Core (`ToListAsync`, `FirstAsync`, `CountAsync`, `AnyAsync`, ...). They are extension methods in `Microsoft.EntityFrameworkCore`; if you forget the using, you'll silently call the sync versions and block a thread-pool thread inside Kestrel.

See [Async streams and await foreach](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md) for the iteration model.

## Allocation profile

| Source type | `foreach` cost | `Where` cost |
|---|---|---|
| `T[]` (typed `T[]`) | array indexer, no enumerator alloc | `WhereArrayIterator` — one alloc per call |
| `List<T>` (typed `List<T>`) | struct enumerator, no alloc | `WhereListIterator` — one alloc per call |
| `IEnumerable<T>` (interface) | boxed enumerator on heap | `WhereEnumerableIterator` — one alloc per call |

Type your local as the concrete collection on hot paths to keep the struct enumerator. The moment you assign to `IEnumerable<T>`, the JIT can no longer specialize and you box the enumerator.

```csharp
List<int> data = ...;
foreach (var x in data) { ... }              // struct enumerator, no alloc

IEnumerable<int> seq = data;
foreach (var x in seq) { ... }               // boxed enumerator, ~32-byte alloc
```

LINQ chains over `IQueryable<T>` are dominated by the round-trip cost; allocation noise is irrelevant. Over `IEnumerable<T>`, every operator allocates an iterator object and boxes its closure if it captures a variable. For numeric hot paths, `for (int i = 0; i < span.Length; i++)` on a `Span<T>` from `CollectionsMarshal.AsSpan(list)` beats any LINQ chain.

## Senior patterns for EF Core

```csharp
// AsNoTracking — no change-tracker entries, no identity resolution.
// Read-only queries should always use it.
var summaries = await db.Orders.AsNoTracking()
    .Where(o => o.CustomerId == id)
    .Select(o => new OrderSummaryDto(o.Id, o.Total, o.PlacedAt))
    .ToListAsync(ct);

// AsNoTrackingWithIdentityResolution — for read-only queries with shared
// references in the result set; avoids duplicate object graphs.
```

Two rules that pay off forever:
1. **Always project to a DTO** at the query boundary. Returning entities leaks the schema, drags lazy-load surprises into the controller, and forces over-fetching of every column.
2. **Push every filter you can** above the `AsEnumerable()`/`ToList()` boundary. Every predicate that runs server-side is one fewer row crossing the wire.

**Senior-level gotchas:**
- A captured local in a predicate is a **closure** — `db.Orders.Where(o => o.CustomerId == id)` allocates a closure object on each call. Hot paths can hoist the lambda into a static `Expression<Func<...>>` cached at startup. EF Core 6+ also caches compiled queries by expression-tree shape, so this matters less than it used to.
- Returning `IQueryable<T>` from a repository leaks the persistence model into callers — they can compose more filters but also more bugs (N+1, accidental client eval). Returning `IAsyncEnumerable<T>` of DTOs is the senior compromise: streamable, materialized shape, no provider leak.
- `IEnumerable<T>` extension methods that already allocate a buffer (`ToList()`, `ToArray()`) followed by `.AsEnumerable()` is pure noise — the buffer is already in memory.
- Inside a transaction, two enumerations of the same `IQueryable` can return **different** rows because of concurrent writers. Snapshot with `ToListAsync` once.
- `Count()` on an `IQueryable<T>` issues `SELECT COUNT(*)` even if you've already materialized other queries from the same table — it's a separate round trip. For "is there any?" use `AnyAsync()`, which translates to `EXISTS` and short-circuits.
- `IEnumerable<T>` is covariant; `IList<T>` is invariant. Returning `IReadOnlyList<T>` is usually the right read-side type — covariant *and* gives the caller `Count` and indexing.
- A `ToList()` immediately followed by another `Where(...).ToList()` doubles the allocations for no benefit. Build the full chain, then materialize once.
- When you marshal a query across an `IAsyncEnumerable<T>` boundary, the producer's `CancellationToken` is **not** automatically threaded — use `WithCancellation(ct)` (extension method) at the consumer's `await foreach`.
- `AsQueryable()` over a `List<T>` does *not* let unit tests faithfully reproduce EF Core behavior. `EnumerableQuery<T>` happily evaluates anything; the real provider would have thrown. Test against a real provider (in-memory or sqlite) when correctness matters.
- `TryGetNonEnumeratedCount` (.NET 6+) returns `Count` if the source implements `ICollection<T>` / `IReadOnlyCollection<T>` without enumerating. Use it when you need a count for buffer sizing without forcing an iteration on a possibly-lazy source.
