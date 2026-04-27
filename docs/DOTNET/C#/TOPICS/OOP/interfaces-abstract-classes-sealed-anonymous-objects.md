# Interfaces, abstract classes, sealed, anonymous objects

_Targets .NET 10 / C# 14. See also: [Classes, records, structs](./classes-records-structs.md), [Encapsulation, inheritance, polymorphism](./encapsulation-inheritance-polymorphism.md), [Static and generics](./static-and-generics.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md), [Default interface members](../GENERAL_TOPICS/default-interface-members.md)._

Two abstraction shapes (`interface`, `abstract class`) and two scope shapes (`sealed`, anonymous types) — together they cover most of the polymorphism surface that isn't generics. The interesting senior questions: **when do default interface members harm you**, **when does `sealed` matter to the JIT**, and **why is `static abstract` on interfaces the most consequential C# 11 feature**.

## Interface vs abstract class

| Concern                                      | `interface`                              | `abstract class`                       |
|----------------------------------------------|------------------------------------------|----------------------------------------|
| Multiple inheritance                         | Yes (any number)                         | No (single base only)                  |
| Instance fields                              | No                                       | Yes                                    |
| Constructors                                 | No (only `static` ctor since C# 8)       | Yes                                    |
| Default implementations                      | DIM (C# 8+); explicit interface only by default | Yes (regular methods)            |
| `static abstract` / `static virtual` members | Yes (C# 11)                              | Yes (`abstract static` since forever, but interface form enables generic dispatch) |
| Access modifiers on members                  | C# 8+ allows; default is `public`        | Full set                               |
| Versioning safety (adding a member)          | Binary break for implementers unless DIM | Binary safe if non-abstract            |
| Runtime cost                                 | IMT lookup (interface dispatch)          | V-table slot (slightly cheaper)        |
| Boxing of structs                            | Yes when called through the interface    | N/A (structs can't extend abstract classes) |

Rule of thumb:
- **Interface** for a *capability* (`IDisposable`, `IComparable<T>`, `IAsyncEnumerable<T>`).
- **Abstract class** for a *partial implementation* of a related family (`Stream`, `TextReader`, `IdentityUser`).
- Use both: an interface for the contract, an abstract class for a default implementation that downstream consumers can derive from. ASP.NET Core does this everywhere (`IFilter` + `Filter`).

```csharp
public interface IRepository<T>
{
    Task<T?> FindAsync(Guid id, CancellationToken ct);
    Task SaveAsync(T item, CancellationToken ct);
}

public abstract class RepositoryBase<T> : IRepository<T>
{
    protected ILogger Log { get; }
    protected RepositoryBase(ILogger log) => Log = log;

    public abstract Task<T?> FindAsync(Guid id, CancellationToken ct);

    public virtual async Task SaveAsync(T item, CancellationToken ct)
    {
        Log.LogInformation("Saving {Type}", typeof(T).Name);
        await SaveCoreAsync(item, ct);
    }

    protected abstract Task SaveCoreAsync(T item, CancellationToken ct);
}
```

## Default interface members (DIM)

```csharp
public interface ILogger
{
    void Log(LogLevel level, string message);

    // Default implementation — callers without an override get this:
    void LogInformation(string message) => Log(LogLevel.Information, message);
}
```

Available since C# 8 / .NET Core 3.0. The two reasons to use them:
1. **Adding a method to a published interface without breaking implementers.** With a default body, existing implementers still compile.
2. **Mixin-style helpers** — methods that delegate to other interface members.

When *not* to use them:
- Anything stateful (interfaces still can't have instance fields).
- Anything you'd reasonably override often — DIM is invoked through interface dispatch only when the receiver is typed *as the interface*. `concreteType.Method()` calls the *class*'s method, not the DIM, if the class has any same-named member. The "diamond" rules around DIM resolution are subtle (covered in [Default interface members](../GENERAL_TOPICS/default-interface-members.md)) and easy to get wrong.

DIMs require the runtime to support them — full .NET 5+ does; Xamarin/Mono and .NET Framework do not.

## Static abstract / static virtual interface members (C# 11)

Game-changer for generic algorithms:

```csharp
public interface IAddable<T> where T : IAddable<T>
{
    static abstract T Zero { get; }
    static abstract T operator +(T a, T b);
}

public static T Sum<T>(IEnumerable<T> items) where T : IAddable<T>
{
    var acc = T.Zero;
    foreach (var i in items) acc += i;
    return acc;
}
```

This unblocked generic math (`INumber<T>`, `IFloatingPoint<T>`, `IBinaryInteger<T>`) and the `IParsable<T>` / `ISpanParsable<T>` family. The dispatch is **resolved at JIT time per closed generic instantiation** — no v-table, no boxing, no runtime cost over a hand-written specialisation.

```csharp
public interface IParsable<TSelf> where TSelf : IParsable<TSelf>
{
    static abstract TSelf Parse(string s, IFormatProvider? provider);
    static abstract bool TryParse(string? s, IFormatProvider? provider, out TSelf result);
}

// Usage:
public static T ParseAll<T>(IEnumerable<string> lines) where T : IParsable<T> =>
    lines.Select(l => T.Parse(l, CultureInfo.InvariantCulture)).First();

ParseAll<int>(new[] { "1", "2" });
ParseAll<DateTime>(new[] { "2024-01-01" });
```

The `where T : IFoo<T>` self-referential constraint is the Curiously Recurring Template Pattern (CRTP) and is the price of admission for static-abstract dispatch.

## Explicit interface implementation

```csharp
public sealed class Bag : IEnumerable<int>, IDisposable
{
    public IEnumerator<int> GetEnumerator() => …;
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();   // explicit

    void IDisposable.Dispose() { … }   // explicit — only callable via IDisposable cast
}
```

Use cases:
- **Name collision** between two interfaces with the same member signature.
- **API hiding** — keep the interface method off the public type's public surface (`((IDisposable)bag).Dispose();` is the only way to call it).
- **Different return types** — the public method returns `Bag` while the interface form returns `object`.

The IL name is mangled (`MyNamespace.IInterface.MethodName`) — visible to reflection but not to IntelliSense on the concrete type.

## `sealed`

### On classes

```csharp
public sealed class Order { … }
```

- Prevents subclassing.
- Lets the JIT **devirtualise** every virtual call dispatched against a static type of `Order` — no v-table lookup. Free perf.
- Removes accidental coupling: callers can't sneak in a subclass with surprising overrides.
- Is *almost always* the right default for concrete domain types and DI implementations.

Adding `sealed` to an existing public class is a binary break — anything that subclassed it stops compiling. Removing `sealed` is non-breaking. So: add `sealed` early.

### On overrides

```csharp
public class A { public virtual void X() { } }
public class B : A { public override void X() { } }
public class C : B { public sealed override void X() { } }   // C and below cannot override
public class D : C { /* public override void X() — error CS0239 */ }
```

Useful when you need to extend (still allow subclassing for other members) but lock down a single override against further specialisation.

### `sealed record`

A `record class` is sealable too. Sealing also stops the `EqualityContract` virtualisation cost — equality between two sealed-record instances skips the contract check. Useful when the record is a leaf in a hierarchy (`abstract record Shape` + `sealed record Circle`).

## Anonymous types

```csharp
var p = new { Name = "Ada", Age = 36 };
Console.WriteLine(p.Name);
```

What the compiler emits:
- An `internal sealed class <>f__AnonymousType0<TName, TAge>` per *shape* (set of property names + arity) per assembly.
- Read-only properties (no setters).
- A constructor taking all properties in declaration order.
- `Equals` / `GetHashCode` based on all properties (structural equality).
- A `ToString` like `{ Name = Ada, Age = 36 }`.

Two anonymous types with the **same property names in the same order** in the *same assembly* unify to the same generic type. Cross-assembly they don't.

### Constraints

- No naming — `var` only.
- No method members — properties only.
- Properties are read-only and `init`-able only via the ctor (no `with` support; not a record).
- Cannot escape methods cleanly — the type has no name, so a method return type of "anonymous shape" requires generics + tuple-like deconstruction.
- Implicitly typed; nullability flows in but you can't annotate.

### When to use them

LINQ projections that don't escape the local query:

```csharp
var summary = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Amount) })
    .ToList();
```

Once you need to flow the shape across method boundaries → graduate to `record class` (named, public, no surprises).

### Anonymous vs `record` vs tuple

| Need                                                  | Use                  |
|-------------------------------------------------------|----------------------|
| Quick, in-method shape with named members             | Anonymous type       |
| Reusable, named, value-equality DTO across methods    | `record class`       |
| Lightweight unnamed N-tuple (positional)              | `(int, string)` tuple|
| Small immutable value with structural equality        | `readonly record struct` |

## Senior-level gotchas

- **Adding a member to a published interface is a binary break** unless you give it a default implementation — and a DIM still requires consumers to be on a runtime that supports it. If you ship libraries on `netstandard2.0`, skip DIMs entirely.
- **DIM resolution is *not* virtual through the type hierarchy.** If `class C : IFoo` and `IFoo` has a DIM `Bar()`, then `var c = new C(); c.Bar();` is a **compile error** — the DIM is only callable through an `IFoo` reference. Surprising, deliberate.
- **Static-abstract interface methods can't be invoked off an interface variable** — only off a generic constraint. `IFoo.M()` doesn't compile; `T.M()` where `T : IFoo` does. The JIT picks the implementation per closed `T`.
- **`sealed` on a struct is implicit and not allowed explicitly.** Structs are sealed; writing `sealed struct Foo` is CS0106. Records of either kind can be sealed.
- **`sealed override` doesn't seal the *type*** — only the method. Subclassing remains legal; just that one method is locked.
- **Interface dispatch on a struct boxes** when called through the interface variable. `((IComparable<int>)5).CompareTo(6)` allocates. The static-abstract form (`int.CompareTo(5, 6)` via generic constraint) does not.
- **Explicit interface implementation cannot be `virtual` directly** — to allow subclasses to override the explicit form, write a public protected method and call it from the explicit member. Awkward; rarely worth it.
- **Anonymous types cannot be used in `dynamic`** in a useful way — properties are not exposed via the DLR through anonymous-type metadata cleanly. Prefer `ExpandoObject` if you need open-shape dynamics.
- **Anonymous-type cross-assembly type identity isn't preserved** — two assemblies declaring `new { X = 1 }` produce *different* CLR types. Comparing instances across assemblies fails reflection-based equality unless you reduce to `IDictionary<string, object>`.
- **`abstract` static methods on a non-interface class predate C# 11** but never participated in generic dispatch — they were just "must override on each subclass." The interface form is what enables `where T : IFoo<T>` static-method calls.
- **Default interface members on a generic interface** still don't allow per-instance state via the interface — there is no `this.field` in a DIM body. Workaround: ConditionalWeakTable, or admit you want an abstract class.
- **`record` implementing an interface** auto-generates the property — but if the interface declares `int X { get; }` and your positional record is `record Foo(int X)`, the synthesised `init` setter is non-public to the interface contract; explicit casting works fine. Watch when consumers do `IFoo f = new Foo(1)` — `f.X` reads the same field, no surprise.
- **`sealed` removes a versioning escape hatch.** If your library ever wanted users to subclass `Order` for cross-cutting concerns (logging, validation), sealing closes that door. Compose-by-default is the right answer; sealing forces it.
- **Anonymous type's `Equals` / `GetHashCode` allocate** when comparing `string` properties (delegates to the string's hashcode — fine) but the compiler-generated `ToString` allocates a `StringBuilder`. Don't use anonymous types as cache keys in hot paths; promote to a `record struct`.
- **Adding `static abstract` to an interface is a binary break** — every existing implementation must add the static member. Treat it like adding an `abstract` instance method.
