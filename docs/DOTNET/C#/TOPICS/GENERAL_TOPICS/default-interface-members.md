# Default interface members (DIM)

_Targets .NET 10 / C# 14. See also: [Interfaces, abstract classes, sealed](../OOP/interfaces-abstract-sealed-anonymous.md), [Extension members](./extension-members.md), [Generic constraints and variance](./generic-constraints-and-variance.md)._

Default interface members — added in C# 8 / .NET Core 3.0 — let an interface ship **a body** for a method, property, indexer, or event. The driving scenario was API evolution: a library adds a new member to a public interface without breaking every existing implementer. They are **not** a substitute for traits, mix-ins, or abstract classes, and reaching for them by default is usually a sign of a shaky design.

## The shape

```csharp
public interface ICache<TKey, TValue> where TKey : notnull
{
    TValue? Get(TKey key);
    void    Set(TKey key, TValue value);

    // Default — implementers get it for free:
    bool TryGet(TKey key, out TValue? value)
    {
        value = Get(key);
        return value is not null;
    }
}
```

An existing `ICache<,>` implementer that never heard of `TryGet` still compiles and still type-checks; callers can call `TryGet` on any `ICache` reference.

## How it's resolved

A default member is an **interface method with a body** in IL — a new flavor of `.method virtual` on the interface type. Calls dispatch through the interface vtable:

```csharp
ICache<string, User> cache = new MemoryCache();
cache.TryGet("k", out var u);   // dispatches via the interface slot
```

Key rule: default members are reachable **only through the interface type**. You cannot call them through the implementing class unless the class overrides them or casts `this` to the interface.

```csharp
public sealed class MemoryCache : ICache<string, User>
{
    public User? Get(string key) => ...;
    public void Set(string key, User v) => ...;
    // TryGet is implicitly provided by the interface.
}

var mc = new MemoryCache();
// mc.TryGet(...);        // CS0122 / CS1061 — not visible on MemoryCache
((ICache<string, User>)mc).TryGet("k", out _);  // OK
```

That asymmetry is deliberate: DIM is an **interface** feature, not a class-inheritance feature.

## Overriding a default

An implementer overrides a default by providing an explicit implementation of the same signature:

```csharp
public sealed class LoggingCache : ICache<string, User>
{
    public User? Get(string key) => ...;
    public void Set(string key, User v) => ...;

    // Override the default:
    public bool TryGet(string key, out User? value)
    {
        value = Get(key);
        // extra logging...
        return value is not null;
    }
}
```

Either a normal implementation (`public bool TryGet(...)`) or an explicit interface implementation (`bool ICache<string, User>.TryGet(...)`) satisfies the override. With an explicit implementation, the method is still reachable only via the interface.

## Diamond resolution

If a type implements two interfaces that both provide a default for the same method, the compiler forces you to pick:

```csharp
public interface IA { void Greet() => Console.WriteLine("A"); }
public interface IB { void Greet() => Console.WriteLine("B"); }

public class C : IA, IB { }
// CS8705: 'C' does not implement 'IA.Greet' because 'IA.Greet' is not the most specific member.
```

Resolution: provide an explicit implementation in `C`, or have one interface re-declare the member as abstract and override it:

```csharp
public class C : IA, IB
{
    void IA.Greet() => Console.WriteLine("A");
    void IB.Greet() => Console.WriteLine("B");
}
```

The compiler only picks a default automatically when **one interface is strictly more derived** than the others — then the most-derived default wins.

## `sealed`, `abstract`, and visibility

Default members support the full set of interface modifiers:

- `public` (default).
- `private` — a helper method only callable by other default members in the same interface. Useful for refactoring.
- `protected` — callable only from derived interfaces / implementers.
- `internal` — visible inside the assembly.
- `abstract` — the default for a member with no body (the classic interface method). Can be used explicitly to signal intent.
- `sealed` — cannot be overridden by implementers. Pair with a body for "this is the canonical behavior, don't touch."
- `virtual` — implicit on interface members with a body; rarely written explicitly.
- `static` — see below.

```csharp
public interface IRateLimiter
{
    // Private helper — only usable from other default members in this interface.
    private static TimeSpan Clamp(TimeSpan t) => t < TimeSpan.Zero ? TimeSpan.Zero : t;

    TimeSpan Window { get; }

    // Sealed default: implementers cannot override.
    sealed TimeSpan SafeWindow() => Clamp(Window);
}
```

## `static abstract` / `static virtual` members (C# 11)

The most far-reaching DIM feature: `static abstract` interface members. These power **generic math** and the `INumber<T>`, `IParsable<T>`, `IAdditionOperators<T,T,T>` family.

```csharp
public interface IParsable<TSelf> where TSelf : IParsable<TSelf>
{
    static abstract TSelf Parse(string s, IFormatProvider? provider);
    static abstract bool  TryParse(string? s, IFormatProvider? provider, out TSelf result);
}

public static T ParseAll<T>(IEnumerable<string> src) where T : IParsable<T>
    => src.Aggregate(default(T)!, (_, s) => T.Parse(s, null));
```

The key enabler is the **curiously-recurring `TSelf` constraint**: the interface references "the implementing type" as a generic parameter, letting operators and factories return concrete values rather than interface references.

`static virtual` (with a body) provides a default:

```csharp
public interface IShape<TSelf> where TSelf : IShape<TSelf>
{
    static abstract TSelf Origin { get; }
    static virtual  TSelf Default => TSelf.Origin;
}
```

Implementers get `Default` for free by implementing `Origin`.

## When to use DIM

Good fits:

- **API evolution** — adding a convenience overload or a derived property to a shipped interface without breaking implementers.
- **Mixins of stateless behavior** — e.g., `ToString`-like helpers built on primitive methods.
- **Generic math / abstraction over static members** — the `static abstract` case; no alternative in C# for this.
- **Private helpers for interface authors** — `private static` helpers keep interface code DRY without polluting the type space.

Bad fits:

- **State-bearing "mixins".** Default members cannot touch fields (interfaces have no instance state). Every "helper" is implemented in terms of abstract members — fine for small deltas, painful for anything rich.
- **Replacing abstract base classes.** Diamond problems, limited access modifiers, and the "only via interface" call-site asymmetry mean you end up reinventing half of class inheritance badly.
- **"Just adding a method without breaking consumers."** If consumers are also implementers, the default member can silently drift from their contract expectations. Consider an additive, opt-in extension method or a new interface instead.

## DIM vs extension methods vs abstract classes

| Need                                                     | Reach for                     |
|----------------------------------------------------------|-------------------------------|
| Backport new logic onto all implementers of an interface | Default interface member      |
| Add convenience that doesn't need to be virtual          | Extension method              |
| Share fields, constructor logic, or multi-step lifecycle | Abstract class                |
| Abstract over `static` members (math, parsing)           | `static abstract` interface member (no alternative) |

## Senior-level gotchas

- **Default members are unreachable through the concrete class.** If `MemoryCache : ICache<…>` and `TryGet` is a default, `memoryCache.TryGet(...)` fails to compile. You must call through the interface reference or override on the class.
- **Runtime requires .NET Core 3.0+.** DIM relies on CLR support for interface methods with bodies; .NET Framework cannot consume DIM-bearing assemblies at all. Analyzers warn when targeting `netstandard2.0` with DIM present.
- **`virtual` + `new` interactions** in hierarchies of interfaces can produce surprising dispatch. Prefer `sealed` on canonical defaults and force implementers to opt out with explicit re-declaration.
- **Diamond resolution rule is "most derived interface wins, else ambiguous"** — adding a second interface to a type can silently change which default wins if one extends the other. Re-test.
- **Adding a default to a shipped interface is source-compatible, not always binary-compatible.** If an implementer was compiled against the older interface and separately references the new one, JIT'd code is fine; but Roslyn source generators targeting the old shape may fail.
- **`static abstract` interface members require generic type constraints** to be usable (`where T : ISomething<T>`). Without the constraint, you can't call `T.Parse(...)`. This cascades into generic method signatures.
- **Default members cannot declare fields.** They can declare `static` fields on the interface, but instance state lives on the implementer. "Stateful mixin" is not a thing.
- **Access modifiers require C# 8+ language version and matching CLR**. Attempting `private` members on an interface while targeting older TFMs raises CS8703.
- **`sealed` on a default method prevents overriding**, but reflection / `MethodInfo.Invoke` can still reach it. `sealed` is a language constraint, not a CLR one for default members.
- **Event defaults are legal but awkward** — the interface has no storage for the delegate, so the default must accept backing-field access from an abstract member. Almost always clearer as abstract + helper method.
- **Tooling** (designers, mocking libraries, some older source generators) may not recognize DIM-only members. Moq/NSubstitute handle them in current versions; older custom proxies likely do not.
