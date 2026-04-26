# Scrutor for Assembly Scanning

_Targets .NET 10 / C# 14. See also: [microsoft-di](./microsoft-di.md), [scrutor-for-decorators](./scrutor-for-decorators.md)._

`Scrutor` (NuGet: `Scrutor`, Kristian Hellang) layers convention-based assembly scanning on top of `IServiceCollection`. It does **not** replace [Microsoft DI](./microsoft-di.md) — it produces the same `ServiceDescriptor` entries you'd write by hand, just driven by reflection over an assembly. Use it the moment you find yourself writing `services.AddScoped<IFooHandler, FooHandler>()` more than five times for the same shape; do **not** use it as a global "register everything" hammer — selective scanning with explicit filters is what keeps the registration list comprehensible.

## Install

```bash
dotnet add package Scrutor
```

No host wiring, no startup config — `IServiceCollection.Scan(...)` becomes available as an extension.

## The minimal scan

```csharp
builder.Services.Scan(scan => scan
    .FromAssemblyOf<OrderHandler>()           // pick the source
    .AddClasses(c => c.AssignableTo<IHandler>())  // pick the classes
    .AsImplementedInterfaces()                // pick what to register them as
    .WithScopedLifetime());                   // pick the lifetime
```

Reads as a sentence: *from the assembly containing `OrderHandler`, add classes assignable to `IHandler`, register each as the interfaces it implements, with scoped lifetime.* Every Scrutor scan is some combination of those four concerns: **source → class filter → "as" projection → lifetime**.

## Source selectors

| Method | Picks |
|---|---|
| `FromAssemblyOf<T>()` | The assembly containing `T` |
| `FromAssembliesOf(typeof(A), typeof(B))` | The assemblies containing the listed types |
| `FromAssemblies(asm1, asm2)` | Explicit `Assembly` instances |
| `FromCallingAssembly()` | The assembly that called the scan extension |
| `FromExecutingAssembly()` | The assembly running the scan code |
| `FromApplicationDependencies()` | **Every** loaded assembly Scrutor can find via the dependency context |
| `FromDependencyContext(DependencyContext.Default)` | Same as above, explicit |

`FromApplicationDependencies` is the trap: it scans Microsoft.*, third-party packages, and anything else the runtime has loaded. The reflection cost balloons, you accidentally register types you didn't write (often with subtly wrong lifetimes), and a transitive package upgrade can silently change your DI graph. Default to `FromAssembliesOf(typeof(MarkerInThisLayer))` — explicit, fast, stable.

## Class selectors

```csharp
.AddClasses(filter =>
    filter
        .AssignableTo<IRequestHandler>()              // implements/inherits this
        .Where(t => t.Name.EndsWith("Handler"))       // free-form predicate
        .InNamespaceOf<OrderHandler>()                // namespace prefix match
        .NotInNamespaceOf<TestDoubles.FakeHandler>()  // exclusion
        .WithAttribute<RegisterAttribute>())          // marker attribute opt-in
```

`AddClasses` defaults to **public, non-abstract, non-static** types. `AddClasses(publicOnly: false)` includes internals (useful when you want to register non-public implementations of a public interface).

For open generics:

```csharp
.AddClasses(c => c.AssignableTo(typeof(IRequestHandler<,>)))
.AsImplementedInterfaces()
.WithScopedLifetime();
```

Scrutor handles open-generic implementations against open-generic interfaces correctly — every closed `RequestHandler<TQuery, TResult>` gets registered against the matching `IRequestHandler<TQuery, TResult>`.

## "As" selectors

| Method | Registers each scanned class as |
|---|---|
| `AsSelf()` | Itself (the concrete type) |
| `AsImplementedInterfaces()` | **Every** interface it implements |
| `AsMatchingInterface()` | The interface whose name matches `I{ClassName}` (`Foo` → `IFoo`) |
| `As<TBase>()` | A specific base/interface (must be assignable) |
| `AsSelfWithInterfaces()` | Both itself and every implemented interface (forwarded — same instance per scope) |

`AsImplementedInterfaces()` is the convenience pick that bites you. If `EmailNotifier` implements `INotifier`, `IDisposable`, and `IEnableLogger`, you've now registered it as **all three**, including `IDisposable` (which is rarely what you want). Prefer `AsMatchingInterface()` or `As<INotifier>()` when you know the contract.

`AsSelfWithInterfaces()` registers two descriptors but uses a forwarding factory so resolving by interface or by concrete type yields the **same instance per scope** — important when you have a class that exposes both an "ergonomic" interface and a "rich" concrete API.

## Lifetime selectors

```csharp
.WithSingletonLifetime()
.WithScopedLifetime()
.WithTransientLifetime()
```

Pick deliberately. Scanning `*Handler` classes as singletons because "they're stateless" only holds until someone adds a scoped dep, at which point you have a captive dependency in production. Default scoped is the safest choice for handlers, repositories, and use-case classes.

## Registration strategy

By default, every Scrutor registration **appends** to `IServiceCollection`. If your scan re-registers an already-registered service, you end up with multiple descriptors — `IEnumerable<T>` resolves all, single-service `GetService<T>()` returns the last one. Control this with:

```csharp
.UsingRegistrationStrategy(RegistrationStrategy.Skip)     // keep existing, skip the new one
.UsingRegistrationStrategy(RegistrationStrategy.Replace())// remove existing, add the new one
.UsingRegistrationStrategy(RegistrationStrategy.Throw)    // fail loudly on duplicate
.UsingRegistrationStrategy(RegistrationStrategy.Append)   // default — add anyway
```

`Replace()` takes a `ReplacementBehavior`: `ServiceType` (default — replace by service type), `ImplementationType` (replace by implementation), or `Both`. Use `Throw` in tests so duplicate registrations fail CI instead of silently changing resolution order in production.

## Realistic patterns

**Register every CQRS handler with one line:**

```csharp
services.Scan(s => s
    .FromAssembliesOf(typeof(GetOrderQuery), typeof(CreateOrderCommand))
    .AddClasses(c => c.AssignableTo(typeof(IRequestHandler<,>)))
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

**Repository convention (`Foo` ↔ `IFoo`):**

```csharp
services.Scan(s => s
    .FromAssemblyOf<UserRepository>()
    .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
        .AsMatchingInterface()
        .WithScopedLifetime());
```

**Marker-attribute opt-in (most explicit):**

```csharp
[AttributeUsage(AttributeTargets.Class)]
public sealed class RegisterScopedAttribute : Attribute;

services.Scan(s => s
    .FromAssemblyOf<Marker>()
    .AddClasses(c => c.WithAttribute<RegisterScopedAttribute>())
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

The attribute pattern keeps registration **opt-in**, which means a new class isn't registered until the author chooses — much safer in a large codebase than "register everything ending in Handler."

## Performance

Scanning runs **once at startup** during `ConfigureServices`. The cost is reflection over the assemblies you select — typically 5–50 ms for a normal app, climbing into hundreds of milliseconds with `FromApplicationDependencies` on a big graph. Cache the assembly list once and pass it explicitly:

```csharp
private static readonly Assembly[] AppAssemblies =
    [typeof(GetOrderQuery).Assembly, typeof(InfrastructureMarker).Assembly];

services.Scan(s => s.FromAssemblies(AppAssemblies)
    .AddClasses(c => c.AssignableTo(typeof(IRequestHandler<,>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

Runtime resolution is unaffected — Scrutor only writes `ServiceDescriptor`s; once `BuildServiceProvider()` runs, the container behaves identically to a hand-written one.

## Senior-level gotchas

- **`FromApplicationDependencies` scans third-party assemblies.** You will accidentally register `Microsoft.Extensions.*` internals and anything else the runtime loaded. Always prefer `FromAssembliesOf(...)` with explicit marker types.
- **`AsImplementedInterfaces` registers `IDisposable`, `IAsyncDisposable`, `ICloneable`** — every interface, system or user. Resolving `IDisposable` from the container then returns *every* registered service that implements it. Use `AsMatchingInterface` or `As<T>()` for contracts you actually care about.
- **`AddClasses` skips abstract and non-public by default.** A class marked `internal` won't be picked up unless you pass `publicOnly: false`. Check this before "but my class isn't getting registered."
- **Append is the default.** Calling `Scan` twice on overlapping assemblies adds duplicate descriptors. `IEnumerable<T>` resolutions get duplicates, single-service `GetService<T>` returns the last one written. Use `RegistrationStrategy.Skip` or `.Throw` to make this safe.
- **`AsMatchingInterface` requires the interface to actually exist.** If `Foo` doesn't implement `IFoo`, the scan skips it silently — no error, just a missing registration that surfaces as a `null` deep in the call graph.
- **Open-generic registration via scanning bypasses MEDI's open-generic facility.** Scrutor produces one closed registration per concrete `Handler<TQuery, TResult>` it finds. That works, but you can't add an open-generic registration *and* scan closed ones for the same interface — the scan-produced closed entries shadow the open one for the closed types they match.
- **Scrutor uses reflection.** Trimming and NativeAOT will strip types Scrutor wants to enumerate. Either annotate marker assemblies with `[DynamicallyAccessedMembers]` on the constraint side, or skip Scrutor entirely for AOT and use source-generated registration helpers (e.g. MediatR source generator).
- **Scope validation still applies.** Scanning a singleton lifetime onto a class that injects a scoped service produces the same captive-dependency error you'd get from manual `AddSingleton<T>()`. Scrutor doesn't change MEDI's lifetime rules — it just generates the registrations.
- **`InNamespaceOf<T>`** matches **prefix**, not equality — `InNamespaceOf<OrderHandler>()` picks up `MyApp.Orders` *and* `MyApp.Orders.Internal`. Use `Where(t => t.Namespace == "...")` for exact-namespace matching.
- **The scan list at startup is logically ordered, but resolution is not.** Don't assume "the order I scanned is the order `IEnumerable<T>` returns" — for safety, sort consumers explicitly (`OrderBy(h => h.GetType().Name)`) or apply explicit ordering metadata if order matters.
