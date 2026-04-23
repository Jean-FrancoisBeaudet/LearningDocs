# Immutable collections

_Targets .NET 10 / C# 14. See also: [Frozen collections](./frozen-collections.md), [Readonly collections](./readonly-collections.md), [Concurrent collections](./concurrent-collections.md), [Array, List, Dictionary, HashSet, Stack, Queue](./array-list-dictionary-hashset-stack-queue.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md)._

`System.Collections.Immutable` (in the BCL since .NET Core) gives you collections whose every "mutation" returns a **new** instance — the existing one is observably untouched. They use *persistent* data structures (structural sharing) so the new instance reuses most of the old one's memory.

Immutable collections trade speed for safety. They are excellent for shared state, configuration, snapshots, and lock-free updates. They are the wrong choice when you have a hot loop building a buffer.

## The lineup

| Type | Internal representation | Mutation cost | Lookup cost |
|---|---|---|---|
| `ImmutableArray<T>` | `readonly struct` wrapping `T[]` | O(n) — copies whole array | O(1) random |
| `ImmutableList<T>` | AVL tree | O(log n) | O(log n) |
| `ImmutableDictionary<K,V>` | Hash array mapped trie (HAMT-like) | O(log n) | O(log n) |
| `ImmutableHashSet<T>` | HAMT-like | O(log n) | O(log n) |
| `ImmutableSortedDictionary<K,V>` | AVL tree by key | O(log n) | O(log n) |
| `ImmutableSortedSet<T>` | AVL tree | O(log n) | O(log n) |
| `ImmutableQueue<T>` | Two immutable stacks | O(1) amortized | O(1) head |
| `ImmutableStack<T>` | Cons list (immutable linked list) | O(1) push/pop | O(1) head |

## `ImmutableArray<T>` — the one you'll use most

A `readonly struct` wrapping a `T[]`. Cheap to pass around (it's a single reference inside a struct). Random-access is `arr[i]` — a normal array dereference.

```csharp
ImmutableArray<int> a = [1, 2, 3];
ImmutableArray<int> b = a.Add(4);          // allocates a new int[4], copies a, sets [3]=4
                                            // a is unchanged
```

Every single-element mutation **copies the whole array**. That's the cost of "ImmutableArray." For a tight loop of 10k Adds, you've allocated 10k arrays totaling 50M ints. Use the **builder** instead:

```csharp
var bld = ImmutableArray.CreateBuilder<int>(initialCapacity: 10_000);
for (int i = 0; i < 10_000; i++) bld.Add(i);
ImmutableArray<int> result = bld.MoveToImmutable();   // O(1) — hands over the buffer
```

`MoveToImmutable` requires `Count == Capacity` (no slack). Use `bld.ToImmutable()` to copy out an exactly-sized array if you don't want to size precisely.

### The `default` trap

```csharp
ImmutableArray<int> arr = default;
int n = arr.Length;     // 💥 NullReferenceException
bool empty = arr.IsDefault;     // true — the inner T[] is null
bool emptyOrZeroLen = arr.IsDefaultOrEmpty;   // true if either
```

Because `ImmutableArray<T>` is a struct, `default(ImmutableArray<T>)` is reachable — it's an instance with a null inner array. Accessing `Length`, the indexer, or any LINQ over it throws. **Always** initialize fields to `ImmutableArray<T>.Empty`:

```csharp
public ImmutableArray<string> Tags { get; init; } = ImmutableArray<string>.Empty;
```

This is the #1 bug in records that hold `ImmutableArray` properties.

## `ImmutableList<T>` — when frequent edits matter

AVL-balanced binary tree. Every "mutation" returns a new tree that shares O(n - log n) of the old tree's nodes. `Add`, `Insert`, `Remove`, indexer all O(log n).

```csharp
ImmutableList<int> list = ImmutableList<int>.Empty;
ImmutableList<int> list2 = list.Add(1).Add(2).Insert(0, 99);   // 3 allocations of log-tall tree slices
```

Random access is O(log n), not O(1). For read-mostly with random access, prefer `ImmutableArray<T>`. For frequent middle-inserts on a long sequence, `ImmutableList<T>` wins.

## `ImmutableDictionary<K,V>` and `ImmutableHashSet<T>`

HAMT-like (hash array mapped trie). Lookup is O(log n) but with a small base (32-way fan-out → log32). Insertion creates a new path of nodes from root to bucket — O(log n) allocations and copies.

```csharp
var d = ImmutableDictionary<string, int>.Empty
    .WithComparers(StringComparer.OrdinalIgnoreCase)
    .Add("a", 1)
    .Add("b", 2);

var d2 = d.SetItem("a", 99);   // d still sees "a" => 1
```

For pure lookup workloads, **prefer `FrozenDictionary<K,V>`** — same read-only semantics, much faster lookups. `ImmutableDictionary` earns its keep when you need lock-free, copy-on-write updates (see `ImmutableInterlocked` below).

## `ImmutableInterlocked` — lock-free state updates

Pair an immutable collection with `ImmutableInterlocked` to get atomic updates over a single field — no `lock`, no `ConcurrentDictionary` overhead.

```csharp
private ImmutableDictionary<string, Subscriber> _subs =
    ImmutableDictionary<string, Subscriber>.Empty;

public void Subscribe(string topic, Subscriber sub) =>
    ImmutableInterlocked.AddOrUpdate(
        ref _subs,
        topic,
        addValueFactory:    _      => sub,
        updateValueFactory: (_, _) => sub);

public void Unsubscribe(string topic) =>
    ImmutableInterlocked.TryRemove(ref _subs, topic, out _);
```

Internally: a `CompareExchange` loop. The factories may run multiple times under contention — make them pure. Reads (`_subs[key]`) are pure pointer dereferences with **no synchronization at all** — that's the win versus `ConcurrentDictionary` for very hot read paths. See [Concurrent collections](./concurrent-collections.md) for the broader comparison.

## Builders — the escape hatch

Every immutable collection has a `Builder` you can use to do many mutations efficiently, then freeze:

```csharp
var bld = ImmutableDictionary.CreateBuilder<string, int>(StringComparer.Ordinal);
foreach (var (k, v) in source) bld[k] = v;
ImmutableDictionary<string, int> snapshot = bld.ToImmutable();
```

Builders are mutable, single-threaded objects internally backed by the same trie shape. `ToImmutable()` is cheap: it freezes the trie nodes in place. Don't keep using the builder after `ToImmutable()` if you want repeatable behavior — its later mutations may or may not affect the returned snapshot depending on the type. Treat it as one-shot: build → freeze → discard.

## `ImmutableArray<T>` and records

```csharp
public sealed record Order(string Id, ImmutableArray<LineItem> Items);
```

Looks great until you compare two orders by `==`: **`ImmutableArray<T>.Equals` is reference equality of the inner `T[]`**, not element-wise equality. Two orders with structurally equal items are not `Equal` unless they share the array.

If you need element-wise value equality on the record, override:

```csharp
public sealed record Order(string Id, ImmutableArray<LineItem> Items)
{
    public bool Equals(Order? other) =>
        other is not null && Id == other.Id && Items.SequenceEqual(other.Items);

    public override int GetHashCode() =>
        HashCode.Combine(Id, Items.Length); // good enough; full hash is O(n)
}
```

This is a real footgun in DDD code that uses records as value objects.

## `ImmutableArray<T>` vs `ImmutableList<T>` — the choice

| Question | If yes → |
|---|---|
| Mostly read, occasional rebuild from scratch? | `ImmutableArray<T>` |
| Need O(1) indexer? | `ImmutableArray<T>` |
| Many incremental edits to a long sequence? | `ImmutableList<T>` |
| Care about struct-style cheap pass-by-value? | `ImmutableArray<T>` |
| The collection is small and read-mostly? | `ImmutableArray<T>` |

`ImmutableArray<T>` is the default. Pick `ImmutableList<T>` only when you've proven the per-mutation O(n) cost matters.

**Senior-level gotchas:**
- `default(ImmutableArray<T>)` throws on every member access. Always initialize fields to `.Empty`. Roslyn's CA1825 / IDE0301 catch some cases.
- `ImmutableArray<T>.Equals` is **reference equality on the inner array**. Records that hold one and depend on value equality are subtly broken — add `SequenceEqual` overrides.
- Calling `.Add()` in a loop on an immutable collection is O(n²) overall. Use a `Builder` whenever you'd write a loop.
- `ImmutableInterlocked` factories may run multiple times. Side-effecting factories (logging, allocations) are wrong here for the same reason as `ConcurrentDictionary.GetOrAdd`.
- "Immutable" is not "frozen" — the **values** stored are still freely mutable. `ImmutableArray<List<int>>` keeps the array immutable; the lists inside are not.
- For pure-read scenarios, `FrozenDictionary` beats `ImmutableDictionary` substantially on lookup speed. Use Immutable only when you need cheap incremental mutation.
- `WithComparer` on `ImmutableDictionary<K,V>` rebuilds the entire trie. Set the comparer once at construction (`ImmutableDictionary<K,V>.Empty.WithComparers(cmp)`); don't toggle it later.
- `ImmutableArray.Create(items)` (params) allocates an intermediate `T[]` for the `params` then copies it. For a known-size build, `ImmutableArray<T>.Create(item1, item2, ...)` overloads up to 4 args avoid the allocation, or use `ImmutableArray.CreateRange(span)` to wrap a span directly.
- `ImmutableArray<T>` boxing: when you assign one to an `IList<T>` or `IReadOnlyList<T>` interface, the struct is boxed. The mutation methods on `IList<T>` then throw `NotSupportedException`. Prefer to keep the type concrete in fields and signatures.
- Calling `.ToImmutableArray()` on something that already is an `ImmutableArray<T>` (after a couple of LINQ ops) re-allocates. If you control the pipeline, build into an `ImmutableArray<T>.Builder` directly.
