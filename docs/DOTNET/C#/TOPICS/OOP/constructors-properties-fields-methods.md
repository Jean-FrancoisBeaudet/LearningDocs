# Constructors, properties, fields, methods

_Targets .NET 10 / C# 14. See also: [Classes, records, structs](./classes-records-structs.md), [Encapsulation, inheritance, polymorphism](./encapsulation-inheritance-polymorphism.md), [Static and generics](./static-and-generics.md), [Primary constructors](../GENERAL_TOPICS/primary-constructors.md), [Volatile and Interlocked](../THREAD_SYNCHRONIZATION/volatile-and-interlocked.md), [Async/await](../CONSTRUCTS/async-await.md)._

The four building blocks of every type. Most of the surprises live not in *what* they do but in **when** they run, **how** the compiler lowers them, and **which analyzer** flags you're now violating. This note covers the modern (C# 12–14) shape of each.

## Constructors

### Instance constructors

```csharp
public sealed class Order
{
    public Guid Id { get; }
    public decimal Total { get; }

    public Order(Guid id, decimal total)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(total);
        (Id, Total) = (id, total);
    }

    // Chaining within the same type:
    public Order(decimal total) : this(Guid.NewGuid(), total) { }
}
```

Order of execution when `new Derived()` runs:
1. Field initialisers in `Derived` (top-down).
2. `Derived`'s ctor body — but **first** the chained `: base(...)` runs.
3. Field initialisers in `Base` run before `Base`'s ctor body.

So a virtual call from a base ctor reaches the *derived* override before the derived ctor has run — a well-known footgun. Don't call virtuals from constructors. CA2214 flags it.

### Static constructors

```csharp
public static class Cache
{
    private static readonly Dictionary<int, string> _data;

    static Cache()
    {
        _data = LoadFromDisk();
    }
}
```

- Runs **once** per closed generic type per AppDomain, on the first access (member call, field read, type ref through reflection).
- Thread-safe by the runtime — the CLR holds a type-init lock; other threads block until it completes.
- Cannot have parameters or access modifiers.
- If it throws, the type becomes permanently unusable in that AppDomain (a `TypeInitializationException` wraps the original exception on every subsequent access).
- `beforefieldinit` (the IL flag, set when there's no explicit `static` ctor) tells the runtime it's free to run static field initialisers *lazily*, before any static member access. Adding an explicit `static` ctor removes the flag and forces initialisation eagerly on first access — sometimes a perf regression.

### Copy constructors

Compiler-generated for `record class` (used by `with`):

```csharp
public record class Foo(int X)
{
    // Override to add validation/cloning of mutable fields:
    protected Foo(Foo original)
    {
        X = original.X;
        // deep-copy mutable members here
    }
}
```

User-defined copy ctors on plain `class`es exist but aren't called automatically — you must invoke them.

### Primary constructors

Covered in depth in [primary-constructors.md](../GENERAL_TOPICS/primary-constructors.md). Two-line summary: parameters become in-scope for the entire type body; the compiler captures them as private fields *only if used outside initialisation*; analyzer `IDE0290` will rewrite traditional ctors into primary form.

### `required` members and `SetsRequiredMembers`

```csharp
public class User
{
    public required string Email { get; init; }
    public required Guid Id { get; init; }

    // A ctor that fully initializes all required members can opt out:
    [SetsRequiredMembers]
    public User(Guid id, string email) => (Id, Email) = (id, email);
}

var u1 = new User { Id = Guid.NewGuid(), Email = "x" };  // ok
var u2 = new User(Guid.NewGuid(), "x");                  // ok — attribute opts out
var u3 = new User();                                     // CS9035: 'Email', 'Id' required
```

`required` is the modern alternative to repeating `if (x is null) throw …` in a ctor — the compiler enforces it at the call site. Combine with `init` for immutable construction.

### Object and collection initializers

```csharp
var u = new User
{
    Email = "ada@x.com",
    Id = Guid.NewGuid()
};

var list = new List<int> { 1, 2, 3 };          // calls Add 3×
var dict = new Dictionary<string, int> { ["a"] = 1 };  // indexer init

// Target-typed new (C# 9+):
List<int> nums = new() { 1, 2, 3 };
```

Initializers run **after** the chosen constructor — order matters when the ctor has side effects.

## Properties

### Auto-properties and the synthesised backing field

```csharp
public string Name { get; set; }    // synthesises private field <Name>k__BackingField
```

The backing-field name is mangled and unspeakable in C# — but visible to `Newtonsoft.Json`'s private-field discovery and to reflection.

### `init`-only setters

```csharp
public string Email { get; init; }
```

Settable only during object initialisation (the ctor or the `{ … }` initializer). After construction, the property is effectively read-only. Backed by an `IsExternalInit` modreq the compiler emits — older runtimes can't see it, which is why it's only safe on `net5+` (or with a polyfill type).

### The `field` keyword (C# 13)

```csharp
public string Name
{
    get => field;
    set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
}
```

`field` references the compiler-synthesised backing field without you having to declare `private string _name`. Lets you add validation/normalisation without losing auto-property ergonomics. Mind the breaking change: a class with an existing field literally named `field` now needs `@field` to disambiguate.

### Computed and expression-bodied

```csharp
public decimal Subtotal => Quantity * UnitPrice;
public string Display => $"{Name} ({Email})";
```

No backing field; recomputed every read. Don't use for expensive calculations on hot paths — cache to a field or use `Lazy<T>`.

### `required` properties

See above. `required` + `init` is the modern "constructor-less DTO" pattern.

### Ref-returning properties and indexers

```csharp
public ref int this[int i] => ref _buffer[i];
```

Returns a managed reference to the storage. Callers can read and write through it: `obj[3] = 7;`. Combines with `Span<T>` to give zero-copy access to interior buffers. Requires the *containing* storage to outlive the reference — the C# 11 ref-safety rules enforce this.

### Indexers

```csharp
public sealed class Bag
{
    private readonly Dictionary<string, int> _items = new();

    public int this[string key]
    {
        get => _items.TryGetValue(key, out var v) ? v : 0;
        set => _items[key] = value;
    }
}
```

Indexers can be overloaded by parameter list, named with `IndexerName("Item")` to surface a different name to other languages, and made `init`-only.

## Fields

### `readonly`

Assignable only in the declaration or in a constructor of the same class. Doesn't make the *referent* immutable — a `readonly List<int>` can still be `.Add`-ed to. Use immutable types or `IReadOnlyList<T>` if that matters.

### `const`

Compile-time literal. Inlined into every consumer's IL — changing a `public const` is a binary breaking change. Use `static readonly` for values that may evolve.

```csharp
public const int Version = 3;        // baked into every consumer DLL
public static readonly int Build = 9; // resolved at runtime — safe to change
```

### `static`

One slot per closed generic type. Initialised by the static ctor (or the `beforefieldinit` lazy path).

### `volatile`

A weak fence — every read becomes acquire, every write becomes release. **Doesn't** give you atomicity for 64-bit values on 32-bit platforms, doesn't help with read-modify-write (`++`). Reach for `Interlocked` or `lock` for those — see [Volatile and Interlocked](../THREAD_SYNCHRONIZATION/volatile-and-interlocked.md).

## Methods

### Instance vs static

A static method has no `this`; a non-`virtual` instance call still passes `this` as a hidden first argument. The JIT can sometimes inline static methods more aggressively.

### Extension methods

```csharp
public static class StringExt
{
    public static bool IsEmail(this string s) => s.Contains('@');
}

"x@y".IsEmail();
```

Compiled as a regular static method; the call site is sugar — the compiler picks the right `using`-imported static class. `null this` is allowed (`((string?)null).IsEmail()` calls the method with `s == null`); always null-check.

C# 14 introduces **extension members** (extension properties and static extension methods, declared inside `extension` blocks) — see [Extension members](../GENERAL_TOPICS/extension-members.md).

### Local functions

```csharp
public int Compute(int seed)
{
    return Loop(seed, 0);

    static int Loop(int seed, int acc) =>
        seed == 0 ? acc : Loop(seed - 1, acc + seed);
}
```

`static` local functions cannot capture enclosing locals — and that's the *point*: no closure allocation. Prefer `static` local functions over lambdas for callbacks that don't need closure state.

### Partial methods and partial properties

```csharp
public partial class Generated
{
    public partial string Render();
}

public partial class Generated
{
    public partial string Render() => "<html/>";
}
```

C# 13 extended `partial` to properties and indexers — used heavily by source generators that need to declare a member shape and let the user supply the body (or vice versa). The signature must match exactly across the parts; nullability annotations included.

### Expression-bodied members

Apply to ctors, methods, properties, indexers, and operators. Pure syntactic sugar — same IL.

### `params` collections (C# 13)

```csharp
public void Log(string format, params ReadOnlySpan<object?> args) { … }

Log("x={0}, y={1}", 1, 2);
```

`params` now accepts any collection type recognised by collection-expression rules (`Span<T>`, `ReadOnlySpan<T>`, `IEnumerable<T>`, `List<T>`, etc.) — no longer just arrays. The `ReadOnlySpan` overload avoids the per-call `object[]` allocation that legacy `params object[]` causes.

### Optional arguments

```csharp
public void Send(string body, int retries = 3, CancellationToken ct = default) { … }
```

Default values are baked into the **caller's** IL, same hazard as `const`: changing a default in a library is a binary break. Prefer overloads when the default lives in the public API.

### `ref`, `in`, `out`

```csharp
public bool TryParse(string s, out int value) { … }
public void Add(in BigStruct x) { … }
public ref int Get(int i) => ref _buffer[i];
```

| Modifier | Direction | `this` semantics inside | Caller must initialise? |
|----------|-----------|------------------------|-------------------------|
| `ref`    | In + out  | Mutable                | Yes                     |
| `in`     | In only   | Treated as `readonly` (defensive copy on a non-`readonly struct`!) | Yes |
| `out`    | Out only  | Must be assigned before any read | No (compiler treats as unassigned) |
| `ref readonly` (C# 12) | In only | Same as `in` but caller-side syntax `ref` | Yes |

Pass big `readonly struct` values by `in` for performance; with a non-`readonly struct` you get defensive copies and lose the win. CA1819 / IDE0250 surface these.

### Deconstructors

```csharp
public sealed class Pair(int x, int y)
{
    public int X => x;
    public int Y => y;
    public void Deconstruct(out int x_, out int y_) => (x_, y_) = (X, Y);
}

var (a, b) = new Pair(1, 2);
```

Records auto-generate `Deconstruct`. For non-records, write it manually — useful for tuples-pattern-matching against domain types.

## Senior-level gotchas

- **Calling a virtual method from a base constructor** reaches the derived override before the derived ctor body has run, so the override sees an only-partially-constructed object. CA2214. Initialize via template-method patterns or explicit `Init()` calls *after* construction.
- **`static readonly` of a mutable type** (e.g. `static readonly List<int> Defaults`) is a thread-safety hazard — readers and writers compete on the *contents*, not the reference. Use `ImmutableList<T>` or an exposed `IReadOnlyList<T>`.
- **`const` of a public reference value** is restricted to `string` and `null`; otherwise the compiler refuses. The deeper rule: only types with literal forms can be `const`.
- **Adding a `static` ctor** can silently regress startup performance because it removes `beforefieldinit` and forces eager type init on first member access. Audit your perf-critical static classes for accidental static ctors.
- **`init` setters are mutable inside the ctor body** — if you want hard immutability, store to a `private readonly` backing field and expose `=> _field;`.
- **`required` is enforced only at the C# language level**, not by reflection. `Activator.CreateInstance` followed by reflection-based hydration (legacy serializers) won't throw if a required property is missing. Use a deserializer that respects `[SetsRequiredMembers]` annotations or migrate to `System.Text.Json` + the source generator.
- **`params ReadOnlySpan<T>`** wins over `params T[]` only if the call site doesn't already have an array — the compiler can stackalloc the temp. With a pre-existing array, both forms are identical.
- **`ref` returns + a mutable `List<T>`**: there is no `ref` indexer on `List<T>`. Use `CollectionsMarshal.AsSpan(list)[i]` (returns `ref T`) when you need to mutate value-type elements in place without a copy.
- **Defensive copies on `in T`** — passing a non-`readonly struct` by `in` makes every member access copy the struct. Always pair `in` with `readonly struct` (or measure with BenchmarkDotNet — the analyzer IDE0250 catches it).
- **Optional argument values are call-site-baked** — changing a default in a NuGet library doesn't propagate until consumers recompile. Treat default values as part of the binary contract.
- **`partial` cross-file initialisation order** is undefined for *static* fields between partial declarations of the same type. The C# spec says "in some order"; relying on a specific order (e.g. partial A defines `_x`, partial B's static ctor reads `_x`) is fragile.
- **Static ctor exceptions are sticky** — once `TypeInitializationException` fires, every subsequent access to *any* member of the type throws the same wrapped exception. Don't do I/O in static ctors; use `Lazy<T>` and let the caller handle failure.
- **`field` keyword shadows existing identifiers**. A class with `private object field;` gets the auto-property's synthesised backing field instead. `@field` to disambiguate, but the cleaner fix is renaming the field.
- **Extension method resolution is `using`-scoped**, not type-scoped. A `string.IsEmail()` call binds to whichever extension class is in scope — a downstream `using` import can silently switch which implementation runs.
- **`out` parameters in async methods are forbidden** (`async` methods can only return `Task` / `Task<T>` / `ValueTask` / async iterators). Returning a tuple is the modern equivalent.
