# Readonly collections

_Targets .NET 10 / C# 14. See also: [Array, List, Dictionary, HashSet, Stack, Queue](./array-list-dictionary-hashset-stack-queue.md), [Immutable collections](./immutable-collections.md), [Frozen collections](./frozen-collections.md), [Span and Memory](../MEMORY_MANAGEMENT/span-and-memory.md)._

"Readonly" in the BCL means **the consumer of this reference cannot mutate it through this reference**. It does **not** mean immutable. The original holder can mutate freely; even worse, a clever consumer can sometimes downcast back to the mutable type. Understand the gap between read-only and immutable before designing your APIs.

## The interface family

```
IReadOnlyCollection<T>     : IEnumerable<T>          // Count
   └─ IReadOnlyList<T>                               // [int index] (out T variance)
   └─ IReadOnlySet<T>      (net5+)                   // IsSubsetOf, Overlaps, Contains, ...

IReadOnlyDictionary<TKey,TValue>                      // [TKey], TryGetValue, Keys, Values
```

These are **interfaces**, so any type that implements them works (`List<T>`, `T[]`, `ImmutableArray<T>`, `FrozenDictionary<K,V>` ...). They expose a read-only API surface. Whether the *underlying object* mutates depends on the concrete type.

`IReadOnlyList<T>` is **covariant in T** (`out T`). `IReadOnlyCollection<T>` is too. That makes them safer to expose from public APIs than `IList<T>` (invariant) — and is why `IReadOnlyList<string>` can be assigned to a variable of type `IReadOnlyList<object>`.

## Concrete wrappers — `ReadOnlyCollection<T>` / `ReadOnlyDictionary<K,V>`

```csharp
List<int> data = [1, 2, 3];
ReadOnlyCollection<int> view = data.AsReadOnly();
// view[0] is 1; view.Add(...) doesn't compile.

data.Add(99);
Console.WriteLine(view.Count);   // 4 — view sees the new item
```

`AsReadOnly()` returns a thin wrapper. **It is a view, not a snapshot.** Any mutation through the original `data` reference is visible through `view`. If you publish `view` to other threads while another thread is mutating `data`, you have a data race.

.NET 9 added `Dictionary<K,V>.AsReadOnly()` as the equivalent for dictionaries (previously you had to construct `new ReadOnlyDictionary<K,V>(dict)` by hand).

```csharp
var d = new Dictionary<string, int> { ["a"] = 1 };
IReadOnlyDictionary<string, int> ro = d.AsReadOnly();
```

## The downcast escape hatch

This is the most common "readonly is not safe" surprise:

```csharp
public sealed class Catalog
{
    private readonly List<Item> _items = new();
    public IReadOnlyList<Item> Items => _items;   // looks safe…
}

// Caller:
var catalog = new Catalog();
var list = (List<Item>)catalog.Items;            // … but the runtime type leaks
list.Add(new Item());                             // mutated through the back door
```

`IReadOnlyList<Item>` only restricts the API surface. The runtime type is `List<Item>`, and any consumer can downcast. To prevent this:

1. Wrap before exposing: `_items.AsReadOnly()` returns a `ReadOnlyCollection<Item>`. The downcast still succeeds (`(ReadOnlyCollection<Item>)`) but `ReadOnlyCollection<T>` has no public mutation API. The underlying list is still mutable through the *original* field reference.
2. Snapshot before exposing: `_items.ToImmutableArray()` or `_items.ToArray()`. Caller sees a copy; their mutation can't reach your state.
3. Hold the field as `ImmutableArray<T>` from the start — there is no mutable form to downcast to.

## "Read-only view" vs "Immutable" vs "Frozen" — pick the right one

| You want… | Type |
|---|---|
| Caller-side read-only API, you keep mutating internally | `IReadOnlyList<T>` over a `List<T>` (with the downcast caveat) |
| A snapshot the caller cannot affect, you may rebuild on your side | Return `ImmutableArray<T>` |
| The caller and you both read often, no one mutates | Return `FrozenDictionary<K,V>` / `FrozenSet<T>` |
| Pure compile-time signal that this method does not mutate | Use `IReadOnly*` interface in the parameter type |
| Span-style read-only buffer view (no allocation) | `ReadOnlySpan<T>` / `ReadOnlyMemory<T>` |

If you find yourself defensively copying inside every method that takes an `IReadOnlyList<T>` because you don't trust the caller — change the parameter to `ImmutableArray<T>` and stop copying.

## `ReadOnlySpan<T>` / `ReadOnlyMemory<T>` — different beasts

These are **not** part of the `IReadOnly*` family. They're allocation-free **windows** into existing memory.

```csharp
ReadOnlySpan<char> head = "hello world".AsSpan(0, 5);
ReadOnlyMemory<byte> body = buffer.AsMemory(start, length);
```

`ReadOnlySpan<T>` is a `ref struct` — stack-only, can't be a field on a heap object, can't cross an `await`. `ReadOnlyMemory<T>` is heap-friendly and async-friendly. Both expose only read methods on the underlying buffer.

These are the right tool for parser/serializer code, not for "I want to expose a read-only collection from a service" — that's `IReadOnlyList<T>` / `ImmutableArray<T>` territory. See [Span and Memory](../MEMORY_MANAGEMENT/span-and-memory.md) for the full treatment.

## Defensive copy at API boundaries

When the readonness must be guaranteed regardless of caller behavior, copy at the boundary:

```csharp
public IReadOnlyList<Item> GetItems() => _items.ToArray();              // O(n), allocates
public ImmutableArray<Item> GetItems() => _items.ToImmutableArray();    // O(n), value-typed
public FrozenSet<int> GetAllowedCodes() => _allowed.ToFrozenSet();      // O(n) build, fast reads
```

The cost is one allocation per call. For low-throughput public APIs, the safety is worth it. For per-request hot paths, hold the immutable/frozen form internally and return it directly.

## The `params` and collection-expression interaction

```csharp
public void LogAll(params ReadOnlySpan<string> messages)   // C# 13+
{
    foreach (var m in messages) Log(m);
}

LogAll("a", "b", "c");                                     // zero allocation
```

`params ReadOnlySpan<T>` (C# 13) replaces `params T[]` for variadic methods that don't escape the array. The compiler may stack-allocate the backing storage. Prefer this for hot-path logging / formatting helpers.

## Comparison: every "read-only-ish" collection at a glance

| Type | Safe to share across threads | Caller can mutate via downcast | Caller can observe my later mutations | Allocation |
|---|---|---|---|---|
| `IReadOnlyList<T>` over `List<T>` | No | Yes | Yes | None |
| `ReadOnlyCollection<T>` (`.AsReadOnly()`) | Only if you stop mutating original | No (no mutation API) | Yes | Wrapper allocation |
| `ImmutableArray<T>` | Yes | No | No (snapshot) | Snapshot allocation |
| `FrozenSet<T>` / `FrozenDictionary<K,V>` | Yes | No | No (snapshot) | Build cost |
| `ReadOnlySpan<T>` over an array | No (stack-only) | No (no setter) | Yes if buffer mutates | None |

**Senior-level gotchas:**
- **`IReadOnlyList<T>` is not immutable.** It is "read-only via this reference." The classic bug: returning `IReadOnlyList<T>` from a property whose backing field is a `List<T>` and being surprised when callers mutate it through a downcast.
- `ReadOnlyCollection<T>` from `AsReadOnly()` is a **view**. If your goal is "give the caller a thing they can iterate without seeing my future updates," you want `ToImmutableArray()` — not a wrapper.
- `IReadOnlyDictionary<K,V>` does **not** implement `IReadOnlyCollection<KeyValuePair<K,V>>` in older runtimes; it does in current ones. If you target a wide TFM range, don't rely on the inheritance.
- `ICollection<T>.IsReadOnly` exists for the legacy `ICollection<T>` interface — `true` means writes throw. Most `IList<T>` implementations return `false`. It's a dynamic check, not a type-level guarantee.
- `IReadOnlyList<T>` and `IList<T>` are unrelated in the interface hierarchy. A type can implement one without the other (and `Array` historically only implemented `IList<T>`, not `IReadOnlyList<T>`, until later runtimes added it).
- Records with `IReadOnlyList<T>` properties get reference equality on that property (records call `EqualityComparer<T>.Default.Equals`, which is reference equality for `IReadOnlyList<T>`). For value equality of the contents, override `Equals` and `GetHashCode` or use `ImmutableArray<T>` plus `SequenceEqual`.
- `ReadOnlySpan<T>` cannot appear in async methods, lambdas that capture it, or fields. The compiler error is clear ("ref struct cannot be used here") but it bites when you try to refactor allocating code into async helpers.
- `params ReadOnlySpan<T>` (C# 13) is overload-preferred over `params T[]` when both exist. Adding the span overload may silently change which method dispatch picks for callers — verify with the SharpLab or with explicit type annotations.
- For multi-threaded scenarios where you "publish" a read-only collection by assigning it to a `volatile` field, ensure the field type is the **interface** the readers use (`IReadOnlyList<T>` or `ImmutableArray<T>`). Mixing `volatile` with a value-typed `ImmutableArray<T>` field is fine because it's a single-reference struct (8 bytes) — `Volatile.Read` / `Volatile.Write` cover it.
- `Dictionary<K,V>.AsReadOnly()` is **.NET 9+**. On older TFMs, do `new ReadOnlyDictionary<K,V>(dict)` explicitly.
