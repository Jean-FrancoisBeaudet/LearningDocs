# Operators: Arithmetic, Boolean, Bitwise, Ternary, Comparison, Equality

_Targets .NET 10 / C# 14. See also: [Member access, type testing, pattern matching](../EXPRESSIONS/member-access-type-testing-pattern-matching-collection-expressions.md), [Value types](../TYPES/value-types.md), [Reference types](../TYPES/reference-types.md), [Classes, records, structs](../OOP/classes-records-structs.md)._

C# operators look trivial until you get to equality, overflow, and pattern combinators. This note focuses on the parts where **what** an operator does is obvious but **what the runtime does** is not — overflow semantics, short-circuit vs bitwise, floating-point gotchas, and where user-defined operators change the calculus.

## Arithmetic — `+ - * / %`

```csharp
int a = 7, b = 3;
a / b;      // 2     — integer division, truncates toward zero
a % b;      // 1
7.0 / 3;    // 2.333… — one operand is double → both promoted
```

Division rules:
- **Integer `/`**: truncates toward zero. `-7 / 3 == -2` (not `-3`).
- **Integer `%`**: sign follows dividend. `-7 % 3 == -1`.
- **Floating `/` by zero**: produces `±Infinity` or `NaN` (never throws).
- **Integer `/` by zero**: throws `DivideByZeroException`.

### Overflow: `checked` vs `unchecked`

Integer arithmetic is **unchecked by default** (wraps silently). Change the default per-project with MSBuild:

```xml
<PropertyGroup>
  <CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
</PropertyGroup>
```

Or scope it in code:

```csharp
int max = int.MaxValue;

unchecked { int w = max + 1; }   // wraps to int.MinValue
checked   { int t = max + 1; }   // throws OverflowException

// Expression forms:
int x = checked(a * b);
int y = unchecked(a * b);
```

Guidance: turn `CheckForOverflowUnderflow` on for financial / safety-critical code and use `unchecked { }` blocks for the hot arithmetic you've *proven* can't overflow (hash combining, bit packing). Default-off + explicit `checked(…)` at the boundary is also a valid style.

### Generic math (C# 11)

Operators on generic types are now expressible via `INumber<T>` and friends:

```csharp
static T Sum<T>(ReadOnlySpan<T> xs) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var x in xs) total += x;
    return total;
}
```

`T.Zero`, `T.One`, `T.AdditiveIdentity`, and the `+`, `-`, `*`, `/`, `%`, comparison and equality operators are all virtual statics on the interface. This is why C# 11 added **static abstract interface members**.

## Boolean — `&& || !` (short-circuit) vs `& | ^` (eager)

```csharp
if (x != null && x.Count > 0)    // safe — short-circuits on null
if (x != null & x.Count > 0)     // 💥 NullReferenceException on null x
```

`&&` / `||` short-circuit: the right operand is evaluated only if needed. `&` / `|` always evaluate both sides. `^` is logical XOR (no short-circuit form — both sides always needed).

The single-operand forms exist because they work on **any** integral type and on custom types that define `operator &(T, T)` plus `operator true(T)` / `operator false(T)`. That's how `&&` works on user types: the compiler lowers `a && b` to `T.false(a) ? a : T.&(a, b)`.

Short-circuit interacts with side effects — count on it, or refactor:

```csharp
if (TryGetCache(key, out var v) || TryGetRemote(key, out v))  // second call only on cache miss
    return v;
```

## Bitwise — `& | ^ ~ << >> >>>`

```csharp
0b1100 & 0b1010;   // 0b1000
0b1100 | 0b1010;   // 0b1110
0b1100 ^ 0b1010;   // 0b0110
~0b1100;           // ...11110011 (two's complement)

int x = -8;
x >> 1;            // -4  — arithmetic shift, sign bit replicated
x >>> 1;           // 2147483644 — logical shift, zero-filled (C# 11+)
```

`>>>` (C# 11) is the unsigned-right-shift operator. Before C# 11, you needed to cast to `uint`, shift, then cast back.

Shift counts are masked to the operand's bit width (`x << 33` on an `int` acts as `x << 1`). `BitOperations` in `System.Numerics` covers rotate / popcount / leading-zero count in hardware-intrinsic-optimized form.

## Ternary and the null family — `?: ?? ??= ?. ?[]`

```csharp
var label = n == 1 ? "item" : "items";

string name = userInput ?? "anonymous";      // coalesce
cache ??= LoadOnce();                        // assign-if-null

int? len = user?.Name?.Length;               // chain, short-circuit on null
var first = list?[0];                        // null-conditional indexer
```

Ternary arms must share a **common type** (C# 9 target-typed `?:` relaxes this — both arms can be convertible to the target):

```csharp
// Pre-C# 9: CS0173 — no common type between `int` and `string`
// C# 9+: OK if the target (here `object`) accepts both.
object o = flag ? 1 : "x";
```

Two senior-level notes:
- `??` and `?.` do **not** create new nullable types. `user?.Name` where `Name` is non-nullable `string` yields `string?` because the whole expression can be null.
- `??=` evaluates the RHS **only** when the LHS is null — useful for lazy init without `Lazy<T>`'s overhead on single-thread paths.

## Comparison — `< <= > >=`

Defined on numeric primitives and any type with matching operators or implementing `IComparable<T>`/`IComparisonOperators<T,T,bool>` (C# 11).

```csharp
record Money(decimal Amount, string Currency) : IComparable<Money>
{
    public int CompareTo(Money? other) => other is null
        ? 1
        : Amount.CompareTo(other.Amount);   // ignoring currency on purpose
}
```

Strings compare ordinally via `string.Compare(a, b, StringComparison.Ordinal)` — **avoid** `a < b` on strings; the compiler rejects it, but don't reach for `a.CompareTo(b) < 0` either if a culture-aware compare is the real intent. Be explicit:

```csharp
if (string.Compare(a, b, StringComparison.OrdinalIgnoreCase) > 0) …
```

## Equality — `==` `!=` vs `Equals` vs `ReferenceEquals`

This is the most booby-trapped area of C# equality, and where interviews concentrate:

```csharp
object a = "hello";
object b = "hello";
a == b;                 // True — reference equality on `object`, but string interning
a.Equals(b);            // True — virtual Equals dispatches to String.Equals

string s1 = "hello";
string s2 = "hello";
s1 == s2;               // True — string overloads operator ==, calls String.Equals
ReferenceEquals(s1, s2);// True  — both pooled via string interning; don't rely on it

string s3 = new string('x', 3);
string s4 = new string('x', 3);
ReferenceEquals(s3, s4);// False — distinct allocations; == is still True
```

The rules:
- **`==` on `object`**: reference equality.
- **`==` on a type that overloads it** (strings, records, many value types): whatever the overload says.
- **`Equals(object)`**: virtual — dispatches to the runtime type.
- **`EqualityComparer<T>.Default.Equals`**: picks the best implementation for `T` (IEquatable, ValueType default, reference). **Use this in generic code**, not `==` — the latter binds at compile time.

```csharp
public static bool Same<T>(T a, T b)
{
    // return a == b;                         // CS0019 unless T is constrained
    return EqualityComparer<T>.Default.Equals(a, b);
}
```

### Value-type equality — the free-by-default version is slow

Structs without an explicit `Equals` override fall back to `ValueType.Equals`, which **uses reflection** to compare fields. That's a ~100x performance cliff. Always override `Equals`, `GetHashCode`, and `operator ==`/`!=` on value-typed keys. Records do this for you.

### Record equality

```csharp
public record Point(int X, int Y);
new Point(1,2) == new Point(1,2);   // True  — structural equality
```

The compiler synthesizes `Equals(Point?)`, `==`, `!=`, `GetHashCode`, and a virtual `EqualityContract`. Inherit a record and the contract extends — two records of different runtime types are never equal, even with identical fields.

### Floating-point: `NaN != NaN`

```csharp
double nan = double.NaN;
nan == nan;              // False — IEEE 754
nan.Equals(nan);         // True  — Equals treats NaN as equal to itself

double.NaN == double.NaN;// False
```

So `list.Contains(double.NaN)` works (uses `Equals`) but `list.IndexOf(x => x == double.NaN)` doesn't. Use `double.IsNaN(x)`.

### Nullable equality

```csharp
int? a = null, b = null;
a == b;                  // True — both "has no value"
a < b;                   // False — any comparison with null yields false
```

## Pattern operators: `is` `is not` `and` `or`

```csharp
if (input is string { Length: > 0 } s) …
if (code is >= 200 and < 300) …
if (user is not { IsActive: true }) return;
```

These are **not** boolean operators in the traditional sense — they're part of the pattern-matching grammar. `and`, `or`, `not` exist only inside a pattern; outside, you still write `&&`, `||`, `!`. Details in [pattern matching](../EXPRESSIONS/member-access-type-testing-pattern-matching-collection-expressions.md) and [switch expression](./if-switch-and-switch-expression.md).

## User-defined operators

### Operator overloading

```csharp
public readonly struct Meters(double Value)
{
    public static Meters operator +(Meters a, Meters b) => new(a.Value + b.Value);
    public static Meters operator *(Meters a, double s) => new(a.Value * s);
    public static bool operator ==(Meters a, Meters b) => a.Value == b.Value;
    public static bool operator !=(Meters a, Meters b) => !(a == b);

    public override bool Equals(object? o) => o is Meters m && this == m;
    public override int GetHashCode() => Value.GetHashCode();
}
```

Rules:
- Operators are `public static`.
- Override `==`/`!=` together; override `Equals` and `GetHashCode` **together with them** (analyzer `CS0660`/`CS0661`).
- Overloading `true` / `false` operators lets your type drive `if (x)` and `&&`/`||`.

### `checked` user-defined operators (C# 11)

```csharp
public static Meters operator +(Meters a, Meters b)        => new(a.Value + b.Value);
public static Meters operator checked +(Meters a, Meters b) => new(checked(a.Value + b.Value));
```

When the caller is in a `checked` context, the `checked` operator is selected. Without a `checked` overload, the unchecked one is used unconditionally — a subtle bug factory if you advertised overflow safety.

### Static abstract / virtual on interfaces (C# 11)

```csharp
public interface IAddable<T> where T : IAddable<T>
{
    static abstract T operator +(T a, T b);
}
```

Foundation for `INumber<T>`. Callable via generics.

### Conversion operators

```csharp
public static implicit operator double(Meters m) => m.Value;   // loss-free → implicit
public static explicit operator Meters(double d) => new(d);    // intent needed → explicit
```

Guidance: `implicit` only if the conversion **cannot lose information and never throws**. Otherwise `explicit` — force the caller to write the cast.

## Senior-level gotchas

- **`==` in generic code** binds to `object.==` unless you constrain `T : IEqualityOperators<T,T,bool>` (C# 11) — use `EqualityComparer<T>.Default.Equals` instead.
- **Struct default `Equals` uses reflection** — always override it. Records of struct shape (`record struct`) fix this automatically.
- **`Dictionary<string, V>` with a default comparer is ordinal, not culture-aware** — still, *always* pass `StringComparer.Ordinal` explicitly so the next reader isn't guessing.
- **`NaN != NaN`**, but `NaN.Equals(NaN) == true`, and `EqualityComparer<double>.Default.Equals(NaN, NaN) == true` — LINQ/collection membership uses `Equals`, so NaN *can* be "found". Sort ordering of NaN is implementation-defined; filter it out before sorting.
- **`a == b` on two `object`s holding the same unboxed value** returns `false` — each box is a new allocation. Always unbox or use `.Equals`.
- **Overloading `==` without overriding `Equals`/`GetHashCode`** breaks `HashSet` / `Dictionary` silently. Analyzer `CS0660` flags it; treat it as error.
- **`checked` only affects operators in the same expression** — not those inside called methods. `checked(Compute(max, max))` does not propagate into `Compute`.
- **`?.` on a value type returns nullable**: `user?.BirthDate.Year` where `BirthDate` is `DateTime` yields `int?`, not `int`.
- **Unsigned right shift `>>>` is distinct from `>>`** — on `int`/`long`, `>>` is arithmetic (sign-preserving). Mixing them silently changes results for negative values.
