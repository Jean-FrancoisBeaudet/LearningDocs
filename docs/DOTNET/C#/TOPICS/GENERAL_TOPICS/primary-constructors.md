# Primary constructors

_Targets .NET 10 / C# 14. See also: [Classes, records, structs](../OOP/classes-records-structs.md), [Constructors, properties, fields, methods](../OOP/constructors-properties-fields-methods.md), [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md)._

Primary constructors let you declare constructor parameters directly in the type header. `record`s had them since C# 9; C# 12 extended the feature to plain `class` and `struct`. They change the **compile-time** shape of the type but not its runtime cost — the compiler still emits a conventional ctor and fields.

## The shape

```csharp
// Class with a primary constructor:
public sealed class OrderService(IOrderRepo repo, ILogger<OrderService> log)
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        log.LogInformation("Loading order {Id}", id);
        return await repo.FindAsync(id, ct);
    }
}

// Struct with a primary constructor:
public readonly struct Range2D(int x, int y, int w, int h)
{
    public int Area => w * h;
}
```

Both `repo`/`log` and `x`/`y`/`w`/`h` are **parameters in scope for the entire type body**. The compiler captures each parameter as a hidden private field only if it is *actually used* outside the constructor initialization phase.

## How it differs from `record` primary ctors

| Feature                   | `class` / `struct` primary ctor | `record` primary ctor                |
|---------------------------|---------------------------------|--------------------------------------|
| Auto-properties?          | No                              | Yes (one per parameter, `{ get; init; }`) |
| `with`-expression support | No                              | Yes                                  |
| Structural equality?      | No                              | Yes (`Equals`/`GetHashCode`/`==`)    |
| `Deconstruct`?            | No                              | Yes                                  |
| `ToString` with members?  | No                              | Yes                                  |
| Multiple primary ctors?   | No (one per type)               | No                                   |

Put another way: `record` is the "full metadata package"; `class`/`struct` primary ctors are just **parameter scope sugar** — you get one ctor wired up, nothing else.

## Parameter capture — the one concept that trips people

A primary-ctor parameter is **not** a field, **not** a property. It's a parameter whose lifetime is extended to the instance if it's used outside ctor initialization.

```csharp
public sealed class Counter(int initial)
{
    private int _count = initial;         // used at init time only — no capture
    public int Current => _count + offset; // 'offset' would be captured if declared here
    private readonly int offset = initial; // used during init — no capture here either

    public int Initial() => initial;       // use outside init — compiler emits a private field for 'initial'
}
```

- If a parameter is used only to initialize fields/properties, the compiler **doesn't** keep it around — no field is emitted.
- If it's used in a method, property body, or lambda, the compiler emits a hidden `private int <initial>P;` field and routes all uses through it.
- The captured field's name is mangled, so reflection doesn't see `initial`.

### You cannot `readonly`-mark the parameter

There's no way to say `readonly int initial` in the header. If you need immutability guarantees, assign to a `private readonly` field explicitly:

```csharp
public sealed class Counter(int initial)
{
    private readonly int _initial = initial;  // explicit field; analyzer won't nag
    public int Initial => _initial;
}
```

This also sidesteps the (legitimate) complaint that primary-ctor captures look mutable from inside the type body — because they are.

### Mutation warnings

If a primary-ctor parameter is mutated inside the type, the compiler warns (`CS9124` / analyzer `IDE0290`):

```csharp
public sealed class Svc(ILogger log)
{
    public void DoWork()
    {
        log = NullLogger.Instance;   // CS9124: Parameter 'log' should not be assigned to
    }
}
```

This is telling you: "You probably meant a field." Take the hint.

## Where primary constructors shine — DI

The sweet spot: a **sealed class** taking dependencies from DI that it only ever *uses*, never reassigns.

```csharp
public sealed class CheckoutHandler(
    IOrderRepo repo,
    IPaymentGateway payments,
    ILogger<CheckoutHandler> log)
    : IRequestHandler<Checkout, CheckoutResult>
{
    public async Task<CheckoutResult> Handle(Checkout cmd, CancellationToken ct)
    {
        var order = await repo.FindAsync(cmd.OrderId, ct)
                    ?? throw new NotFoundException(cmd.OrderId);

        var tx = await payments.ChargeAsync(order, ct);
        log.LogInformation("Charged order {Id} tx {Tx}", order.Id, tx.Id);
        return new(tx.Id);
    }
}
```

No `private readonly` boilerplate per dependency, no explicit ctor. A 20-line service becomes 15.

## Where primary constructors lose

### Public API shapes

If `class Money(decimal amount, string currency)` means callers expect `m.Amount` and `m.Currency` to exist, you're setting up confusion. `class` primary ctors **don't** synthesize properties.

```csharp
public sealed class Money(decimal amount, string currency)
{
    // Readers of this code cannot do m.Amount or m.Currency — those properties don't exist.
    // They can do 'new Money(10, "CAD")' and nothing else.
}
```

If you want auto-properties + ctor-as-shape, use `record` or `record struct`. If you want plain data with explicit properties, write them out.

### Inheritance and protected access

A primary-ctor parameter is private-captured. Subclasses can't see it:

```csharp
public class Base(ILogger log) { public void A() => log.LogInformation("A"); }
public sealed class Derived(ILogger log) : Base(log) { public void B() => log.LogInformation("B"); }
// 'log' is captured TWICE — once in Base, once in Derived.
```

Both base and derived capture the same argument into **separate** private fields. For a shared dependency you'd want visible to subclasses, declare a `protected readonly` field explicitly and assign it from the primary ctor parameter.

### Field initializers and ordering

Primary-ctor params run **before** field initializers and body ctor code, same as regular ctor params:

```csharp
public sealed class Foo(int seed)
{
    private readonly int _a = seed * 2;     // seed is available here
    private readonly int _b = _a + seed;    // _a initialized above, seed still in scope
}
```

But multiple ctors (overloads) must chain to the primary via `: this(...)`:

```csharp
public sealed class Foo(int seed)
{
    public Foo() : this(0) { }              // ok
    public Foo(string s) : this(int.Parse(s)) { }
}
```

### `readonly struct` interactions

```csharp
public readonly struct Point(int x, int y)
{
    public int X => x;                       // fine — read-only capture
    public int Y => y;
    // public void Set(int nx) { x = nx; }   // CS8341: cannot assign in readonly struct
}
```

`readonly struct` + primary ctor is a clean way to declare a small value type without writing `public int X { get; } public Point(int x) => X = x;` boilerplate — at the cost of losing the `X` property (it has to be written).

## Senior-level gotchas

- **Primary-ctor parameters are mutable inside the body.** The compiler warns, but no `readonly` modifier exists. If immutability matters (concurrency, defensive copies), assign to an explicit `private readonly` field.
- **No auto-properties on `class`/`struct` primary ctors.** `new User(id, name).name` does **not** compile from outside the class — the name is a captured field, not a property. Writers expecting record-like ergonomics get surprised.
- **Double capture through inheritance.** `class Derived(ILogger log) : Base(log)` stores `log` in Base's private capture field AND Derived's. Each base class captures independently.
- **Can't chain to a secondary ctor without going through the primary.** Every secondary ctor must `: this(…)`. Awkward when the primary is the "expensive" ctor — invert the design or skip the primary.
- **`= parameter` field initializers with side effects** run once per ctor invocation. Moving side effects from a body ctor into a primary-ctor capture changes observability.
- **Nullable annotations on primary-ctor parameters** affect the *captured field* too — `class Svc(ILogger? log)` means nullable checks flow into the body. Mixing `?` / non-`?` on the primary vs secondary ctors produces subtle CS warnings.
- **Debugger and source gen tools** sometimes struggle with the mangled capture field names. If you're writing a Roslyn generator that walks primary-ctor types, expect to handle `IMethodSymbol.Parameters` rather than `IFieldSymbol` for the captures.
- **Analyzer `IDE0290` ("use primary constructor")** will aggressively rewrite your ctor-assignment pairs. Disable it on a team that's not on board — the non-sealed or API-shaped classes don't want this.
- **Don't mix primary ctor with a `static` factory pattern** unless the primary is private. Exposing `new Foo(dep)` alongside `Foo.Create(config)` splits construction across two surfaces.
- **`record` inherits differently**: `record Derived(int X) : Base(X)` forwards AND synthesizes `X` as a property. Mixing `record` and `class` in a hierarchy with primary ctors is a confusion hotspot.
