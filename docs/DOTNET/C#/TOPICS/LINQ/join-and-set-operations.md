# Join and Set operations

_Targets .NET 10 / C# 14. See also: [Filtering, Projections, Sorting](./filtering-projections-sorting.md), [IEnumerable vs IQueryable](./ienumerable-vs-iqueryable.md), [Pagination and Grouping](./pagination-and-grouping.md), [Readonly collections](../COLLECTIONS/readonly-collections.md), [Frozen collections](../COLLECTIONS/frozen-collections.md)._

LINQ's relational operators. They look like SQL but the semantics differ in subtle ways — equi-join only, deferred-but-buffering, and the `*By` variants change which side becomes the lookup. This is also where EF Core's nav-property model competes with `Join` and usually wins.

## `Join` — inner equi-join

```csharp
public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>(
    this IEnumerable<TOuter>  outer,
    IEnumerable<TInner>       inner,
    Func<TOuter, TKey>        outerKeySelector,
    Func<TInner, TKey>        innerKeySelector,
    Func<TOuter, TInner, TResult> resultSelector,
    IEqualityComparer<TKey>?  comparer = null);
```

```csharp
var rows = orders.Join(
    customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { o.Id, CustomerName = c.Name, o.Total });
```

LINQ's `Join` is **equi-join only** — the keys must match by equality (uses `IEqualityComparer<TKey>`). There is no `On o.X > c.X` operator. Inequality joins must be expressed as `SelectMany` + `Where`, which translates to a CROSS JOIN + filter in SQL — orders-of-magnitude slower than an indexed range scan. If you need it, push the inequality into raw SQL or a stored procedure.

### Implementation note

LINQ-to-Objects' `Join` builds a `Lookup<TKey, TInner>` from the `inner` source, then streams the `outer` and probes. Memory cost is `O(|inner|)`; that's per-call, no caching. If you join the same inner repeatedly, hoist a `ToLookup` and use it directly:

```csharp
var byCustomer = customers.ToLookup(c => c.Id);
foreach (var o in orders)
    foreach (var c in byCustomer[o.CustomerId])
        yield return (o, c);
```

### Composite keys

```csharp
var rows = a.Join(
    b,
    x => new { x.TenantId, x.Code },          // anonymous type — value equality
    y => new { y.TenantId, y.Code },
    (x, y) => (x, y));

// Tuple equivalent — same semantics.
var rows2 = a.Join(b,
    x => (x.TenantId, x.Code),
    y => (y.TenantId, y.Code),
    (x, y) => (x, y));
```

Both work in LINQ-to-Objects. EF Core 7+ translates both. Anonymous types are slightly more reliable in older EF Core versions.

## `GroupJoin` + `SelectMany` + `DefaultIfEmpty` — left outer join

LINQ has no `LeftJoin` operator; the canonical pattern composes three:

```csharp
var rows = customers
    .GroupJoin(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, os) => new { Customer = c, Orders = os })
    .SelectMany(
        x => x.Orders.DefaultIfEmpty(),
        (x, o) => new
        {
            x.Customer.Name,
            OrderId = o?.Id,
            Total   = o?.Total ?? 0m
        });
```

`DefaultIfEmpty()` injects a single `default(T)` element if the inner sequence is empty — that's how the "outer" side gets to appear with `null` partners. For value-type inners, the `default` is the zeroed struct, not null, so check via `Any()` upstream when that distinction matters.

In EF Core, the BCL nav-property pattern is almost always cleaner than manual `GroupJoin`:

```csharp
var rows = await db.Customers
    .SelectMany(
        c => c.Orders.DefaultIfEmpty(),
        (c, o) => new { c.Name, OrderId = (int?)o.Id, Total = (decimal?)o.Total ?? 0m })
    .ToListAsync(ct);
```

EF Core's optimizer recognizes `nav.DefaultIfEmpty()` as a left join. Manual `Join`/`GroupJoin` disables nav-property tracking and shadow-property optimizations — prefer nav-property navigation in EF Core 5+.

## `Zip` — pairwise

```csharp
// Two-arity, with selector.
var pairs = a.Zip(b, (x, y) => (x, y));

// Two-arity, tuple overload (.NET 6+).
IEnumerable<(int, string)> tuples = a.Zip(b);

// Three-arity (.NET 6+).
var triples = a.Zip(b, c, (x, y, z) => (x, y, z));
```

Stops at the shorter sequence — extra elements in the longer source are silently dropped. If "one side ran out" is significant, check `Count()` first, or pad the shorter side with `Concat(Enumerable.Repeat(default, n))`.

`Zip` is **not** a join — it pairs by **position**, not by key. Pairing by position over an unordered source is meaningless; only use it when both sources have an intrinsic order (e.g., parallel arrays, columnar data).

## Set operations

```csharp
var union   = a.Union(b);                                      // distinct(a ∪ b)
var inter   = a.Intersect(b);                                  // distinct(a ∩ b)
var diff    = a.Except(b);                                     // distinct(a − b)
var concat  = a.Concat(b);                                     // a then b, duplicates kept
var symdiff = a.Union(b).Except(a.Intersect(b));               // (no built-in symmetric diff)
```

| Operator | Distinct? | Order preserved? | Set semantics? |
|---|---|---|---|
| `Concat` | no | yes | no — pure sequence concatenation |
| `Union` | yes | source-first encounter order | yes |
| `Intersect` | yes | order of `a` | yes |
| `Except` | yes | order of `a` | yes |

`Union`/`Intersect`/`Except` build a `HashSet<T>` from the right side internally; that's `O(|right|)` memory per call. If the right side is large, hoist it to a `HashSet<T>` and use `Where(x => !set.Contains(x))` for `Except`-like semantics — same complexity, more explicit.

For repeated set-membership queries against a fixed catalog, `FrozenSet<T>` (`ToFrozenSet()`) is the senior choice — same `Contains` semantics, a build-time-optimized hash table that's typically 30–50% faster than `HashSet<T>` on lookup. See [Frozen collections](../COLLECTIONS/frozen-collections.md).

### Comparer plumbing

```csharp
var distinctNames = users.Select(u => u.Name)
                         .Distinct(StringComparer.OrdinalIgnoreCase);

var common = listA.Intersect(listB, StringComparer.OrdinalIgnoreCase);
```

The default comparer is `EqualityComparer<T>.Default`. For records, that's the auto-generated value equality. For your own classes, the default is reference equality unless you override `Equals`/`GetHashCode` — set ops "won't deduplicate" is almost always missing equality. Pass an `IEqualityComparer<T>` instead of overriding `Equals` if the project's domain semantics differ across call sites.

## `*By` variants (.NET 6+)

```csharp
var distinctByEmail = users.DistinctBy(u => u.Email);

var unionByEmail  = a.UnionBy(b, x => x.Email);                            // distinct on key
var interByEmail  = a.IntersectBy(b.Select(x => x.Email), x => x.Email);
var exceptByEmail = a.ExceptBy(b.Select(x => x.Email), x => x.Email);
```

These project a key, do the set operation by that key, and return source elements (not keys). The "first wins" element in `UnionBy`/`DistinctBy` is from `a`. EF Core does not translate the `*By` variants on `IQueryable<T>` (as of EF Core 8) — express in raw SQL or restructure as `GroupBy(...).Select(g => g.First())`.

## EF Core translation of set operators

```csharp
var combined = await q1.Union(q2).ToListAsync(ct);     // SQL UNION
var both     = await q1.Intersect(q2).ToListAsync(ct); // SQL INTERSECT
var only1    = await q1.Except(q2).ToListAsync(ct);    // SQL EXCEPT
var append   = await q1.Concat(q2).ToListAsync(ct);    // SQL UNION ALL
```

Both sides must project the **same shape** (same number of columns, same nullable types). EF Core enforces this at translation time; mismatches throw with a "different projection" message. Common fix: explicit `Select(x => new Dto(...))` on both sides so the projection is identical.

## EF Core: prefer navigation properties

```csharp
// Manual join — works, but loses nav-property benefits and the optimizer's join pruning.
var rows = await db.Orders.Join(
    db.Customers, o => o.CustomerId, c => c.Id,
    (o, c) => new { o.Id, c.Name }).ToListAsync(ct);

// Idiomatic EF Core — projection through the nav.
var rows2 = await db.Orders
    .Select(o => new { o.Id, o.Customer.Name })
    .ToListAsync(ct);
```

EF Core emits the equivalent SQL JOIN automatically and may prune unused tables or fold the join out entirely if the projection doesn't need it. Manual `Join` blocks those optimizations. The rule of thumb: use nav-property projections for "I have a foreign key and want to reach across"; reach for `Join` only when there is no nav (e.g., joining ad-hoc tables in the same context, or composing across two different `DbContext` instances after materialization).

**Senior-level gotchas:**
- LINQ's `Join` is **equi-only**. `On a.X > b.X` requires `SelectMany` + `Where` — and in EF Core that's a CROSS JOIN + filter. For range joins on indexed columns, drop to raw SQL or `FromSql` to keep the index seek.
- `Zip` silently truncates to the shorter sequence. If "one side missing" is a bug, check lengths upstream.
- `Concat` keeps duplicates and order — if you're using it as `Union`, you have a bug. If you wanted `Union`, you allocated a `HashSet<T>` you didn't think about.
- `Union`/`Intersect`/`Except` deduplicate using `EqualityComparer<T>.Default`. For classes without overridden equality, the result is reference-equality deduplication — usually not what you want. Pass a comparer or use a record.
- `Intersect` builds the hash set from the **second** argument; if one side is much larger, prefer `Where(x => smallSet.Contains(x))` to control which side hashes.
- `GroupJoin(...).SelectMany(...)` looks heavy but the BCL recognizes the pattern and (in .NET 8+) uses a single internal iterator. Don't rewrite it as nested `foreach` for "performance" — the LINQ form is fine.
- `DefaultIfEmpty()` over a `Nullable<T>` source still produces `null` elements; on a value-type source it produces `default(T)`, which can be a valid value (`0`, `0m`, `DateTime.MinValue`). Project to `T?` first if you need to distinguish "no row" from "row with default value".
- The `*By` variants (.NET 6+) do not translate in EF Core. The translation gap will close eventually; check the EF Core release notes for your TFM.
- `UnionBy`/`DistinctBy` keep the **first** element per key. If you wanted the **last**, reverse the source first or restructure as a group with `Last()`.
- `Join`'s key comparer must hash the same way on both sides — passing different comparers per side is impossible (`Join` takes one `IEqualityComparer<TKey>`). Asymmetric matching needs a custom key projection on each side that normalizes (e.g., `key => key.ToUpperInvariant()`).
- Set operators on EF Core require both sides to project the same shape and types. A nullable-vs-non-nullable mismatch is the most common source of "the projection differs" errors at translation.
- `ToHashSet(comparer)` (.NET 472/Core 2.0+) is the easy way to materialize a deduplicated set with custom equality, then drive the loop yourself instead of chaining set ops.
