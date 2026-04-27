# Classes, records, structs

_Targets .NET 10 / C# 14. See also: [Constructors, properties, fields, methods](./constructors-properties-fields-methods.md), [Encapsulation, inheritance, polymorphism](./encapsulation-inheritance-polymorphism.md), [Interfaces, abstract, sealed, anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md), [Primary constructors](../GENERAL_TOPICS/primary-constructors.md), [ref return, ref struct, record struct](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md), [value types](../TYPES/value-types.md), [reference types](../TYPES/reference-types.md)._

C# offers four shapes of user-defined types: `class`, `struct`, `record class`, and `record struct` — plus the special-purpose `ref struct`. They split along two axes: **identity vs value semantics** (does equality compare references or contents?) and **reference vs value layout** (does the variable hold a pointer to the heap, or the bytes themselves?). Picking the wrong axis is the most common cause of subtle correctness and perf bugs in a mature codebase.

## The runtime split

| Shape           | Layout    | Equality default | Mutability default | Inheritance         |
|-----------------|-----------|------------------|--------------------|---------------------|
| `class`         | Reference | Reference (`==`) | Mutable            | Single base + many interfaces |
| `struct`        | Value     | Field-by-field via reflection (boxed!) | Mutable | None (sealed implicitly) |
| `record class`  | Reference | Structural (compiler-generated) | `init`-only by default | Single base record + interfaces |
| `record struct` | Value     | Structural (compiler-generated) | Mutable unless `readonly` | None |
| `ref struct`    | Stack-only value | Forbidden (cannot box) | Free | None; cannot implement interfaces (mostly) |

Layout drives lifetime. A `class` instance always lives on the GC heap and the variable holds a managed pointer; a `struct` is *inlined* into its containing storage (the stack frame, an array slot, a field of a class, a register if small enough). Equality drives meaning: `record` synthesises an `Equals` that compares **all instance fields**, not the reference.

## `class` — the workhorse

```csharp
public sealed class Order
{
    public Guid Id { get; }
    public decimal Total { get; private set; }

    public Order(Guid id, decimal total) => (Id, Total) = (id, total);

    public void Adjust(decimal delta) => Total += delta;
}
```

- Reference type → assignment copies the pointer; mutation is observable through every alias.
- Default `Equals` is reference equality. Override only if the type has a meaningful identity *value* (rarely the right call — usually you want a `record` instead).
- Eligible for finalization if you author a finalizer (don't, unless wrapping unmanaged resources — see [Finalizer and Dispose pattern](../MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md)).
- **Default to `sealed`**. Open inheritance is a runtime contract you rarely need; sealing lets the JIT devirtualize call sites and is reversible.

## `struct` — value type, no allocations on the stack path

```csharp
public readonly struct Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException();
        return new Money(Amount + other.Amount, Currency);
    }
}
```

Use a struct when:
- The value is conceptually a single number / coordinate / small bundle (`Point`, `Money`, `DateRange`).
- It's small (≤ 16 bytes is the usual rule of thumb — beyond that, copying through return/parameter starts to outweigh allocation savings).
- It's logically immutable.

`readonly struct` makes every instance method treat `this` as `readonly`; without it, the compiler defensively copies the struct on every member access through a `readonly` field — a silent perf foot-gun. Use it by default.

### The mutable-struct hazard

```csharp
public struct Counter { public int Value; public void Inc() => Value++; }

readonly Counter _c = new();
_c.Inc();          // compiles, but mutates a *defensive copy* of _c, not _c itself
Console.WriteLine(_c.Value); // 0
```

The CLR can't enforce `readonly` on the field's bytes if the method might write to `this`, so it copies. Same trap with `foreach` (the iteration variable is a copy), with `List<T>` indexer (`list[0].Inc()` mutates a temp), and with closures.

**Rule:** if it's a struct, make it `readonly struct` and return new values from operations. If you find yourself wanting `.Modify()` semantics, use a class.

## `record class` — reference type with value equality

```csharp
public record class Customer(Guid Id, string Name, string Email);

var a = new Customer(Guid.NewGuid(), "Ada", "ada@x.com");
var b = a with { Email = "ada@y.com" };

a == b;        // false — Email differs
a.Equals(b);   // false
a.Id == b.Id;  // true — copied by 'with'
```

What the compiler synthesises (visible in the IL):

- A primary constructor binding each parameter to a public `init` property.
- `Equals(T?)`, `Equals(object?)`, `GetHashCode()` — field-by-field.
- `==` / `!=` operators delegating to `Equals`.
- `Deconstruct(out …)` matching the primary ctor.
- A `protected virtual Type EqualityContract { get; }` returning `typeof(T)` — used to make derived records *not* equal to base records with the same fields.
- A protected copy constructor `protected Customer(Customer original)` — used by `with`.
- A `<Clone>$` method (the IL name; not callable from C#) used by `with`.
- `PrintMembers(StringBuilder)` and `ToString()` for the `Customer { Id = …, Name = … }` form.

Customise any of them by writing your own — e.g. omit a field from equality:

```csharp
public record class AuditedRow(int Id, string Data, DateTime CapturedAt)
{
    // Exclude CapturedAt from equality:
    public virtual bool Equals(AuditedRow? other) =>
        other is not null && Id == other.Id && Data == other.Data;

    public override int GetHashCode() => HashCode.Combine(Id, Data);
}
```

### Positional vs nominal records

```csharp
public record class A(int X, int Y);                // positional — primary ctor + auto-init properties
public record class B { public int X { get; init; } public int Y { get; init; } }  // nominal
```

Positional records get `Deconstruct` for free; nominal records do not. Mix freely (positional + extra body members), but be deliberate — positional parameters become **public** init properties whether you want them public or not.

### Inheritance and `EqualityContract`

```csharp
public record class Animal(string Name);
public record class Dog(string Name, string Breed) : Animal(Name);

Animal a = new Dog("Rex", "Lab");
Animal b = new Animal("Rex");
a == b;     // false — EqualityContract differs (Dog vs Animal)
```

Derived records inherit equality and refine it with their own contract. This is why `record` doesn't compose well with `class` in mixed hierarchies — equality semantics diverge.

## `record struct` — value type with structural equality

```csharp
public readonly record struct Point(int X, int Y);

new Point(1, 2) == new Point(1, 2);   // true, no boxing
```

Differs from `record class`:
- No `<Clone>$` — `with` produces a value-typed copy via the *primary constructor*, not a clone method.
- Properties default to `get; set;` (mutable!) unless the record is `readonly record struct` — then they're `readonly init`. **Default to `readonly record struct`** for the same reason as plain structs.
- No inheritance.

`record struct` shines for tiny immutable values you want hash/equals on without writing them by hand: `Point`, `Range`, `Money`, `Coordinates`. It composes well with `Dictionary<TKey, …>` and `HashSet<T>` because `GetHashCode` is generated.

## `ref struct` — stack-only

A `ref struct` (and the C# 11 `ref struct` interfaces, C# 13 `allows ref struct` constraint) is the type you reach for when you need a typed view over memory that *must not* migrate to the heap: `Span<T>`, `ReadOnlySpan<T>`, `Utf8JsonReader`, `SpanSplitEnumerator`. Not used for domain modelling — covered in detail in [Span&lt;T&gt;](../HEALTH_AND_PERFORMANCE/span-of-t.md) and [ref return, ref struct, record struct](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md).

## Decision table

| You need…                                                       | Choose                |
|-----------------------------------------------------------------|-----------------------|
| Identity, mutable state, behaviour-rich domain entity           | `sealed class`        |
| Immutable DTO, value-equality, easy `with` copies               | `record class`        |
| Tiny immutable value (≤ 16 bytes), no allocations               | `readonly record struct` |
| Performance-critical container with mutable fields, no equality | `readonly struct` + factory methods |
| Stack-only typed view over memory                               | `ref struct`          |
| Polymorphic root + closed set of variants                       | `abstract record class` + sealed leaves (or pattern-matching with classes) |

## Senior-level gotchas

- **Default `struct` equality boxes.** `ValueType.Equals` uses reflection on every field — a hot-path equality check on a big struct can allocate megabytes/sec. Override `Equals(T)` and `GetHashCode()` (or use a `record struct`).
- **`record class` parameters are `public init` properties whether you want them or not.** No `private` positional record. If a parameter is internal-only, drop the positional form and write the property explicitly.
- **`with` on a `record class` calls the *copy constructor*, not the primary ctor.** Validation logic placed in the primary ctor is bypassed. Put invariants in a property setter or in a parameterless `Validate()` method on the type.
- **`record struct`'s `with` copies field-by-field via the synthesised method.** Mutating one field then calling `with` produces unexpected results if your design assumed `with` always returned a *new* value — it does, but the source may already be the wrong thing.
- **Mutable struct in a `Dictionary<int, MyStruct>`** — indexer returns a copy. `dict[key].Mutate()` silently does nothing. `CollectionsMarshal.GetValueRefOrNullRef` is the by-ref alternative.
- **`record` inheritance breaks equality with siblings.** `Dog == Cat` for two records inheriting `Animal` is always false even with identical fields. Good for correctness, surprising for newcomers.
- **Sealing a `record` with `sealed record class Foo …`** removes `EqualityContract` virtualisation cost. If you don't intend to subclass, seal — same JIT devirt benefit as `sealed class`.
- **Auto-property in a `struct` ctor must initialise via `this = default` or all fields explicitly** (CS0843). C# 11 relaxed this for primary ctors; older patterns still trip it.
- **`Equals` on a `class` defaults to reference equality but `GetHashCode` defaults to a synchronisation-block hash** that is *stable per-instance* but unrelated to fields. Putting plain `class` instances in a `HashSet<T>` works only because the hash is unique per ref — moving the same logical entity into a new instance breaks lookups.
- **`record class : record struct` is a compile error.** Records can only inherit from records of the same kind. Mixing them in an "abstract base" hierarchy doesn't work — promote to a polymorphic interface or use sealed `record class` leaves under an `abstract record class` root.
- **`required` on a record property** (C# 11) is allowed but rare — the primary ctor already enforces presence. Useful only on nominal records.
- **`record struct` is not `readonly` by default**, even though its properties look like records. `var p = new Point(); p.X = 5;` compiles. Always write `readonly record struct` unless you have a deliberate reason.
- **Anonymous types compile to `internal sealed class` records-in-spirit** — see [interfaces / abstract / sealed / anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md) — but they predate `record` and have stricter limits (no inheritance, no naming).
