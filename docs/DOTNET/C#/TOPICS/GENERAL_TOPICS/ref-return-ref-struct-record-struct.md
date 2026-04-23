# `ref` returns, `ref struct`, `record struct`

_Targets .NET 10 / C# 14. See also: [Span / Memory](../MEMORY_MANAGEMENT/span-memory.md), [Value types](../TYPES/value-types.md), [Generic constraints and variance](./generic-constraints-and-variance.md), [OOP / records](../OOP/classes-records-structs.md)._

Three distinct features, all aimed at **avoiding allocations and copies** while preserving safety: `ref` returns let callers refer to storage they don't own; `ref struct` constrains a type to the stack so it can hold borrowed references; `record struct` gives value semantics with compiler-generated equality. Together they're the backbone of high-performance C#.

## `ref` returns and `ref` locals

A `ref` return exposes a **managed pointer** to storage owned by something else — a field, an array element, an unboxed value.

```csharp
public sealed class Bag<T>
{
    private T[] _items = new T[16];
    private int _count;

    public ref T this[int i] => ref _items[i];     // ref return via indexer

    public ref T Add(T value)
    {
        if (_count == _items.Length) Array.Resize(ref _items, _count * 2);
        _items[_count] = value;
        return ref _items[_count++];
    }
}

var bag = new Bag<Point>();
ref var slot = ref bag.Add(default);
slot.X = 42;                                         // mutates in place, no copy
```

### `ref readonly`

Use when you want to expose a reference **without** letting callers mutate through it. Common for big `readonly struct`s you'd rather not copy:

```csharp
public ref readonly Matrix4x4 View => ref _view;
```

Assigning to a non-`ref readonly` local copies; using `ref readonly var` preserves the reference.

### Scope / escape rules

The compiler enforces that a `ref` never outlives what it points to. Rules (simplified):

- A `ref` to a **local** cannot be returned (the local dies with the frame).
- A `ref` to a **field of a struct parameter** cannot be returned (the struct may be in the caller's frame).
- A `ref` to an element of an array/`Span<T>` the method received as a parameter **can** be returned.

C# 11 introduced `scoped` to **narrow** escape, and `[UnscopedRef]` to **widen** it (opt out of the default scoping on `this` for struct methods).

```csharp
public struct S
{
    public int Value;

    [UnscopedRef]
    public ref int GetRef() => ref Value;  // caller can now keep this ref; otherwise scope-limited
}
```

The JIT doesn't enforce this — the compiler does. Once you compile, there's nothing stopping bad IL from aliasing freed storage; `ref safety` is a language-level contract.

## `ref struct`

A `ref struct` is a value type **pinned to the stack**. Used to safely hold refs into other storage without risking heap capture.

Canonical examples: `Span<T>`, `ReadOnlySpan<T>`, `Utf8JsonReader`, `SequenceReader<T>`, `ValueStringBuilder`.

### Restrictions (pre-C# 13)

- **Can't be boxed.** No `object`, no interface cast, no `typeof(ref struct) : IDisposable` awkwardness.
- **Can't be a field** of a non-`ref struct`.
- **Can't be a generic type argument** unless the generic opts in with `allows ref struct`.
- **Can't be captured** by a lambda, iterator, or `async` method (those close over state on the heap).
- **Can't cross `await` or `yield`.**
- **Can't implement interfaces** — until C# 13 relaxed this with [ref-struct interfaces](https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-13.0/ref-struct-interfaces); you can now implement interfaces, but you still can't box.

### `ref` fields (C# 11+)

A `ref struct` can declare **`ref` fields**, which is how `Span<T>` actually works:

```csharp
public readonly ref struct Span<T>
{
    private readonly ref T _reference;
    private readonly int _length;
    // …
}
```

The `ref` field holds a managed pointer to the element data (heap array, stackalloc buffer, unmanaged memory). `_length` bounds checks keep it safe.

### `readonly ref struct` and `readonly` methods

Make the whole struct `readonly` when its state is immutable (like `ReadOnlySpan<T>`). For mutable `ref struct`s, mark individual methods `readonly` when they don't mutate — the compiler will copy on invocation of non-`readonly` methods through a `readonly` reference otherwise.

### `using` with `ref struct`

C# 8 added duck-typed `Dispose` for `ref struct`s (they can't implement `IDisposable` pre-C# 13). `await using` never applies — they can't cross `await`.

```csharp
public ref struct Rental
{
    private readonly byte[]? _rented;
    public Rental(int size) => _rented = ArrayPool<byte>.Shared.Rent(size);
    public Span<byte> Span => _rented;
    public void Dispose() => ArrayPool<byte>.Shared.Return(_rented!);
}

using var r = new Rental(4096);
Work(r.Span);
```

## `record struct` / `readonly record struct`

A `record struct` is a **value type** with compiler-generated `Equals`, `GetHashCode`, `ToString`, `Deconstruct`, and a copy constructor wired up for `with`-expressions.

```csharp
public readonly record struct Money(decimal Amount, string Currency);

var m = new Money(10, "CAD");
var n = m with { Amount = 12 };       // new value, original unchanged
Console.WriteLine(m == n);             // false
Console.WriteLine(m.ToString());       // Money { Amount = 10, Currency = CAD }
```

### `readonly` vs mutable

- `readonly record struct` — all auto-properties are `{ get; init; }`; the struct is fully immutable. Preferred default.
- `record struct` (mutable) — properties are `{ get; set; }`. Lets you mutate through a local, but `with` still produces a copy.

### How equality is generated

For value types, the compiler emits **field-by-field** equality using `EqualityComparer<T>.Default` per field. No reflection, no boxing. Overriding is opt-in per-method; overriding `Equals(Other)` on the `record struct` lets you customize without touching `GetHashCode`.

### When `record struct` beats `record class`

- Tight loops where per-iteration allocation of a reference-type `record` shows up in profiler traces.
- Dictionary / set **keys** where structural equality saves writing `Equals`/`GetHashCode` by hand.
- Small data carriers (<= ~16 bytes rule of thumb). Above that, copy costs start to bite.

### When `record struct` loses

- Large payload (>64 bytes): every pass-by-value copies. Use `in` parameters or fall back to `record class`.
- Inheritance is not supported for `record struct` — value types are sealed by design.
- Cross-`await` / lambda capture: fine, unlike `ref struct`. But the copy-on-capture can be surprising.
- Mutable `record struct` in a `readonly` field position copies silently on every method call — same old struct-mutation trap.

### `record class` vs `class` vs `record struct` vs `struct`

| Goal                                 | Pick                                             |
|--------------------------------------|-------------------------------------------------|
| Reference identity, mutable state    | `class`                                         |
| Reference identity, structural eq    | `record class`                                  |
| Value identity, small payload, immutable | `readonly record struct`                    |
| Value identity, small payload, mutable   | `record struct` (rarely the right call)     |
| Small perf-critical aggregate, no equality needed | `readonly struct`                   |

## Putting them together

```csharp
public readonly record struct Range(int Start, int Length);

public static ref readonly int FindFirst(ReadOnlySpan<int> data, int target, out Range hit)
{
    for (int i = 0; i < data.Length; i++)
    {
        if (data[i] == target)
        {
            hit = new Range(i, 1);
            return ref data[i];
        }
    }
    hit = default;
    return ref Unsafe.NullRef<int>();
}
```

- `ReadOnlySpan<int>` is a `ref struct` — cannot escape, cannot be stored in a field.
- `ref readonly int` returns a pointer into `data`; the caller can read without copying but can't mutate.
- `Range` is a `readonly record struct` — free equality and deconstruction, no heap allocation.
- `Unsafe.NullRef<T>()` is the sentinel for "no ref" — check with `Unsafe.IsNullRef(ref x)`.

## Senior-level gotchas

- **A `ref` local defeats debugger inspection** on old tooling — it shows the *value* at a point in time, not the live storage. Step through carefully.
- **Reassigning a `ref` local** uses `ref` on the RHS too: `slot = ref other;` — forgetting `ref` silently copies and is a source of "why didn't my mutation stick" bugs.
- **`readonly struct` + method call through a `readonly` field** avoids the defensive copy; **non-`readonly` struct** through a `readonly` field copies on every method call. `readonly` is free performance, not just immutability.
- **`ref struct` can't implement `IDisposable` pre-C# 13**, but `using` still works duck-typed. Don't assume tooling (analyzers, AOT trimmers) handles the pattern everywhere.
- **`Span<T>` in an async method**: the compiler fails the build. Workaround: capture in a sync helper, extract only what you need into a heap-safe type, then `await`.
- **`record struct` equality uses `EqualityComparer<T>.Default`** per field — if a field is a `byte[]`, equality is **reference-based**, which is almost never what you want. Provide a custom `Equals` or use a `ReadOnlyMemory<byte>` + content hashing.
- **`with` on a `record struct` produces a new value on the stack** (good) but when assigned to a field copies again. In tight loops, prefer explicit field updates.
- **`scoped` and `[UnscopedRef]` are load-bearing for API design.** Missing `scoped` often means "this API can't accept a `stackalloc` buffer"; surprise `[UnscopedRef]` on a library method is a minefield for callers.
- **A `ref readonly` return from a class property does *not* prevent mutation of the enclosing object** — it only freezes the view through that ref. Don't confuse the two.
- **Array covariance does not apply to `Span<T>`**, on purpose. `Span<Dog>` is not a `Span<Animal>`. If the API wants polymorphism, design around `Span<T>` invariance.
