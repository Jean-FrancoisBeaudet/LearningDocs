# Tuples, Attributes, Regex

_Targets .NET 10 / C# 14. See also: [Generic constraints and variance](./generic-constraints-and-variance.md), [Primary constructors](./primary-constructors.md), [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md), [Reflection](../REFLECTION_AND_DYNAMIC_CODE/reflection-member-discovery.md)._

Three small-but-load-bearing language features bundled here because the plan groups them and they share a theme: metadata and lightweight data shapes the compiler lowers for you.

## Tuples — `ValueTuple` vs `System.Tuple`

Modern tuple syntax (`(int x, string s)`) always lowers to **`System.ValueTuple<...>`**, a family of generic `struct`s. The old **`System.Tuple<...>`** is a reference type from .NET Framework days; don't emit it unless interop forces you.

```csharp
(int id, string name) GetUser() => (42, "Ada");

var u = GetUser();
Console.WriteLine($"{u.id} {u.name}");     // named elements are compile-time only

var (id, name) = GetUser();                 // deconstruction
```

### What the compiler actually stores

Names travel as metadata, not runtime state. The field names on `ValueTuple` are fixed `Item1..Item8`; your names are attached via a synthesized `[TupleElementNames]` attribute on the enclosing member.

```csharp
// signature as seen by reflection:
[return: TupleElementNames(new[] { "id", "name" })]
ValueTuple<int, string> GetUser();
```

Implications:
- Renaming a tuple element on a public API is a **source break** for callers using the names, but **binary-compatible** (the underlying ValueTuple is unchanged).
- Past 7 positional elements the compiler nests: `ValueTuple<T1..T7, TRest>` where `TRest` is another `ValueTuple`. Eight is usually a code smell — use a `record` or `record struct`.

### Equality

`ValueTuple<...>` overrides `Equals` / `GetHashCode` structurally, and since C# 7.3 tuple `==` / `!=` is a per-element lift: `(1, "a") == (1, "a")` is `true`, element-by-element, with the operator chosen per element type. `null` in a `(string, int)?` compares element-by-element too.

### Deconstruction

Any type can be deconstructed by supplying a `Deconstruct` method (instance or extension):

```csharp
public readonly record struct Point(int X, int Y);

// record auto-generates Deconstruct — for plain classes:
public sealed class Address(string city, string zip)
{
    public string City => city;
    public string Zip  => zip;
    public void Deconstruct(out string city, out string zip) => (city, zip) = (City, Zip);
}

var (city, zip) = new Address("Ottawa", "K1A");
```

### When to choose what

| Shape                     | Use                                        |
|---------------------------|--------------------------------------------|
| Return 2–4 values locally | tuple literal                              |
| Dictionary key            | `ValueTuple` — it's a struct, structural equality, cheap hash |
| Part of a public API      | **named `record` or `record struct`** — tuple names are metadata, and readers want a type they can grep for |
| Deep/public domain model  | `record class` with primary-ctor positional pattern |

## Attributes

Attributes are inert metadata attached to declarations. The compiler emits them into the assembly; code (reflection, analyzers, source generators, the runtime itself) reads them later.

```csharp
public sealed class RoleAttribute(string name) : Attribute
{
    public string Name { get; } = name;
    public bool   Default { get; init; }
}

[Role("admin", Default = true)]
public class AdminDashboard { }
```

### Declaration rules

- Derive from `System.Attribute`. The convention is the `Attribute` suffix; you can omit it at use-site (`[Role("x")]`).
- Positional args go to the constructor. Named args (`Default = true`) go to public settable properties / fields.
- Only **constants**, **`typeof`**, and **1-D arrays** of those are legal attribute arguments — anything else is a compile error.
- Seal attribute classes unless you specifically need inheritance; the reflection lookup is faster and intent is clearer.

### Targets and `[AttributeUsage]`

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Property,
                AllowMultiple = false,
                Inherited = true)]
public sealed class AuditAttribute : Attribute { }
```

When a declaration has multiple aspects, specify the target explicitly:

```csharp
public class Foo
{
    [field: NonSerialized]                       // the backing field, not the property
    public EventHandler? Changed;

    [return: MaybeNull]                          // the return value
    public string? TryGet() => null;

    [param: NotNull]                             // the parameter
    public void Set(string s) => _ = s.Length;
}

[module: SkipLocalsInit]                          // module-level
```

### Conditional attributes

`[Conditional("DEBUG")]` on a method causes calls to be **erased** at call sites when the symbol isn't defined. Cheap assertions without needing `#if`.

```csharp
[Conditional("DEBUG")]
public static void Assert(bool cond) { if (!cond) throw new Exception(); }
```

### Reading attributes — reflection vs source-gen

Reflection:
```csharp
var attr = typeof(AdminDashboard).GetCustomAttribute<RoleAttribute>();
```

For hot paths and AOT, prefer **source generators** that discover attributes at compile time and emit static dispatch. Classic examples: `[GeneratedRegex]`, `[JsonSerializable]`, `[LoggerMessage]`, `[InterceptsLocation]`. These avoid reflection, trim/AOT cleanly, and often inline.

Runtime-honored attributes worth knowing:
- `[ModuleInitializer]` — static void method runs once at module load.
- `[MethodImpl(MethodImplOptions.AggressiveInlining)]` — strong hint, not guarantee.
- `[SkipLocalsInit]` — elides `.locals init`, matters for `stackalloc`-heavy code.
- `[CallerMemberName]` / `[CallerArgumentExpression]` — see [CallerArgumentExpression](./caller-argument-expression.md).

## Regex — don't reach for `Regex.IsMatch` anymore

Since .NET 7, `[GeneratedRegex]` produces a compile-time regex with **no runtime compilation**, **no reflection**, AOT-safe, and usually **faster** than both the cached and `Compiled` runtime options.

```csharp
using System.Text.RegularExpressions;

public partial class Email
{
    [GeneratedRegex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$", RegexOptions.IgnoreCase)]
    private static partial Regex Pattern();

    public static bool IsValid(string s) => Pattern().IsMatch(s);
}
```

The generator emits a concrete `Regex` subclass with hand-rolled matching logic. Requires the containing method be `static partial` (or instance partial) with no body.

### The old flavors

| API                                        | When to use                                    |
|--------------------------------------------|------------------------------------------------|
| `Regex.IsMatch(input, pattern)`            | Hides a 15-entry LRU cache keyed by pattern string. Cheap once warm, but a typo produces silent cache misses. Avoid in hot paths — the cache key includes options, and contended lookups block. |
| `new Regex(pattern, RegexOptions.Compiled)`| Emits IL via Reflection.Emit. Fast match, slow startup, **not AOT/trim-safe**. Superseded by `[GeneratedRegex]`. |
| `RegexOptions.NonBacktracking`             | Guarantees linear time in input length, at the cost of some features (no backreferences, no lookaround). Use it to kill ReDoS risk on user-supplied patterns. |

### ReDoS

Catastrophic backtracking on patterns like `^(a+)+$` against `"aaaaaaaaaaaaaaaaaaaaaaaaa!"` can take seconds. Mitigations:
- Set `Regex.MatchTimeout` per instance (or the `REGEX_DEFAULT_MATCH_TIMEOUT` AppDomain data slot for global default).
- Use `RegexOptions.NonBacktracking` for untrusted patterns.
- Validate pattern complexity upstream; never compile patterns you don't own.

### Spans in modern regex

`Regex.EnumerateMatches(ReadOnlySpan<char>)` returns a ref struct enumerator of `ValueMatch` (offset + length) — zero allocations per match. Pair with `[GeneratedRegex]` for tight tokenizers.

```csharp
foreach (var m in Pattern().EnumerateMatches(input))
    Process(input.Slice(m.Index, m.Length));
```

## Senior-level gotchas

- **Tuple element names are metadata only** — renaming breaks callers that use the names but doesn't bump an assembly version. Review CI on public APIs accordingly.
- **Tuples aren't free** — a `(long, long, long, long)` dictionary key copies 32 bytes per lookup. For hot paths, profile vs a bespoke `readonly record struct`.
- **Attribute ctor args are `const`** — you can't pass a `typeof(Foo<int>)` with an open generic argument, can't pass a `DateTime`, can't pass an interpolated string. Newcomers hit this constantly.
- **`Inherited = true` on `[AttributeUsage]` only works through class inheritance, not interfaces.** If you need interface-method attributes to flow, you have to walk `GetInterfaces()` yourself — reflection won't.
- **`Regex.IsMatch(string, string)` silently thrashes the LRU cache** when called with dynamically-built patterns. Wrap in a `[GeneratedRegex]` partial or hold an instance field.
- **`RegexOptions.Compiled` + trimming/AOT = broken assembly at runtime.** Always prefer `[GeneratedRegex]` now; the PM guidance since .NET 7.
- **`NonBacktracking` drops features silently** (no named groups in some positions, no lookahead/lookbehind, no atomic groups). Test the matches, don't assume parity.
- **`Conditional` attribute erases the *call*, not the argument evaluation.** `Debug.Assert(ExpensiveCheck())` **is** erased call-and-all; `if (cond) Log(…)` around it isn't. Matters when the argument has side effects.
- **`record struct` with tuple-like fields** often beats a tuple in readability without losing the structural-equality perf benefit.
