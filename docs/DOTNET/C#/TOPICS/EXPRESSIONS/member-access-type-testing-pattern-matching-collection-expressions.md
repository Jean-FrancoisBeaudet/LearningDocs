# Member access, type testing, pattern matching, collection expressions

_Targets .NET 10 / C# 14. See also: [Operators](../CONSTRUCTS/operators.md), [if, switch, switch expression](../CONSTRUCTS/if-switch-and-switch-expression.md), [Classes, records, structs](../OOP/classes-records-structs.md), [Value types](../TYPES/value-types.md), [Reference types](../TYPES/reference-types.md), [Array, List, Dictionary, HashSet, Stack, Queue](../COLLECTIONS/array-list-dictionary-hashset-stack-queue.md), [Interpolated string handlers](../GENERAL_TOPICS/interpolated-string-handlers.md)._

C#'s expression surface grew fast. Member access has four flavors (`.`, `?.`, `[]`, `?[]`), type testing has three (`is`, `is not`, `as`), and patterns + collection expressions are effectively a sub-language with their own grammar and lowering. This note focuses on the grammar, the *lowering*, and the traps.

## Member access — `.` `?.` `!.` `[]` `?[]`

```csharp
var name   = user.Name;                 // plain member access
var len    = user?.Name?.Length;        // null-conditional chain — result is string's Length or null
var first  = list?[0];                  // null-conditional indexer
var forced = user!.Name;                // null-forgiving — compile-time only, no runtime check
```

Key semantics:
- `?.` short-circuits the **entire rest** of the chain: if `user` is null, `Name`, `Length`, and anything chained after that evaluates to `null` without touching the intermediate operands.
- The *whole* expression takes type `T?` (nullable). For value-typed members, that means lifting: `user?.BirthDate.Year` has type `int?`, not `int`, even though `Year` itself is `int`.
- `!.` is the **null-forgiving** operator. It silences nullable-reference-type warnings. It does **not** emit a runtime null check — `x!.Foo` is just `x.Foo` at runtime.
- `?[]` is the indexer analog: `array?[0]` returns `null` if `array` is null; otherwise evaluates `array[0]`.

Lowering of `a?.b?.c` is (roughly):

```csharp
var __t = a;
(__t is null) ? null : (
    var __u = __t.b;
    (__u is null) ? null : __u.c
);
```

Each `?.` introduces a temporary and a null check. This matters for performance (each `?.` reads the operand once — no double evaluation) and for thread safety (the read is stable after the first temp).

## Indexing, `Index` (`^`), and `Range` (`..`)

```csharp
int[] xs = [1, 2, 3, 4, 5];
xs[0];          // 1
xs[^1];         // 5  — index-from-end
xs[^2..];       // [4, 5]
xs[1..3];       // [2, 3]
xs[1..^1];      // [2, 3, 4]
xs[..];         // whole copy
```

Under the hood:
- `^n` is `System.Index.FromEnd(n)`.
- `a..b` is `System.Range.Create(Index a, Index b)`.
- Arrays, `string`, `Span<T>`, `ReadOnlySpan<T>`, and `List<T>` all support it directly. Everything else participates via the **pattern**: a type with an `int Length`/`Count` property + an indexer of type `int` + (optionally) a `Slice(int, int)` method gets `Index`/`Range` support automatically.

```csharp
public sealed class Ring<T>
{
    private readonly T[] _buf;
    public int Count => _buf.Length;
    public T this[int i] => _buf[i];
    public Ring<T> Slice(int start, int length) => new(_buf.AsSpan(start, length).ToArray());
    public Ring(T[] b) => _buf = b;
}

Ring<int> r = new([10, 20, 30, 40]);
var tail = r[1..^0];     // Slice(1, Count-1) — the compiler translates
```

Fast path for `List<T>` ranges without allocation: `CollectionsMarshal.AsSpan(list)[..k]` gives a `Span<T>` over the underlying array — no copy.

Note: `foreach` uses the same duck-typed pattern (`GetEnumerator` + `Current` + `MoveNext`), and it doesn't care whether your type implements `IEnumerable`.

## Type testing — `is`, `is not`, `as`, `(T)x`

```csharp
object o = "hi";

o is string;                         // True
o is string s;                       // True, binds s
o is not null;                       // True — canonical null check
o as string;                         // "hi" — safe cast, null on mismatch
(string)o;                           // "hi" — throws InvalidCastException on mismatch
```

Rules:
- **`is T`** — runtime type check. On reference types, **never** matches null (even for nullable references — `(string?)null is string` is `false`).
- **`is T x`** — *declaration pattern*: same check plus binding. The binding is in scope in the `true` branch, and may be in scope on the fall-through path if the compiler can prove definite assignment (the `if (o is not string s) return;` idiom).
- **`is not null`** — the canonical null check. Bypasses user-defined `operator ==` (important: records and many value types overload it). Analyzer `CA1508` nudges you toward it.
- **`as`** — safe cast: returns null on mismatch. Works for reference types and nullable value types. For non-nullable value types, use `is T x` — there's no "null" to return.
- **`(T)x`** — hard cast: throws `InvalidCastException` on mismatch; required for unboxing to a non-nullable value type (`(int)boxedInt`).

Idiom: always prefer `is T x` over `as T` + null check:

```csharp
// ❌ old idiom
if (o as string is string s) …

// ✅ modern
if (o is string s) …
```

## Patterns — the grammar

The same grammar is used by `is`, `switch` statement arms, and `switch` expression arms. Summary table (deep dive in [if, switch, switch expression](../CONSTRUCTS/if-switch-and-switch-expression.md)):

| Pattern | Shape | Example |
|---|---|---|
| Constant | `42`, `"x"`, `null`, `nameof(X)` | `case 42:`, `x is null` |
| Type | `T`, `T name` | `x is Circle c` |
| Property | `{ Prop: pattern, … }` | `{ Radius: > 0 }` |
| Extended property (C# 10) | `{ A.B.C: pattern }` | `{ Customer.Country: "US" }` |
| Positional | `(pat, pat)` — needs `Deconstruct` | `(1, _)` |
| Relational | `<`, `>`, `<=`, `>=` | `case > 0 and < 100:` |
| Logical | `and`, `or`, `not` | `is (string or int) and not null` |
| `var` | `var name` | `case var x:` (matches anything, binds) |
| Discard | `_` | `_ =>` arm |
| List (C# 11) | `[pat, pat, ..]` | `is [1, _, 3]` |
| Slice (C# 11) | `..` or `.. pat` | `is [.. var rest]` |

### Constant pattern

Any constant expression. `null` is special-cased — `x is null` is the null check.

### Type pattern and declaration pattern

```csharp
if (o is string)      { /* check */ }
if (o is string s)    { /* check + bind */ }
```

**Important:** a type pattern never matches `null`. `((object?)null) is object` returns `false`. If you want "non-null of this type", use `{ } name` (property pattern with no properties).

### Property pattern

Equality by default, pattern-valued values chain:

```csharp
if (order is { Total: > 0, Items.Count: >= 1 }) …
```

Property patterns read through **getters**, once per arm. Side-effecting getters will fire on every evaluated arm — keep them pure.

### Positional pattern

```csharp
record Point(int X, int Y);
static string Quadrant(Point p) => p switch
{
    (> 0, > 0) => "I",
    (< 0, > 0) => "II",
    (< 0, < 0) => "III",
    (> 0, < 0) => "IV",
    _          => "origin-or-axis"
};
```

Requires a `Deconstruct(out …)` method (records synthesize one; tuples always deconstruct). User types can define one:

```csharp
public void Deconstruct(out int x, out int y) { x = X; y = Y; }
```

### Relational and logical patterns

```csharp
code is >= 200 and < 300           // HTTP 2xx
value is not (null or "" or "-")   // three rejected constants
```

Precedence: `not` is tightest, then `and`, then `or`. Parenthesize generously.

### `var` pattern

Matches **anything**, including `null`. Useful for capturing an intermediate result:

```csharp
if (Compute(x) is var y && y > threshold) …
```

Don't use it as a "match non-null" — `{ } name` does that:

```csharp
if (o is { } nonNull) …      // non-null, bound
if (o is var maybeNull) …    // anything, bound (even null)
```

### List patterns (C# 11)

```csharp
int[] xs = [1, 2, 3, 4];
xs is [1, .., 4];                // True — starts with 1, ends with 4
xs is [_, _, _, _];              // True — exactly 4
xs is [var first, .., var last]; // True, first=1, last=4
xs is [.. var middle, _];        // middle = [1, 2, 3]
```

Works on anything satisfying the indexable+countable shape (indexer taking `int` + `Length` or `Count`). Works on `string`, `T[]`, `List<T>`, `Span<T>`, `ReadOnlySpan<T>`. Slice binding (`.. var rest`) requires the `Slice(int, int)` method or a `Range` indexer.

## `is` vs switch expression vs switch statement

One-liner:

| You want | Use |
|---|---|
| Single-condition with bind | `if (o is T x)` |
| Dispatch with fall-through or side effects | `switch` statement |
| Compute a value from a shape | `switch` expression |

Details in [if, switch, switch expression](../CONSTRUCTS/if-switch-and-switch-expression.md).

## `with` expression — non-destructive mutation

```csharp
public record Customer(string Name, string Country, int Tier);

var c1 = new Customer("Alice", "US", 1);
var c2 = c1 with { Tier = 2 };        // new Customer("Alice", "US", 2)
```

How it works:
- Compiler emits a synthesized virtual `<Clone>$` method on records, and `with` calls it to produce a shallow copy, then applies the initializer block.
- Because `<Clone>$` is virtual, `with` on a base-typed reference produces the correct runtime type:

```csharp
Customer c = new PreferredCustomer("Alice", "US", 1, Points: 500);
var c2 = c with { Tier = 2 };          // still PreferredCustomer at runtime
```

- C# 10 extended `with` to **structs** and **anonymous types**, not just records. C# 10 also allowed `with` on tuples.
- You cannot `with` an arbitrary class — only records (class or struct) and structs. You cannot assign through a property with no setter/init.

## Tuple expressions

```csharp
var point   = (X: 3, Y: 4);            // ValueTuple<int, int>, names are compile-time metadata
var (x, y)  = point;                   // deconstruction
point.X;                               // 3
```

Under the hood it's `System.ValueTuple<T1, T2>` — names are compile-time only (they live in `[TupleElementNamesAttribute]` on the emitted signatures). Tuples are value types; they're copied on assignment. For anything more than a short-lived group, prefer a `record` so you get structural equality and a real type name in stack traces.

Deconstruction is pattern-aware:

```csharp
if (TryGet(id) is (true, var value)) …   // deconstruct (bool Found, T Value)
```

## Target-typed `new()` (C# 9)

```csharp
Dictionary<string, int> counts = new();             // inferred from the target
List<Point> points = new() { new(1,2), new(3,4) };  // nested too
```

Use when the target type is already obvious on the left; don't obscure readability. For collections specifically, the **collection expression** `[]` is usually clearer — see next section.

## Collection expressions (C# 12+)

```csharp
int[]                xs  = [1, 2, 3];                  // array literal
List<int>            ys  = [1, 2, 3];                  // List<int>
Span<int>            sp  = [1, 2, 3];                  // stack-allocated on the current frame
ReadOnlySpan<int>    ros = [1, 2, 3];                  // allocation-free literal
IEnumerable<int>     ie  = [1, 2, 3];                  // materialized; compiler picks a type
ImmutableArray<int>  ia  = [1, 2, 3];                  // via [CollectionBuilderAttribute]
HashSet<string>      hs  = ["a", "b"];                 // C# 13 widened set of supported types
```

Targets in .NET 8+:
- Arrays: `T[]`, `T[,]` (rectangular supported in newer proposals).
- Spans: `Span<T>`, `ReadOnlySpan<T>` (prefer these in hot paths — stack-allocated when safe).
- BCL collections: `List<T>`, `HashSet<T>`, `Queue<T>`, `Stack<T>`, etc.
- Interfaces: `IEnumerable<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`, `ICollection<T>`, `IList<T>` — the compiler picks a concrete type.
- Custom collections with `[CollectionBuilderAttribute]` (see below).

### Spread operator `..`

```csharp
int[] head = [1, 2];
int[] tail = [3, 4];
int[] all  = [..head, 0, ..tail];     // [1, 2, 0, 3, 4]
```

Spreads flatten another enumerable into the literal. If the source is an `IEnumerable<T>`, it's lazily enumerated — but the resulting literal is still materialized. For large sources in a performance-sensitive path, prefer a span-typed target.

### Empty literal

```csharp
int[] empty1 = [];                   // Array.Empty<int>() in the hot path, compiler-optimized
ReadOnlySpan<int> empty2 = [];       // default(ReadOnlySpan<int>)
```

The compiler special-cases the empty literal. For `T[]` it uses `Array.Empty<T>()`; for `ReadOnlySpan<T>` it uses `default`.

### `[CollectionBuilderAttribute]` — opt-in for custom immutable collections

```csharp
[CollectionBuilder(typeof(MyList), nameof(MyList.Create))]
public sealed class MyList<T> : IReadOnlyList<T> { … }

public static class MyList
{
    public static MyList<T> Create<T>(ReadOnlySpan<T> items) => new(items.ToArray());
}

MyList<int> xs = [1, 2, 3];   // compiler calls MyList.Create(new ReadOnlySpan<int> { 1,2,3 })
```

The builder method must accept `ReadOnlySpan<T>` and return a collection of the expected type. `ImmutableArray<T>`, `ImmutableList<T>`, `FrozenSet<T>`, and friends all participate.

### `params` collection expressions (C# 13)

```csharp
static int Sum(params ReadOnlySpan<int> xs) { … }

Sum(1, 2, 3);            // no array allocation — stack-based
Sum([1, 2, 3]);          // same, explicit literal
Sum([..header, ..body]); // spread a params
```

`params ReadOnlySpan<T>` replaces `params T[]` for allocation-free variadic — one of the larger hot-path wins in C# 13.

## `default` literal

```csharp
int    i = default;            // 0
string? s = default;           // null
Point p = default(Point);      // (0, 0) — full default(T) form

T Pick<T>() => default!;       // generic default
```

`default` is equivalent to `default(T)` where `T` is inferred from the target. Useful in generic code where you can't write a concrete literal, and in struct init. The null-forgiving operator (`default!`) suppresses the nullability warning when you know the caller will overwrite.

## `nameof` operator

```csharp
ArgumentNullException.ThrowIfNull(user, nameof(user));
_log.LogInformation("User {Property} changed", nameof(user.Email));
```

- Compile-time name resolution — survives rename refactors.
- Returns the **short** name: `nameof(Namespace.Type.Member)` is `"Member"`, not fully qualified.
- C# 11 relaxed the rules: you can `nameof(someParameter)` inside an attribute on the containing method, accessing the parameter before its binding — so things like `[NotNullWhen(true, nameof(value))]` work.

## `throw` expression — cross-link

```csharp
public string Name { get; } = name ?? throw new ArgumentNullException(nameof(name));
```

Depth in [try/catch/finally/throwing exceptions](../ERROR_HANDLING/try-catch-finally-catch-filters-throwing-exceptions.md).

## Null-coalescing and null-conditional — cross-link

`??`, `??=`, `?.`, `?[]` — depth in [Operators](../CONSTRUCTS/operators.md).

## Expression-bodied members

```csharp
public sealed class Money(decimal amount, string currency)
{
    public decimal Amount   => amount;
    public string  Currency => currency;
    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

Allowed on methods, properties (read-only and read-write since C# 7), indexers, constructors, finalizers, operators, local functions. Keep them **one-liners**; the moment you want a brace-block, switch back to a block body.

## Lambda expressions — brief

```csharp
Func<int, int> square = x => x * x;
Action log         = () => Console.WriteLine("hi");
var adder          = (int a, int b) => a + b;           // natural type inference (C# 10)
var greeter        = [Conditional("DEBUG")] (string s) => Console.WriteLine(s);  // attributes on lambdas (C# 10)

// static lambda — can't capture locals; prevents accidental closures
Func<int,int> f = static x => x + 1;
```

Natural delegate type (C# 10) means the compiler can synthesize an `Action`/`Func` even when there's no contextual target. Static lambdas catch closure bugs in hot paths. Full depth belongs in a future note.

## Raw strings and interpolated expressions

```csharp
var name = "world";
var msg  = $"hello, {name}";
var json = $$"""
{ "user": "{{name}}", "level": 5 }
""";
```

- `$"…"` — interpolation. Each hole is `{expr[,alignment][:format]}`.
- `$$"…"` (C# 11) — double-dollar interpolation; `{` is literal, `{{` starts a hole. Useful inside JSON / braces-heavy text.
- `"""..."""` — raw string literal (C# 11). No escapes — whatever you type is what you get. Opening/closing fence length must be `>= 3`; the longer you make it, the more `"` characters you can include in the body.

Performance note: interpolated strings are **not** always string concatenation. `DefaultInterpolatedStringHandler` (used by `$"…"` in arguments of log/assert APIs) can evaluate conditionally. See [Interpolated string handlers](../GENERAL_TOPICS/interpolated-string-handlers.md).

## Senior-level gotchas

- **`null is T` is always `false`**, even when `T` is `object?`. Use `is null` / `is not null`. Type patterns never match null.
- **`var` pattern matches everything, including null.** Use `{ } x` when you want "non-null and bind."
- **`?.` on a value-typed member still yields a nullable** (`user?.BirthDate.Year` → `int?`). The whole null-conditional chain is nullable.
- **`!.` is compile-time only** — no runtime null check. Lying to the compiler silently reintroduces NREs.
- **Property patterns call getters**, once per evaluated arm. Side-effecting getters run during matching. Keep them pure.
- **`[1, 2, 3]` into `ReadOnlySpan<T>` is stack-allocated**; into `IEnumerable<T>` allocates an array. Choose the target type deliberately in hot paths.
- **Spread `..xs` of an `IEnumerable<T>` enumerates lazily**, but the literal materializes — for large sources in a span-typed target, prefer a pre-sized buffer or `CollectionsMarshal.AsSpan(list)`.
- **`with` preserves the runtime type on records** via the synthesized virtual `<Clone>$` method, so `BaseRecord x with { … }` stays a derived record.
- **`with` on a class that's not a record won't compile** — you need to write your own copy method, or make it a record.
- **List patterns and ranges both rely on the pattern-based shape** (`Length`/`Count` + indexer + optional `Slice`). A type that's enumerable but not indexable won't participate.
- **`is` can match an interface** — handy for feature tests: `if (o is IAsyncDisposable ad) await ad.DisposeAsync();`.
- **`nameof` returns the short name**; use `$"{typeof(T).FullName}.{nameof(T.X)}"` if you need the qualified one.
- **`default` in generic code with NRT** yields `null` for reference types — the compiler may warn `CS8603`. Fix with `where T : notnull` or a `T?` return type.
- **Don't over-use target-typed `new()`** — `IEnumerable<Item> items = new();` doesn't compile, but even when it does, the reader has to track two sides. Collection expressions `[]` read better for collections.
- **Empty collection literal `[]` is target-typed**: `[].Length` doesn't compile (no target), but `int[] xs = [];` picks `Array.Empty<int>()` and allocates nothing.
- **Interpolated strings aren't always concatenation.** In `Debug.Assert(cond, $"…")` and `_log.LogInformation($"…")` the handler can short-circuit when disabled — don't pre-build the string yourself.
