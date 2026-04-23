# Array, List, Dictionary, HashSet, Stack, Queue

_Targets .NET 10 / C# 14. See also: [Value types](../TYPES/value-types.md), [Reference types](../TYPES/reference-types.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md), [Span and Memory](../MEMORY_MANAGEMENT/span-and-memory.md), [IEnumerable vs IQueryable](../LINQ/ienumerable-vs-iqueryable.md), [Concurrent collections](./concurrent-collections.md), [Readonly collections](./readonly-collections.md)._

The mutable, single-threaded BCL collections you reach for every day. Below is the senior view: how each is laid out in memory, where the perf cliffs are, and which API to use to avoid the obvious traps.

## The interface hierarchy

```
IEnumerable<T>
   └─ ICollection<T>           // Count, Add, Remove, Contains, Clear, IsReadOnly
        └─ IList<T>            // indexer, IndexOf, Insert, RemoveAt
        └─ ISet<T>             // UnionWith, IntersectWith, ...

IReadOnlyCollection<T>          // Count + IEnumerable<T>
   └─ IReadOnlyList<T>          // indexer
   └─ IReadOnlySet<T>           // (net5+) read-side set algebra

IDictionary<TKey,TValue> : ICollection<KeyValuePair<TKey,TValue>>
IReadOnlyDictionary<TKey,TValue> : IReadOnlyCollection<KeyValuePair<TKey,TValue>>
```

`IReadOnlyList<T>` is **covariant** in `T` (`out T`). `IList<T>` is invariant — that's the variance trap that drives a lot of API design. See [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md).

## Arrays — `T[]`

Fixed-length, contiguous, reference type. Cheapest possible random-access container.

```csharp
int[] a = new int[8];           // zero-initialized
int[] b = [1, 2, 3];            // collection expression, C# 12+
int[] c = Array.Empty<int>();   // singleton — use this, not `new int[0]`

// Span over an array — zero-copy, the workhorse for high-perf code.
Span<int> s = a.AsSpan(start: 1, length: 4);
```

**Covariance hazard.** Reference-type arrays are covariant by language design — and that's a runtime bug factory:

```csharp
object[] arr = new string[2];
arr[0] = 42;   // 💥 ArrayTypeMismatchException at runtime, not compile time
```

The runtime stamps each store. That check has a real cost on hot loops. Generic collections (`List<T>`) are invariant and avoid both the cost and the bug.

**Multi-dim vs jagged.** `int[,]` is one allocation but uses `ldelem.multi` (slower, bounds-checked per dim). `int[][]` is N+1 allocations but the JIT eliminates inner-loop bounds checks far better. For numerical work, jagged usually wins; for fixed small grids, multi-dim is fine.

## `List<T>` — the default sequence

Backed by a single `T[] _items`. `Capacity` is the array length, `Count` is the populated prefix. Adds beyond capacity allocate a **new array twice the size** and copy.

```csharp
var list = new List<int>(capacity: 1024);   // pre-size when you know
list.Add(1);

// Pre-grow without seeding contents:
list.EnsureCapacity(10_000);                // net6+

// Tighten after a bulk-load:
list.TrimExcess();
```

### Zero-copy span access

```csharp
Span<int> span = CollectionsMarshal.AsSpan(list);
for (int i = 0; i < span.Length; i++)
    span[i] *= 2;                           // mutates the backing array directly
```

`CollectionsMarshal.AsSpan` returns a span over the **live** `_items` — fastest possible iteration with no enumerator allocation. **Do not** mutate `Count` while holding the span: any `Add`/`Remove` may reallocate `_items` and your span becomes a dangling view.

`CollectionsMarshal.SetCount(list, n)` (.NET 8) lets you set `Count` directly and then fill via the span — useful for "allocate N and write into them" patterns. Anything you don't write yourself contains stale data.

### Iteration

`foreach (var x in list)` uses a **struct enumerator** (`List<T>.Enumerator`) — zero allocation. The same loop over `IEnumerable<T> e = list;` boxes the enumerator. Type the local as `List<T>` (or use `var`) on hot paths.

## `Dictionary<TKey,TValue>`

Open-addressing-with-chaining: `int[] _buckets` indexes into `Entry[] _entries`. Each entry stores `{hashCode, next, key, value}`. Buckets store `entry-index + 1` so `0` means empty. On collision, `next` walks a chain inside `_entries`.

```csharp
var d = new Dictionary<string, int>(capacity: 1024, StringComparer.Ordinal);

// TryAdd: one hash, no exception on duplicate.
d.TryAdd("a", 1);

// AVOID this pattern — two hashes:
if (!d.ContainsKey("a")) d.Add("a", 1);

// Mutate-in-place without re-hashing:
ref int slot = ref CollectionsMarshal.GetValueRefOrAddDefault(d, "a", out bool existed);
slot += 1;                                   // increments existing or starts at 1
```

`GetValueRefOrAddDefault` and `GetValueRefOrNullRef` are the senior-level APIs for counter / accumulator dictionaries — they touch the slot once. The naïve `d[k] = d.GetValueOrDefault(k) + 1` does two lookups.

### `GetHashCode` / `Equals` contract

The contract: equal objects must produce equal hash codes; the hash should be stable for the lifetime of the key inside the dictionary. Mutate a key after insertion and you've corrupted the dictionary — you'll never find the entry again.

For custom keys, prefer **records** (auto-generated value equality + hash) or pass an `IEqualityComparer<T>`. For string keys, **always specify** the comparer (`StringComparer.Ordinal`, `StringComparer.OrdinalIgnoreCase`) — the default is `EqualityComparer<string>.Default` (ordinal), which is fine, but being explicit prevents the next maintainer from assuming culture-aware compare.

### `AlternateLookup<TAlternate>` (.NET 9)

Look up by a type that hashes/compares equivalently to the key — typically `ReadOnlySpan<char>` over a `Dictionary<string, V>`:

```csharp
var d = new Dictionary<string, int>(StringComparer.Ordinal) { ["hi"] = 1 };
var alt = d.GetAlternateLookup<ReadOnlySpan<char>>();
ReadOnlySpan<char> span = "hi".AsSpan();
if (alt.TryGetValue(span, out var v)) { ... }   // zero string allocation
```

Same trick exists for `HashSet<T>` and `ConcurrentDictionary<K,V>`. Eliminates the "I have a slice of a buffer, must I allocate a `string` to look it up?" allocation that haunts parsers.

## `HashSet<T>`

Same hash plumbing as `Dictionary` but value-only. Set algebra is in-place:

```csharp
var a = new HashSet<int> { 1, 2, 3 };
var b = new HashSet<int> { 2, 3, 4 };
a.UnionWith(b);          // a = {1,2,3,4}
a.IntersectWith(b);      // a = {2,3,4}
a.ExceptWith(b);         // a = {}
a.SymmetricExceptWith(b);
```

For read-only set checks against a fixed catalog, prefer **`FrozenSet<T>`** (see [Frozen collections](./frozen-collections.md)) — same API, faster `Contains`.

## `Stack<T>` / `Queue<T>` / `PriorityQueue<TElement,TPriority>`

`Stack<T>` and `Queue<T>` are array-backed (Queue is a circular buffer with `_head`, `_tail`). Both grow by doubling.

```csharp
var s = new Stack<int>();        s.Push(1); s.Pop(); s.TryPop(out var x);
var q = new Queue<int>();        q.Enqueue(1); q.Dequeue(); q.TryDequeue(out var y);
```

`PriorityQueue<TElement,TPriority>` (.NET 6+) is a binary min-heap. `Enqueue` / `Dequeue` are O(log n); `Peek` is O(1).

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("low", 10);
pq.Enqueue("hi",   1);
pq.Dequeue();   // "hi" — lowest priority value first
```

**Two non-obvious traps:**
- It is **not stable** — equal priorities pop in unspecified order.
- It compares only on `TPriority` (`IComparer<TPriority>`), not on `TElement`. To break ties, fold a sequence number into the priority (`(priority, seq)` tuple with a tuple comparer).

## `LinkedList<T>` and the `Sorted*` family

`LinkedList<T>` is doubly-linked with one heap allocation per node. Cache-hostile, never the right answer for "I want fast inserts in the middle" — `List<T>.Insert` shifts memcpy-fast and the linked list almost always loses on real benchmarks. Use it only when you legitimately need stable `LinkedListNode<T>` references for O(1) splice.

`SortedDictionary<K,V>` is a red-black tree (O(log n) everything). `SortedList<K,V>` is two parallel sorted arrays — O(log n) lookup, O(n) insert. Read-heavy → `SortedList`; write-heavy → `SortedDictionary`. For most "I want them sorted" needs, you really wanted `OrderBy(...)` at output time.

## Big-O cheatsheet

| Op | `List<T>` | `Dictionary<K,V>` | `HashSet<T>` | `Stack<T>` / `Queue<T>` | `LinkedList<T>` | `SortedDictionary<K,V>` |
|---|---|---|---|---|---|---|
| Add / Push / Enqueue | O(1)¹ | O(1)¹ | O(1)¹ | O(1)¹ | O(1) | O(log n) |
| Lookup by key/index | O(1) | O(1) | O(1) | O(1) peek | O(n) | O(log n) |
| Contains by value | O(n) | O(1) | O(1) | O(n) | O(n) | O(log n) |
| Insert at i | O(n) | n/a | n/a | n/a | O(1) given node | O(log n) |
| Remove by value | O(n) | O(1) | O(1) | n/a | O(n) | O(log n) |
| Iteration | O(n) | O(n) | O(n) | O(n) | O(n) | O(n) sorted |

¹ Amortized — periodic resize is O(n).

## Collection expressions (C# 12)

```csharp
int[] a = [1, 2, 3];
List<int> b = [1, 2, 3];
HashSet<int> c = [1, 2, 3];
ImmutableArray<int> d = [1, 2, 3];

int[] joined = [..a, 99, ..b];           // spread operator
ReadOnlySpan<int> s = [1, 2, 3];         // stack-allocated for spans
```

The compiler picks the cheapest construction strategy per target type. For `ReadOnlySpan<T>` of a const-prim type, it lowers to a static blob — zero allocation, zero copy. Use these freely; they're not just sugar.

**Senior-level gotchas:**
- `Dictionary<K,V>` is **not safe** for any concurrent access, even read-only when another thread is writing. The reader can see a torn entry and infinite-loop the chain walk. Reach for `ConcurrentDictionary` or `FrozenDictionary` (read-only by construction).
- `Capacity` doubling means a `List<T>` that ends with 1024 items has a backing array of length 1024 *or* up to 2048 — `TrimExcess` is the only way to release the slack. Worth doing for long-lived caches.
- `List<T>.RemoveAt(0)` is O(n) (memcpy). For FIFO use `Queue<T>`. For "remove all matching", use `RemoveAll(predicate)` — it does a single in-place compaction in O(n) instead of N×O(n).
- `Dictionary.Remove` followed by `Add` of the same key allocates a new entry; the freed slot is **not** preferentially reused. For high-churn keys, mutate via `CollectionsMarshal.GetValueRefOrAddDefault` instead.
- A struct held in a `List<T>` indexer (`list[i].Field = x`) is a **copy** — the assignment compiles but does nothing. Use `ref var slot = ref CollectionsMarshal.AsSpan(list)[i]; slot.Field = x;`.
- `foreach` over `Dictionary<K,V>` uses an iteration order that is **not** insertion order in current runtimes. The CLR has changed this between major versions — never depend on it.
- `default(ArraySegment<T>)`, `default(Memory<T>)`, etc. are valid empty values; `default(KeyValuePair<K,V>)` has `default(K)` and `default(V)` — don't use them as sentinels in a dict whose values can legitimately be `default`.
- `List<T>.AsReadOnly()` returns a `ReadOnlyCollection<T>` **view** — the underlying list is still mutable through the original reference. See [Readonly collections](./readonly-collections.md). For a true snapshot, `ToImmutableArray()` or `ToFrozenSet()`.
- `PriorityQueue<TElement,TPriority>` cannot be re-prioritized in place. Removing-then-reinserting is the workaround; if you need decrease-key, you want a different data structure (an indexed heap library).
- `HashSet<T>.TrimExcess()` does **not** rebuild buckets to the new size on .NET versions before 8 — set's hash distribution stayed wide. On .NET 8+ it does. Worth checking your TFM if you rely on the rebuild.
