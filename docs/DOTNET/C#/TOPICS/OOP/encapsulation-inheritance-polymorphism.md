# Encapsulation, inheritance, polymorphism

_Targets .NET 10 / C# 14. See also: [Classes, records, structs](./classes-records-structs.md), [Interfaces, abstract, sealed, anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md), [Static and generics](./static-and-generics.md), [Constructors, properties, fields, methods](./constructors-properties-fields-methods.md), [Primary constructors](../GENERAL_TOPICS/primary-constructors.md)._

The "three pillars" framing is older than C# itself, and at a senior level the interesting questions are about **how the CLR implements them** and **when they cost you**. Encapsulation is enforced by the metadata + JIT, not by the C# compiler alone. Inheritance is single-base + many-interfaces by deliberate runtime design. Polymorphism in modern C# is increasingly *pattern-matching* and *generic-static-dispatch* rather than virtual calls.

## Encapsulation

### Access modifiers

| Modifier              | Visible to                                                  |
|-----------------------|-------------------------------------------------------------|
| `public`              | Everyone                                                    |
| `internal`            | Same assembly                                               |
| `protected`           | Same class and any derived class (any assembly)             |
| `protected internal`  | Same assembly **OR** derived class                          |
| `private protected`   | Same assembly **AND** derived class                         |
| `private`             | Same class only (and nested types of that class)            |
| `file` (C# 11)        | Same source file only                                       |

Type-level vs member-level: top-level types can only be `public`, `internal`, or `file` (no `private` top-level types). Nested types can be any of the above relative to the containing type.

### `internal` + `InternalsVisibleTo`

```csharp
// AssemblyInfo.cs (or .csproj <ItemGroup><InternalsVisibleTo Include="MyLib.Tests"/>):
[assembly: InternalsVisibleTo("MyLib.Tests")]
```

Lets test projects see `internal` members without making them `public`. Combined with `public sealed class` + `internal` ctors, you can lock construction to the owning assembly while keeping the *type* visible.

### `file` types

```csharp
// In one .cs file:
file sealed class JsonHelpers { … }

// In another .cs file in the same project:
file sealed class JsonHelpers { … }   // unrelated, no collision
```

A type used by exactly one source file (or by a generated file). Source generators emit `file`-scoped helpers to avoid leaking implementation details. No reflection access from outside the file (the type's metadata name is mangled with the file's hash).

### The `field` keyword and tightening encapsulation

```csharp
public string Name
{
    get => field;
    set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
}
```

Removes the temptation to expose a `private string _name` field next to the auto-property — there's literally no separate field name to leak. Reduces the surface for accidental direct mutation.

### Encapsulation isn't security

`private` is a compile-time + IL metadata flag. Reflection (`BindingFlags.NonPublic`) and `UnsafeAccessor` (.NET 8+) bypass it cleanly. Treat encapsulation as a design tool, not a trust boundary — if you need a real boundary, use a different process or an `internal`-only type plus signed assemblies.

## Inheritance

### Single base, many interfaces

```csharp
public abstract class Shape { public abstract double Area(); }
public sealed class Circle(double r) : Shape, IFormattable
{
    public override double Area() => Math.PI * r * r;
    public string ToString(string? format, IFormatProvider? provider) => $"Circle({r})";
}
```

C# (and the CLR) only allow one base class — by deliberate design, to avoid the "diamond" problem and the v-table merging that C++ inherits. Interfaces give you the rest: a class may implement any number, and an interface may inherit from any number of other interfaces.

Why one base class? Layout. The CLR stores instance fields by walking the inheritance chain top-down — there's a single linear field offset table. Multiple inheritance would require either virtual base offsets (C++'s solution) or interface-style indirection on every field access. The runtime picked simplicity.

### Base call ordering

Same as covered in [Constructors, properties, fields, methods](./constructors-properties-fields-methods.md#instance-constructors): base ctor runs first, then derived field initialisers, then derived ctor body. A `virtual` call in the base ctor reaches the *derived* override — CA2214.

### Primary ctor + base call

```csharp
public abstract class Shape(string name)
{
    public string Name => name;
}

public sealed class Square(double side) : Shape($"square-{Guid.NewGuid()}")
{
    public double Side => side;
}
```

The `: Shape(...)` is a *base ctor invocation*, not just a base type reference. Useful and concise; the gotcha is that **each level captures its own primary-ctor parameter independently**, so `class Derived(ILogger log) : Base(log)` stores `log` in two private fields. See [primary constructors](../GENERAL_TOPICS/primary-constructors.md).

### `protected` vs `private` field design

If a base class wants subclasses to read a value but not mutate it, prefer:

```csharp
protected internal string Name { get; }   // visible, immutable
```

over `protected string _name;` which exposes the field directly and lets a subclass replace the reference.

### `new` modifier — name-hiding

```csharp
public class Base { public int X => 1; }
public class Derived : Base { public new int X => 2; }

Base b = new Derived();
Console.WriteLine(b.X);   // 1 — uses Base.X (no virtual)
Console.WriteLine(((Derived)b).X);  // 2
```

`new` says "I'm intentionally hiding the base member, not overriding it." It's a static-dispatch shadow. Almost always a code smell — if you want polymorphism, use `virtual` + `override`. Analyzer CS0108 warns when you forgot the keyword.

## Polymorphism

### `virtual` / `override` / `sealed`

```csharp
public abstract class Repository
{
    public virtual void Save() => Console.WriteLine("default save");
}

public class SqlRepo : Repository
{
    public override void Save() => Console.WriteLine("sql save");
}

public sealed class CachedSqlRepo : SqlRepo
{
    public sealed override void Save() => Console.WriteLine("cached save");
}
```

- `virtual` puts the method in the type's v-table.
- `override` writes a new entry into the v-table at the same slot.
- `sealed override` prevents further subclasses from overriding — the JIT may then inline / devirtualize.
- `sealed` on the *class* seals **all** virtual methods.

### V-table dispatch — what the JIT does

A virtual call compiles to:

```text
mov  rax, [this+0]      ; load method table pointer
mov  rax, [rax+slotOff] ; load v-table slot
call rax                ; indirect call
```

Two memory loads + an indirect branch. The JIT can devirtualize when the static type is `sealed` or when guarded devirtualisation (.NET 7+ tiered compilation) sees a single concrete type at runtime. Sealing classes is a free perf hint when you don't need extension.

### Interface dispatch

A different, slightly more expensive mechanism: every type's method table has an *interface map* (a list of `(interface, slot)` pairs). The JIT either uses the IMT (Interface Method Table) lookup or — for tiny interfaces — special-cases `IDisposable` etc. Per-call cost is comparable to virtual dispatch on modern CPUs.

### Covariant return types (C# 9)

```csharp
public abstract class Animal { public abstract Animal Clone(); }
public sealed class Dog : Animal { public override Dog Clone() => new(); }
```

The override can return a *more derived* type — callers with a `Dog` reference don't need to cast. Compiler emits a synthetic bridge method to satisfy the base signature.

### Pattern matching as polymorphism

Modern dispatch increasingly happens via switch expressions on types/shapes:

```csharp
public static decimal Area(Shape s) => s switch
{
    Circle { Radius: var r } => Math.PI * r * r,
    Square { Side: var w }   => w * w,
    Rectangle (var w, var h) => w * h,
    _ => throw new InvalidOperationException()
};
```

Pros: closed sets are explicit (compiler nags on missing cases when `Shape` is a discriminated record hierarchy + sealed leaves), no v-table cost, behaviour lives next to the data.

Cons: doesn't extend without modifying the central switch — Open/Closed principle inverted. Choose by axis of change: **frequent new operations on a fixed shape** → pattern matching; **frequent new shapes for fixed operations** → virtual methods.

### Generic-static dispatch (C# 11)

`static abstract` interface members (covered in [interfaces / abstract / sealed / anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md)) give you compile-time polymorphism — generic math is the canonical example. No v-table cost; the JIT specialises per `T`.

## Composition vs inheritance

```csharp
// Inheritance — tightly coupled, single base, fragile to base changes:
public class CachingRepository : Repository { … }

// Composition — loose, swappable, easy to test:
public sealed class CachingRepository(IRepository inner, IMemoryCache cache) : IRepository
{
    public async Task<Order?> FindAsync(Guid id, CancellationToken ct) =>
        cache.GetOrCreate(id, _ => inner.FindAsync(id, ct).GetAwaiter().GetResult());
}
```

Composition wins by default. Reach for inheritance only when:
1. There is genuine **subtype-of** semantics (`Square` *is a* `Shape`).
2. You want to share template-method skeletons that subclasses fill in.
3. Equality/identity flows through the hierarchy (`record` chains).

In every other case — caching, retry, logging, instrumentation, decoration — compose.

## Senior-level gotchas

- **Sealed-by-default class design.** `public sealed class` should be the muscle-memory default; relax to `public class` only when the inheritance is documented and tested. Sealing enables JIT devirtualisation and is a non-breaking thing to add but a breaking thing to remove.
- **`virtual` calls in constructors** — CA2214. The override sees a *partially constructed* derived instance. The fix is the template-method pattern: factor virtual init into a separate method called *after* construction.
- **`new` for hiding base members** — almost always wrong. CS0108 fires when the base member changes and yours is the silent shadow. If you really need name reuse, mark it explicitly: `public new int X { … }`.
- **Property-changing-to-method binary break.** Subclasses compiled against a `public int Count { get; }` won't link against a recompiled base where it became `public int Count()`. Stable inheritance hierarchies need careful API governance.
- **`base.Method()` from an override** is non-virtual — calls exactly the base implementation, even if a deeper subclass also overrides. Use `this` for further polymorphism (and accept that's almost always a refactor smell).
- **Adding a `protected` member is not binary-compatible** in the strictest sense — subclasses in other assemblies don't need recompilation, but a subclass that *also* added the same name now collides. Document the protected surface as part of the public contract.
- **Interface dispatch defeats sealing** — `(IFoo)x.DoIt()` goes through IMT even if `x`'s static type is `sealed FooImpl`. The JIT can sometimes guarded-devirtualise, but assume it won't on cold paths.
- **Covariant return types break legacy reflection** that walks `MethodInfo.ReturnType` expecting it to match the base — a synthetic bridge method appears in the type with the base return type. `BindingFlags.DeclaredOnly` + filtering on attributes fixes it.
- **`InternalsVisibleTo` is transitively trusted** — anything in the named assembly can call your `internal` API. Test projects, mocking frameworks, and source-generator output share that surface; don't put authentication-critical code in `internal`-only members and assume tests can't reach it.
- **`file`-scoped types can collide across generators.** Two source generators that both emit a `file class JsonHelpers` in different files → no collision. But the same generator running twice → undefined; the build aborts.
- **`protected internal` is OR, `private protected` is AND.** Easy to flip mentally. The `internal` part of `protected internal` makes it visible to *all* same-assembly code, not just same-assembly subclasses.
- **Pattern matching exhaustiveness** is checked only when the discriminating expression is a sealed record hierarchy or an enum-like closed set. For an open hierarchy (`abstract class Shape` + assembly-external subclasses), the compiler can't prove exhaustiveness — the `_ => throw` arm is mandatory.
- **`override sealed` versus `sealed override`** — both compile and mean the same thing, but the convention is `sealed override` (modifier order matches `public static readonly`).
- **Encapsulation breaks via `UnsafeAccessor` (.NET 8)** — generated accessors can read `private` fields with no reflection cost. Fast, useful for serializers; treat your `private` fields as *implementation* not *secret*.
- **Composing through a wrapper that implements the same interface** is sometimes called the **Decorator** pattern; ASP.NET Core DI doesn't support decoration natively — Scrutor adds it. Don't roll your own with `IServiceProvider.GetService<…>` calls inside the wrapper; that's service-locator and fights the container.
