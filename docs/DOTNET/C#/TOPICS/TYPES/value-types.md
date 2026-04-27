# Value types

_Targets .NET 10 / C# 14. See also: [Reference types](./reference-types.md), [Classes, records, structs](../OOP/classes-records-structs.md), [ref return, ref struct, record struct](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md), [Primary constructors](../GENERAL_TOPICS/primary-constructors.md), [Tuples, attributes, regex](../GENERAL_TOPICS/tuples-attributes-regex.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md), [Allocations and boxing](../HEALTH_AND_PERFORMANCE/allocations-and-boxing.md)._

A **value type** stores its bytes *inline* — in the stack frame, in a register, in an array element, or as a field of a containing object. The variable *is* the data. Contrast with a [reference type](./reference-types.md), where the variable is a pointer to a heap object. Choosing value semantics is a runtime-layout decision before it is a design decision: it changes copy cost, GC pressure, equality, generic dispatch, and async-state-machine size. Get it wrong and you either bleed allocations or copy 200-byte structs through every parameter.

## What counts as a value type

Every `System.ValueType` derivative — and `System.ValueType` itself derives from `object`, which is the source of boxing. The set:

- Primitives — `int`, `long`, `double`, `bool`, `char`, `byte`, …
- `struct` — user-defined.
- `enum` — a `struct` with a fixed underlying integral type.
- `ValueTuple<…>` — `(int, string)` desugars to `ValueTuple<int, string>`.
- `Nullable<T>` — itself a `struct`.
- `ref struct` — value type with stack-only escape rules; covered separately in [ref return, ref struct, record struct](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md).

`System.ValueType` and `System.Enum` are **abstract reference types** in the metadata (they live on the heap when boxed) but the things you derive from them are value types.

## Where the bytes live

| Storage location              | Example                              |
|------------------------------|--------------------------------------|
| Stack frame (local)          | `int x = 5;`                         |
| CPU register                 | small struct in a tight method       |
| Array element                | `int[] a;` — bytes are inline        |
| Field of a class             | inlined into the heap object         |
| Field of another struct      | inlined into the outer struct        |
| GC heap (boxed)              | `object o = 5;` — allocation         |

Inlining is what makes `int[]` a contiguous block of 4 *N* bytes and `MyStruct[]` a contiguous block of `sizeof(MyStruct) * N`. It is also why mutating `array[i].Field = …` works while mutating `list[i].Field = …` does not — the indexer of `List<T>` returns a *copy*.

## Copy semantics

Assignment, parameter passing, and return all copy the bytes:

```csharp
Point p = new(1, 2);
Point q = p;          // bit-copy
q.X = 99;             // p.X is still 1

void Move(Point pt)   // pt is a fresh copy
{
    pt.X = 0;         // caller's value unaffected
}
```

The cost is proportional to `sizeof(T)`. For `Point` (8 bytes) it is one register move; for a 256-byte struct it is a `cpblk` plus cache pressure. Pass large structs by reference:

```csharp
void Render(in Matrix4x4 m)   // read-only ref, no copy
void Translate(ref Vector3 v) // mutable ref
bool TryParse(ReadOnlySpan<char> s, out Point result) // ref-out
```

`in` parameters are **read-only managed pointers**. The compiler will silently take a defensive copy on every member call through `in` if the struct is **not** `readonly struct` — see the gotchas below.

## Boxing and unboxing

Boxing wraps a value type in a heap object so it can be assigned to `object`, an interface, or a non-constrained generic.

```csharp
int n = 42;
object o = n;          // box: alloc heap object, copy 4 bytes in
int m = (int)o;        // unbox.any: copy 4 bytes back, type-check
```

IL-level: `box` for the assignment, `unbox.any` (or `unbox` + `ldobj`) for the cast. Each box is a heap allocation — typically 24 bytes on 64-bit (object header 16B + payload, rounded). Hot-path boxing turns into the GC pressure profilers love to flag.

Boxes you may not notice:

- `int.GetHashCode()` does not box (compiler calls the override directly), but `((object)n).GetHashCode()` does.
- `string.Format("{0}", n)` boxes `n` to satisfy the `object[]` parameter (use interpolated strings + interpolated string handlers — see [Interpolated string handlers](../GENERAL_TOPICS/interpolated-string-handlers.md)).
- `IEnumerable<int>` over a non-generic collection (`ArrayList`) boxes every element.
- Casting a struct to an interface — `IComparable c = myStruct;` — boxes.
- Using a struct as a `Dictionary<,>` key with `EqualityComparer<T>.Default` is *not* a box (generic specialisation), but calling `dict[boxedKey]` where `dict` is the non-generic `Hashtable` is.
- `Enum.HasFlag` boxed both arguments before .NET 5; on .NET 5+ the JIT intrinsifies it. Rule of thumb on hot paths: still prefer `(value & flag) == flag`.

Use [Allocations and boxing](../HEALTH_AND_PERFORMANCE/allocations-and-boxing.md) to drive this from the profiler side.

## `default` and zero initialisation

Every value type has a *zero value*: `default(T)` is the bit pattern of all zeros. Fields, array elements, and `stackalloc` buffers start as `default(T)`. There is no "uninitialised struct" in safe C#.

```csharp
Point p = default;        // (0, 0)
Span<int> s = stackalloc int[8]; // all zero
```

Until C# 10 a struct could not declare a parameterless constructor — `new MyStruct()` always meant `default(MyStruct)`. C# 10 lifted that restriction, but **`default(T)` still bypasses the parameterless ctor**. Field initialisers and a parameterless ctor only run when the user writes `new T()`, not when the runtime zero-initialises an array, a class field, or a `stackalloc` buffer. Designing a struct that is invalid in its zero-initialised state is almost always a bug.

```csharp
public struct DontDoThis
{
    public DontDoThis() => Items = new List<int>();
    public List<int> Items { get; }   // null after default(DontDoThis)
}
```

## Layout, size, and `[StructLayout]`

The runtime is free to reorder fields under the default `LayoutKind.Auto` to minimise padding. For interop or memory mapping, pin the layout:

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct PacketHeader
{
    public byte Version;
    public byte Flags;
    public ushort Length;
    public uint  Sequence;
}

// Union: same offset for two fields
[StructLayout(LayoutKind.Explicit)]
public struct DoubleBits
{
    [FieldOffset(0)] public double Value;
    [FieldOffset(0)] public long   Bits;
}
```

- `Marshal.SizeOf<T>()` — interop size, follows `[StructLayout]`. Doesn't work for managed-only types containing references.
- `Unsafe.SizeOf<T>()` — managed size as the JIT lays it out. Always available, including for ref-containing structs.

The two can disagree (e.g. `bool` is 4 bytes via `Marshal`, 1 byte via `Unsafe`). For .NET-internal reasoning use `Unsafe.SizeOf<T>()`.

## Equality

`ValueType.Equals(object)` is the default and it is **slow and allocating**:

1. It boxes the right-hand side (already boxed, since the parameter is `object`) — actually the receiver is the boxed `this`.
2. It uses **reflection** on every field if the type doesn't blittably bit-compare (most don't, due to padding and reference fields).
3. It returns boxed bool from each recursive call.

`ValueType.GetHashCode` is just as bad: a reflective walk that historically combined only the **first non-null field**. Two equal-looking structs commonly hash to the same bucket.

Override always:

```csharp
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount  { get; }
    public string  Currency { get; }

    public Money(decimal amount, string currency) =>
        (Amount, Currency) = (amount, currency);

    public bool Equals(Money other) =>
        Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(Money a, Money b) =>  a.Equals(b);
    public static bool operator !=(Money a, Money b) => !a.Equals(b);
}
```

Or skip the boilerplate: `readonly record struct Money(decimal Amount, string Currency);` synthesises all of the above (see [classes, records, structs](../OOP/classes-records-structs.md)).

## `readonly struct` and `readonly` members

Marking the *type* `readonly`:

- All instance fields and auto-properties become read-only.
- Every method's `this` is treated as read-only.
- The compiler stops emitting **defensive copies** when you call methods through an `in` parameter or a `readonly` field.

Without `readonly` on the struct, this code copies on every call:

```csharp
public struct Counter { public int N; public int Get() => N; }

class Holder
{
    public readonly Counter C = new();
    public int Read() => C.Get();   // defensive copy of C, then call
}
```

If `Counter` were a `readonly struct` (or `Get` were a `readonly` member), the copy disappears.

**Default to `readonly struct`.** Mutable structs are rarely correct; the language tolerates them for backwards compat and for performance-driven cases (e.g. `List<T>.Enumerator`).

## Enums

```csharp
public enum LogLevel : byte { Trace, Debug, Info, Warn, Error, Fatal }
[Flags] public enum FilePermissions { None = 0, Read = 1, Write = 2, Execute = 4 }
```

- Underlying type defaults to `int`. Pick the smallest practical for arrays/columns; pick `int` for general use.
- `[Flags]` is metadata-only — it changes `ToString()` and signals intent, but the compiler does not enforce powers-of-two values.
- `Enum.HasFlag(x)` boxed pre-.NET 5; the JIT now intrinsifies the common case. On hot paths still prefer `(value & flag) == flag` — explicit, allocation-free, no method call.
- `Enum.Parse`, `Enum.GetValues`, `Enum.GetNames` allocate. Cache results, or use `Enum.TryParse` + a static `FrozenDictionary` for hot lookups (see [frozen collections](../COLLECTIONS/frozen-collections.md)).

## Tuples

`(int A, string B)` is `ValueTuple<int, string>` — a mutable value type, no allocation, structural equality via `IEquatable<>`.

`Tuple<int, string>` is the legacy reference type — heap-allocated, immutable, awkward `.Item1`/`.Item2` access. Use it only to interop with code that pre-dates `ValueTuple`. Detail: [Tuples, attributes, regex](../GENERAL_TOPICS/tuples-attributes-regex.md).

## `Nullable<T>` — value types that can be null

`int?` is `Nullable<int>`, a struct with two fields:

```csharp
public struct Nullable<T> where T : struct
{
    public bool HasValue { get; }
    public T    Value    { get; }   // throws InvalidOperationException if !HasValue
}
```

Boxing rule: `(object)(int?)null` is `null`, not a boxed `Nullable<int>`. Unbox: `(int?)objectThatBoxesAnInt` works — the runtime detects the underlying type. This is the only place where the runtime "lies" about the boxed type's identity, and it is intentional.

`null`-safe operators:

```csharp
int? a = null;
int  b = a ?? 0;            // null-coalescing
int? c = a + 1;             // lifted operator: null + 1 == null
```

## Generic constraints

```csharp
where T : struct           // any value type *except* Nullable<T>
where T : unmanaged        // value type with no reference fields, recursively
where T : new()            // has parameterless ctor (always true for structs)
where T : Enum             // any enum
```

Notes:

- `where T : struct` excludes `Nullable<U>` so that `T?` in the method body unambiguously means `Nullable<T>`.
- `where T : unmanaged` is what unlocks `sizeof(T)`, `stackalloc T[n]`, and `T*` pointers.
- `Span<T>` and `Memory<T>` work with any `T`, but `Span<T>` is a `ref struct`, so `T` cannot itself be a `ref struct` until C# 13's `allows ref struct` constraint.

See [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md) for the full taxonomy.

## When to choose a value type

Pick `struct` (almost always `readonly struct` or `readonly record struct`) when **all** of these hold:

- The instance represents a **value**, not an **identity** (`Money`, `Point`, `Range`, `Hash`).
- Size is small — rule of thumb ≤ 16 bytes (two pointers on 64-bit). Up to 32 bytes can still pay if you avoid copies via `in`.
- The type is logically **immutable**.
- You don't need polymorphism (value types are sealed).
- You don't intend to put it behind an interface in a hot path (boxes).

Pick a class when you need identity, polymorphism, mutation visible across aliases, or a payload bigger than ~32 bytes.

The decision table in [classes, records, structs](../OOP/classes-records-structs.md#decision-table) covers the records dimension.

## Senior-level gotchas

- **Mutable struct in a `foreach` is a copy.** `foreach (var p in points) p.Move();` mutates a temporary that the loop discards. Same for `List<T>` indexer (`list[i].Mutate()` compiles, mutates a copy). Use `CollectionsMarshal.AsSpan(list)[i].Mutate()` if you genuinely need to mutate in-place.
- **`Dictionary<K, V>.this[K]` returns a copy** for value-type `V`. `dict[key].Field = …` is a silent no-op. Use `CollectionsMarshal.GetValueRefOrNullRef(dict, key)` for by-ref access.
- **`in` parameter + non-`readonly` struct = defensive copy on every member call.** Make the struct `readonly` *or* mark the methods you call `readonly`. Verify with `dotnet build -p:EmitCompilerGeneratedFiles=true` or sharplab — defensive copies show up as `cpblk`/`stloc` in the IL.
- **`ValueType.GetHashCode` reflectively reads the first non-null field.** Two `(0, 1, 2)` and `(0, 9, 9)` `Triple` structs hash identically. Always override.
- **Big structs in async methods bloat the state machine.** Every captured local — including the receiver of a struct method — is a field of the compiler-generated state machine class on the heap. A 200-byte struct in a hot async path allocates 200 bytes per invocation **even though the struct itself is "stack-allocated"**.
- **`stackalloc` size is unbounded by default.** `stackalloc byte[userSuppliedLength]` is a stack overflow waiting to happen. Bound it (`length <= 1024` then fall back to `ArrayPool`) or use `Span<T>` with the heap.
- **Passing a struct as `IEnumerable<…>` boxes it.** `foreach (var x in myStruct)` does *not* box (duck-typed enumerator pattern), but `IEnumerable<int> e = myStruct;` does. The `List<T>.Enumerator` is the canonical example — it's a struct precisely so that `foreach` doesn't allocate.
- **Struct equality with reference fields uses `EqualityComparer<T>.Default`** for those fields under the synthesised `record struct` `Equals` — meaning `byte[]` comparison is **reference**, not content. Override `Equals` if you have collection fields.
- **`new T()` for a generic `T : struct` does *not* call your parameterless ctor before .NET 6.** It emits `Activator.CreateInstance<T>()`, which used to skip user ctors. .NET 6+ honours them, but interop and old runtimes may not.
- **Auto-property setter on a struct mutates `this`** — which is what makes `set` properties on a `readonly struct` illegal at the language level, and what makes them subtle in a non-`readonly` struct (calling the setter through a `readonly` field copies the receiver, mutates the copy, discards it).
- **`Span<T>` over a struct field of a class is fine; over a struct local is fine; over a struct in a `static readonly` field requires `Unsafe.As<…>` because `readonly` would forbid taking a writable ref.** Know the escape rules.
- **Covariance does not apply to value types.** No `(int)1 is object o` magic; boxing is always explicit at the IL level. `List<int>` is not a `List<object>`. Same for `Span<T>`.
- **`record struct` is *mutable* by default** — `Point` from `record struct Point(int X, int Y)` lets you write `p.X = 5`. Always `readonly record struct` unless you have a reason.
- **`default(T)` skips field initialisers and parameterless ctors** — array allocation, `Array.Resize`, `stackalloc`, and uninitialised class fields all produce zero-bit instances. Design for a valid zero state, or accept that the type cannot be safely default-constructed (and document it).
