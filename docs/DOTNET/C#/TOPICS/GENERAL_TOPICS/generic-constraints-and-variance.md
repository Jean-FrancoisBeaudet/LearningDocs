# Generic constraints and variance

_Targets .NET 10 / C# 14. See also: [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md), [Interfaces / abstract / sealed](../OOP/interfaces-abstract-sealed-anonymous.md), [Readonly collections](../COLLECTIONS/readonly-collections.md)._

Generics give you parametric polymorphism without boxing. Constraints tell the compiler **what it's allowed to assume about `T`**; variance tells it **which reference conversions between `I<T>` and `I<U>` are safe**. Both are static: enforced by the compiler, erased-ish at runtime (generics are reified in the CLR, not erased like the JVM — an important difference).

## Constraints — the `where` clauses

| Constraint                | Meaning                                                                 |
|---------------------------|------------------------------------------------------------------------|
| `where T : class`         | Reference type (or nullable reference type if `class?`).               |
| `where T : class?`        | Reference type, no null-state warning.                                 |
| `where T : struct`        | Non-nullable value type. Implies `new()`.                              |
| `where T : notnull`       | Either a non-nullable reference type or a value type. Enforced by nullability analyzer. |
| `where T : unmanaged`     | Value type containing no references — enables `sizeof`, `fixed`, pointer arithmetic. |
| `where T : new()`         | Has a public parameterless ctor. `Activator.CreateInstance<T>()` is the lowering. |
| `where T : BaseClass`     | Derives from (or is) `BaseClass`.                                      |
| `where T : IFoo`          | Implements `IFoo`.                                                     |
| `where T1 : T2`           | Type parameter constraint — `T1` derives from `T2`.                    |
| `where T : allows ref struct` | (C# 13) `T` may be a ref struct. Changes what the generic method is allowed to do with it. |
| `where T : enum`          | Is an `enum`. (Combine with `struct`.)                                 |
| `where T : delegate`      | Is a delegate type.                                                    |

Order rules: `class` / `struct` / `unmanaged` / `notnull` first, then base type, then interfaces, then `new()` last.

### `notnull` vs `class` vs oblivious

```csharp
void A<T>(T x) where T : class   { }   // reference type; warns on `A<string?>(…)`
void B<T>(T x) where T : notnull { }   // either value or non-null ref; `B<int>` and `B<string>` ok
void C<T>(T x)                    { }   // "oblivious" — T? is platform-ambiguous
```

`notnull` is the one to reach for when you want "I can't accept null here, but I don't care if it's a struct or a class." It's enforced only when nullable annotations are on; legacy callers bypass it with warnings.

### `unmanaged` — the interop lever

```csharp
public static Span<byte> AsBytes<T>(ref T value) where T : unmanaged
    => MemoryMarshal.AsBytes(new Span<T>(ref value));
```

`unmanaged` ≈ "contains no managed references, recursively." Primitives, enums, pointers, and structs composed of them qualify. Required for `fixed`, `stackalloc T[]`, and most `MemoryMarshal` APIs — the GC needs to know nothing inside needs tracking.

### `allows ref struct` (C# 13)

Historically, `T` could not be substituted with a `ref struct` like `Span<T>` because the generic code might box it or store it in a field. C# 13 adds opt-in:

```csharp
public static bool TryParseAny<T>(ReadOnlySpan<char> s, out T value)
    where T : IParsable<T>, allows ref struct
    => T.TryParse(s, null, out value!);
```

Inside the method, `T` has the **union** of ref-struct restrictions: can't box, can't be a field of a non-ref-struct, can't be captured by a lambda, can't cross `await`. In exchange, callers can now substitute `Span<char>` etc.

### Static abstract members — the "self" trick

C# 11 added `static abstract` on interfaces, which unlocks generic math and constructor-like contracts:

```csharp
public interface IFactory<TSelf> where TSelf : IFactory<TSelf>
{
    static abstract TSelf Create(int id);
}

public static T Build<T>(int id) where T : IFactory<T> => T.Create(id);
```

The CRTP-style `where TSelf : IFactory<TSelf>` is how the compiler recovers the "return my own type" shape.

## Variance — `in` and `out`

Only **interface and delegate** generic parameters can be variant. Classes cannot. The rule is driven by **where `T` appears**:

- **`out T` (covariant)** — `T` appears only in **output** positions (returns, out params). `IEnumerable<Dog>` → `IEnumerable<Animal>` is safe, because a consumer of `Animal`s is fine with a producer of `Dog`s.
- **`in T` (contravariant)** — `T` appears only in **input** positions. `IComparer<Animal>` → `IComparer<Dog>` is safe, because something that compares any `Animal` also compares any `Dog`.
- Invariant (default) — `T` appears in both, or compiler can't tell. `List<Dog>` is **not** assignable to `List<Animal>`.

```csharp
public interface IRepository<in TId, out TEntity>
{
    TEntity? Get(TId id);         // TId input, TEntity output — both variant positions ok
}

IRepository<Guid, Dog>    dogs    = GetDogRepo();
IRepository<Guid, Animal> animals = dogs;          // covariant out
```

### Why classes can't be variant

Classes can have **fields**. A field position is **both input (on set) and output (on get)**, so it forces invariance. This is also why `List<T>` isn't variant — even without an explicit field, `Add(T)` is contravariant while `this[int] get` is covariant; together, invariant.

### Array covariance — the cautionary tale

Reference arrays are covariant at the **type system level**:

```csharp
Animal[] animals = new Dog[3];
animals[0] = new Cat();        // compiles — throws ArrayTypeMismatchException at runtime
```

This predates generics and can't be removed without breaking .NET 1.0 code. Modern equivalents (`List<T>`, `Span<T>`) are invariant on purpose. `IReadOnlyList<out T>` gets you the safe subset.

### Type inference and variance

The compiler will walk base types and variant interfaces during inference — but it picks the **most-derived** common type, which can surprise you:

```csharp
var list = new List<IEnumerable<string>> { "a".AsMemory().ToString().Split(' '), … };
// variance lets nested IEnumerable<T> widen; a heterogeneous collection initializer may
// lock onto a narrower type than you wanted. When in doubt, annotate explicitly:
List<IEnumerable<object>> list2 = [ …, … ];
```

## Worked example — a generic `Result<T, TError>`

```csharp
public readonly record struct Result<TValue, TError>(TValue? Value, TError? Error, bool IsOk)
    where TValue : notnull
    where TError : notnull
{
    public static Result<TValue, TError> Ok(TValue v)    => new(v, default, true);
    public static Result<TValue, TError> Fail(TError e)  => new(default, e, false);
}
```

- `notnull` keeps both slots meaningful (a `null` `TValue` when `IsOk` is a bug, not an absence).
- `record struct` gives structural equality for free and avoids allocation on the hot path.
- We don't mark `TValue` as `out` because we don't need variance for a result; marking it would also force us off `struct` (variance is interface-only).

## Senior-level gotchas

- **Generics are reified in the CLR.** Unlike Java erasure, `typeof(List<int>) != typeof(List<string>)`. This is why `Activator.CreateInstance` and `MakeGenericType` work at runtime and why JIT specializes value-type generics per-T (good: no boxing; bad: code size).
- **Constraint-inferred calls can devirtualize.** `where T : struct, ISomething` + call `x.Method()` on `T` often devirtualizes the interface dispatch in the JIT — one reason generic math is fast.
- **Variance is compile-time only.** `(IEnumerable<Animal>)dogList` succeeds as a *reference conversion*, not a cast-and-copy. Cheap.
- **`new()` constraint calls `Activator.CreateInstance<T>()`**, which for value types bypasses the ctor (returns `default`) but for reference types does a reflection-ish lookup. Prefer a factory delegate on hot paths.
- **Classes can't be covariant even if they look pure**, because fields exist. If you find yourself wanting a covariant class, extract an `interface IReadOnly…<out T>` and cast to it.
- **Implicit nullable-reference constraint drift.** Adding `where T : class` to a public generic is **not** binary-compatible with existing callers that passed nullable refs — they'll get new warnings. Use `class?` or `notnull` carefully on public APIs.
- **`allows ref struct` is one-way.** Once added, the body must obey ref-struct rules for `T` (no box, no capture). Adding it later is binary-compatible; removing it is a breaking change for `Span<T>` callers.
- **`struct` constraint implies `new()` implicitly** — you don't need both, and specifying both is a compile error.
- **Contravariant delegate assignment**: `Action<Animal>` → `Action<Dog>` works because `in` applies to the argument. Great for event handler adapters; surprising when combined with generic method groups.
