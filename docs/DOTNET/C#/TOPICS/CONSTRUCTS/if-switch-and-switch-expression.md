# `if`, `switch`, and switch expression

_Targets .NET 10 / C# 14. See also: [Pattern matching & expressions](../EXPRESSIONS/member-access-type-testing-pattern-matching-collection-expressions.md), [Operators](./operators.md), [break, continue, return](./break-continue-return.md)._

Three shapes of conditional dispatch. `if` is the flexible all-purpose one; `switch` statement is legacy-shaped but now pattern-aware; the `switch` expression (C# 8) is the one you reach for when you need a **value**. This note focuses on the places where the three differ — exhaustiveness, conversion, flow rules — and on patterns that work in all of them.

## `if` / `else` — the pragmatic default

Nothing surprising:

```csharp
if (user.IsLocked)
    return Unauthorized();

if (user.IsGuest && !policy.AllowGuests)
    return Forbid();

return Ok(user);
```

Style rules that earn their keep in senior code:
- **Guard-clause early returns** over pyramid-of-doom nesting. A flat method with 4 `if (x) return …` lines is easier to debug than one with 4 nested `else`.
- `if` accepts any expression of type `bool` (or one the compiler can coerce via `operator true`). Don't lean on that — stick to explicit booleans.
- **Ternary vs `if`**: `return x ? a : b;` for expression selection; `if` for side-effecting branches. Mixing them (`var x = cond ? Side1() : Side2();`) is a smell.

### Pattern-matching with `is` in `if`

```csharp
if (shape is Circle { Radius: > 0 } c)
    area = Math.PI * c.Radius * c.Radius;
```

The bound variable (`c`) is in scope in the `if` body **and** along the fall-through path if the compiler can prove definite assignment:

```csharp
if (o is not string s) return;
// `s` is in scope here — the compiler knows we returned otherwise
Console.WriteLine(s.Length);
```

This is the idiom `is not` was designed for.

## `switch` statement — the imperative one

```csharp
switch (status)
{
    case OrderStatus.Pending:
    case OrderStatus.Paid:
        Process();
        break;

    case OrderStatus.Shipped when tracking is not null:
        Track(tracking);
        break;

    case OrderStatus.Cancelled:
        goto case OrderStatus.Refunded;     // explicit fall-through
    case OrderStatus.Refunded:
        Refund();
        break;

    default:
        throw new UnreachableException();
}
```

Things to know:
- **No implicit fall-through.** Every non-empty case must `break`, `return`, `throw`, `continue`, or `goto`. Empty cases stack (first two above).
- **`goto case X`** is the only way to fall through explicitly. Rare but the right tool for state-machine–shaped switches.
- **`when` clauses** gate a case with any boolean expression. The first matching arm wins; later arms don't fire.
- **Order matters**: since patterns can overlap (e.g. `int` vs `> 0`), arms are tested top-to-bottom. Place more-specific patterns first.
- **Integral/string switches compile to a jump table** (dense) or a `Dictionary<string,int>` lookup (sparse strings). Pattern-based switches compile to chained `if`/`else` — no jump table. For hot code on constant ints, prefer integral cases over relational patterns if you care about the last few nanoseconds.

## `switch` expression — the functional one

```csharp
decimal fee = card switch
{
    { Brand: "Amex", International: true } => 0.035m,
    { Brand: "Amex" }                      => 0.028m,
    { International: true }                => 0.025m,
    null                                   => throw new ArgumentNullException(nameof(card)),
    _                                      => 0.018m
};
```

Key rules:
- **Produces a value** — every arm must yield something convertible to one common type (or a shared target type in C# 9+).
- **Arms are `pattern [when guard] => expression`**. No `break`, no fall-through.
- **`_`** is the discard pattern — matches anything, doesn't bind.
- **Exhaustiveness**: the compiler emits `CS8509` ("switch expression does not handle all possible values") if it can prove a gap. On a `bool` or a sealed hierarchy, it will often demand a `_ =>` or throw. If no arm matches at runtime, the expression throws `SwitchExpressionException`.
- **No `goto`** inside arms — arms are expressions.

### Return-type convergence

All arms must convert to one type. If they don't, C# 9 target-typing helps:

```csharp
object o = flag switch
{
    true  => 1,
    false => "no"
};
// OK: target is `object`, and both arms are convertible.

// But:
var x = flag switch { true => 1, false => "no" };
// CS8506: no best type — the compiler can't infer.
```

Solution when the natural result is a discriminated set: project to a common base (interface, `object`, or a tuple with named members).

### Defensive exhaustiveness — `UnreachableException`

```csharp
string Label(HttpMethod m) => m switch
{
    HttpMethod.Get     => "GET",
    HttpMethod.Post    => "POST",
    HttpMethod.Put     => "PUT",
    HttpMethod.Delete  => "DELETE",
    _ => throw new UnreachableException($"Unknown method: {m}")
};
```

`UnreachableException` (added in .NET 7) is the right thing to throw in a `_` arm that *should* never fire — much clearer than `InvalidOperationException`. Combined with a sealed enum or hierarchy, it documents intent while keeping the compiler satisfied.

## Patterns — the grammar shared across `is`, `switch`, and switch expressions

| Pattern | Shape | Example |
|---|---|---|
| Constant | `42`, `"x"`, `null` | `case 42:`, `x is null` |
| Type | `Type`, `Type name` | `case Circle c:` |
| Property | `{ Prop: pattern, … }` | `{ Radius: > 0 }` |
| Positional | `(pat, pat)` (needs `Deconstruct`) | `(1, _)` |
| Relational | `< > <= >=` | `case > 0 and < 100:` |
| Logical | `and`, `or`, `not` | `is (string or int) and not null` |
| `var` | `var name` | `case var x:` (matches anything, binds) |
| Discard | `_` | default case in expression form |
| List (C# 11+) | `[pat, pat, ..]` | `is [1, _, 3]` |

Combine freely:

```csharp
static string Bucket(Order o) => o switch
{
    { Total: < 0 }                        => "refund",
    { Items: [] }                         => "empty",
    { Items: [{ Qty: > 0 }] }             => "single-line",
    { Customer.Country: "US" or "CA" }    => "north-america",
    _                                     => "other"
};
```

`o.Customer.Country` is an **extended property pattern** (C# 10) — chain dotted access inside a property pattern, no need to nest `{ Customer: { Country: … } }`.

### List patterns (C# 11)

```csharp
int[] a = [1, 2, 3, 4];
a is [1, .., 4];            // True
a is [_, _, _, _];          // True — exactly 4
a is [var first, .., var last] and { Length: > 1 };
```

Work on anything that satisfies both an indexer (`[int]`) and a `Length`/`Count` property — so `string`, `T[]`, `List<T>`, `Span<T>`, `ReadOnlySpan<T>`.

## Picking between the three

| You want | Use |
|---|---|
| Run statements based on a boolean | `if/else` |
| Dispatch on a handful of enum/constant values with side-effects | `switch` statement |
| Dispatch with fall-through / `goto case` | `switch` statement |
| Compute a value from a shape | `switch` expression |
| Tight pattern + bind inside a single predicate | `is` in an `if` |

## Senior-level gotchas

- **`switch` expression throws `SwitchExpressionException` on no match at runtime** — don't ignore `CS8509`; either cover the gap or `throw new UnreachableException(...)` explicitly.
- **Pattern `switch` compiles to chained branches**, not a jump table — if you genuinely need jump-table speed on dense integer dispatch, keep to pure constant cases.
- **Order-sensitive arms**: when multiple patterns could match, the first wins. Moving `{ IsAdmin: true }` below `{ Department: "HR" }` can silently change behavior.
- **`when` guards defeat exhaustiveness analysis** — the compiler stops trying to prove completeness once it sees one, and may require a `_` arm even when the shape is closed.
- **`var` pattern always matches**, including `null`. It's shorthand for "bind and continue." Use `{ } name` to match non-null + bind, or a type pattern for a null-rejecting match.
- **Property patterns test equality on value-typed getters once per arm** — side-effecting getters (DB calls, counters) will run on every arm test. Keep getters pure.
- **`is null` vs `== null`**: for types that may overload `==` (including many records and structs), `is null` is the **only** way to do a true null check. Analyzer `CA1508` nudges you.
- **C# 14 field-backed properties** interact with property patterns — the compiler still sees the property, not the field, so pattern matching works unchanged.
- **Interview bait**: "does switch expression have return-type limitations?" — yes, all arms must converge on a common type, and target-typing (C# 9) only works when there's an explicit target. See interview Q30.
