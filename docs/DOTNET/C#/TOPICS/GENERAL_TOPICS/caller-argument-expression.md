# `CallerArgumentExpression`

_Targets .NET 10 / C# 14. See also: [Interpolated string handlers](./interpolated-string-handlers.md), [Tuples, attributes, regex](./tuples-attributes-regex.md), [Error handling](../ERROR_HANDLING/try-catch-finally-filters-throwing.md)._

`CallerArgumentExpressionAttribute` (C# 10, `System.Runtime.CompilerServices`) lets a method capture the **source text** of an argument the caller passed, injected at compile time as a string. It joined the `Caller*` family (`CallerMemberName`, `CallerFilePath`, `CallerLineNumber`) and unlocked `ArgumentNullException.ThrowIfNull`-style guard APIs that show the original expression in the exception message.

## The shape

```csharp
public static void Requires(
    [DoesNotReturnIf(false)] bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expression = null)
{
    if (!condition)
        throw new InvalidOperationException($"Precondition failed: {expression}");
}

var user = GetUser();
Requires(user is { IsActive: true, Id: > 0 });
// Throws: "Precondition failed: user is { IsActive: true, Id: > 0 }"
```

The compiler substitutes the literal source text of `condition` into `expression` at the call site. Whitespace is preserved verbatim.

## How the compiler wires it up

At each call site, the compiler:

1. Binds the target parameter named by the attribute (`nameof(condition)`).
2. Takes the **syntactic text** of the expression passed for that parameter.
3. Emits it as a constant string literal — **only if the caller didn't supply the optional parameter explicitly**.

```csharp
// What you write:
Requires(user.IsActive);

// What the compiler emits:
Requires(user.IsActive, expression: "user.IsActive");
```

If the caller passes the optional parameter themselves, their value wins. The attribute is default-injection, not a lock.

## Built-in consumers in the BCL

.NET 6 introduced guard helpers built on this:

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression(nameof(argument))] string? paramName = null)
{
    if (argument is null) throw new ArgumentNullException(paramName);
}
```

So:

```csharp
void Use(User u)
{
    ArgumentNullException.ThrowIfNull(u);          // paramName = "u"
    ArgumentNullException.ThrowIfNull(u.Profile);  // paramName = "u.Profile"
}
```

The exception message names the **expression**, not the parameter — hugely more useful when guards sit inside layered code.

Other BCL consumers:
- `ArgumentException.ThrowIfNullOrEmpty(string?)`
- `ArgumentException.ThrowIfNullOrWhiteSpace(string?)` (.NET 8)
- `ArgumentOutOfRangeException.ThrowIfNegative/ThrowIfZero/ThrowIfGreaterThan/...` (.NET 8)
- `ObjectDisposedException.ThrowIf(bool condition, object instance)` (.NET 7)

## Writing your own assertion helpers

```csharp
public static class Assert
{
    public static T NotNull<T>(
        T? value,
        [CallerArgumentExpression(nameof(value))] string? expr = null) where T : class
        => value ?? throw new InvalidOperationException($"{expr} was null");

    public static void Equal<T>(
        T expected, T actual,
        [CallerArgumentExpression(nameof(actual))] string? actualExpr = null)
    {
        if (!EqualityComparer<T>.Default.Equals(expected, actual))
            throw new InvalidOperationException(
                $"Expected {actualExpr} == {expected}, got {actual}");
    }
}

var name = Assert.NotNull(lookup.Find("alice"));
// Throws: "lookup.Find(\"alice\") was null"

Assert.Equal(200, response.StatusCode);
// "Expected response.StatusCode == 200, got 500"
```

Test framework assertion APIs (xUnit `Assert.That`, FluentAssertions) were the original motivation — automatic message building without `() => expr` lambda tricks.

## Attribute argument — must name a parameter

```csharp
// Valid:
void M(int x, [CallerArgumentExpression(nameof(x))] string? expr = null) { }

// Invalid — "y" is not a parameter of N:
void N(int x, [CallerArgumentExpression("y")] string? expr = null) { }
// CS8965: CallerArgumentExpression argument name must match a parameter of the method
```

Always use `nameof(param)` rather than a raw string — survives rename refactors.

## Interaction with other `Caller*` attributes

They stack; one method can carry all four:

```csharp
public static void Log(
    string message,
    [CallerMemberName] string? member = null,
    [CallerFilePath]   string? file   = null,
    [CallerLineNumber] int    line    = 0)
{ /* ... */ }
```

`CallerArgumentExpression` targets a **parameter**, not call-site metadata, so it coexists rather than competes.

## Pair with nullable flow attributes

Pair `CallerArgumentExpression` with `DoesNotReturnIf`/`[NotNull]` so the nullable analyzer knows the helper aborts — otherwise every `ThrowIfNull` call site still sees the variable as "maybe null":

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression(nameof(argument))] string? paramName = null)
{
    if (argument is null) throw new ArgumentNullException(paramName);
}
```

`[NotNull]` tells the flow analysis "after this call, `argument` is non-null."

## When it's the wrong tool

- **Never** use it to change behavior based on the expression text (parsing the string to choose a branch). The attribute is for **diagnostics**, not metaprogramming.
- **Never** leak the expression text across a security boundary — a user-controlled expression interpolated into an error message can surface field or DB-column names you'd rather not expose.
- **Don't** rely on it in reflection/expression-tree contexts — the substitution is the **caller's** compile-time, so `MethodInfo.Invoke(helper, new[] { arg })` passes `null`.

## Senior-level gotchas

- **Delegate wrappers break substitution.** `Action<object?> guard = ArgumentNullException.ThrowIfNull;` invoked as `guard(x)` reports `"argument"` as the expression, because the call site the compiler sees is inside the delegate invocation, not your user code.
- **Reflection, expression trees, dynamic dispatch, `MethodInfo.Invoke`** all pass `null` for the optional parameter. The attribute is injection at **static** call sites only.
- **Long expressions are captured verbatim**, including whitespace and comments inside the expression. For error telemetry, trim/sanitize before logging if size matters.
- **`ref`/`in`/`out` parameters** work — the captured text is the operand as written, e.g. `ThrowIfNegative(ref x)` captures `"x"`.
- **The attribute adds a parameter to your public API.** Reflection on the method signature sees the optional `string?`. Pick a name that reads well in IntelliSense (`paramName`, `expression`).
- **Overload resolution** considers the optional parameter — avoid declaring both `M<T>(T)` and `M<T>(T, string?)` where ambiguity is possible.
- **`CallerArgumentExpression` captures the argument, not its value.** `ThrowIfNull(GetThing())` captures `"GetThing()"`, not the returned object. For helpers wanting the *value*, overload without the capture.
- **Mixed with explicit `null`**: `ThrowIfNull(null)` captures `"null"` (the literal text). The guard still throws, message just reads `"null"`.
- **Trimming/NativeAOT safe** — compile-time only, no runtime metadata reading.
- **F# 8+** supports it; older F# and VB.NET pass `null`. Cross-language guard helpers should defend against empty expression text.
