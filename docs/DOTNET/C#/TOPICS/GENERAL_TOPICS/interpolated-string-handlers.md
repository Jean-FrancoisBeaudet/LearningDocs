# Interpolated string handlers

_Targets .NET 10 / C# 14. See also: [CallerArgumentExpression](./caller-argument-expression.md), [Span / Memory](../MEMORY_MANAGEMENT/span-memory.md), [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md)._

Interpolated string handlers (C# 10 / .NET 6) change how `$"…"` is compiled. Before .NET 6, an interpolated string lowered to `string.Format` — one allocation for the format string, another for the `object[]` boxing holes, plus whatever the formatters did. Handlers give the compiler a way to hand off the pieces of the interpolation to a custom `ref struct` that builds the result directly, skipping all of that. They also enable **conditional evaluation** of interpolation holes (e.g., in `Debug.Assert`) — the single most impactful perf change.

## The default handler: `DefaultInterpolatedStringHandler`

Every `$"…"` in C# 10+ already uses one — `DefaultInterpolatedStringHandler` in `System.Runtime.CompilerServices`. It's a `ref struct` that:

1. Rents a char buffer from `ArrayPool<char>.Shared` sized from `literalLength` + `formattedCount`.
2. Receives each literal slice and each hole via `AppendLiteral(string)` / `AppendFormatted<T>(T)` calls.
3. For `ISpanFormattable` types, formats **directly into the buffer** — no intermediate string.
4. Returns the final string via `ToStringAndClear()`, which also returns the buffer to the pool.

What you write:

```csharp
var s = $"User {user.Id} logged in at {DateTime.UtcNow:O}";
```

What the compiler emits (conceptually):

```csharp
var h = new DefaultInterpolatedStringHandler(literalLength: 18, formattedCount: 2);
h.AppendLiteral("User ");
h.AppendFormatted(user.Id);
h.AppendLiteral(" logged in at ");
h.AppendFormatted(DateTime.UtcNow, format: "O");
string s = h.ToStringAndClear();
```

Any `ISpanFormattable` (int, Guid, DateTime, decimal, Span<char>, many more) formats without allocating an intermediate string — a significant win for logging, telemetry, and hot paths.

## Writing a custom handler — the key APIs

A type is a valid interpolated string handler if it:

1. Is attributed with `[InterpolatedStringHandler]`.
2. Has a constructor `(int literalLength, int formattedCount, ...)` — the compiler always passes those two values first, optionally followed by additional parameters you declare.
3. Exposes public `AppendLiteral(string)` and one or more `AppendFormatted<T>(T)` / `AppendFormatted(ReadOnlySpan<char>, int alignment = 0, string? format = null)` overloads.
4. Typically (but not required) has a `ToStringAndClear()` / `ToString()` shape or is consumed directly by a calling method.

Most custom handlers are `ref struct`s to stay off the heap.

## Conditional interpolation — `bool shouldAppend` and `out bool handlerIsValid`

Handlers can short-circuit. The constructor may accept an extra `out bool` parameter; if the constructor sets it to `false`, the compiler **skips evaluating all interpolation holes**. Each `AppendFormatted` call can also return `bool` to bail out partway.

```csharp
[InterpolatedStringHandler]
public ref struct AssertInterpolatedStringHandler
{
    private DefaultInterpolatedStringHandler _inner;
    private readonly bool _enabled;

    public AssertInterpolatedStringHandler(
        int literalLength, int formattedCount,
        bool condition,                        // extra parameter supplied by the API
        out bool shouldAppend)                 // the compiler reads this
    {
        if (condition)
        {
            _enabled = false;
            _inner = default;
            shouldAppend = false;              // skip all holes
        }
        else
        {
            _enabled = true;
            _inner = new(literalLength, formattedCount);
            shouldAppend = true;
        }
    }

    public void AppendLiteral(string s) { if (_enabled) _inner.AppendLiteral(s); }
    public void AppendFormatted<T>(T v)  { if (_enabled) _inner.AppendFormatted(v); }

    internal string GetResult() => _enabled ? _inner.ToStringAndClear() : "";
}

public static class Assertions
{
    public static void Require(
        bool condition,
        [InterpolatedStringHandlerArgument("condition")]
        ref AssertInterpolatedStringHandler message)
    {
        if (!condition)
            throw new InvalidOperationException(message.GetResult());
    }
}

Assertions.Require(user.IsActive, $"User {user.Id} '{user.Profile?.Name}' is inactive");
// When user.IsActive is true: no formatting, no string allocation, no Profile access.
```

`[InterpolatedStringHandlerArgument(...)]` tells the compiler to forward named caller arguments into the handler constructor. That's what makes `condition` available in the handler.

This is exactly how `Debug.Assert(bool, [InterpolatedStringHandlerArgument("condition")] ref Debug.AssertInterpolatedStringHandler)` works — zero work when the assertion passes.

## `ILogger` integration — `LoggerMessageInterpolatedStringHandler`

`Microsoft.Extensions.Logging` in .NET 8+ ships handlers so that:

```csharp
logger.LogInformation($"Processed {order.Id} in {sw.Elapsed.TotalMilliseconds}ms");
```

…does **not** format the string when the log level is disabled. The handler checks `logger.IsEnabled(level)` in its constructor and sets the `shouldAppend` output accordingly. Same ergonomics as a formatted log message, but free when filtered out.

Prior to handler support you had to write `logger.LogInformation("Processed {Id} in {Ms}ms", order.Id, ms)` or pay the cost. Handlers gave you the best of both.

## Custom handlers for structured building

You can treat handlers as general-purpose DSLs — they don't have to produce a string at all.

```csharp
[InterpolatedStringHandler]
public ref struct SqlCommandHandler
{
    private readonly SqlCommand _cmd;
    private readonly StringBuilder _sb;
    private int _paramIndex;

    public SqlCommandHandler(int literalLength, int formattedCount, SqlCommand cmd)
    {
        _cmd = cmd;
        _sb = new StringBuilder(literalLength + formattedCount * 4);
        _paramIndex = 0;
    }

    public void AppendLiteral(string s) => _sb.Append(s);

    public void AppendFormatted<T>(T value)
    {
        var name = $"@p{_paramIndex++}";
        _sb.Append(name);
        _cmd.Parameters.AddWithValue(name, (object?)value ?? DBNull.Value);
    }

    public string GetCommandText() => _sb.ToString();
}

public static void Execute(
    this SqlCommand cmd,
    [InterpolatedStringHandlerArgument("cmd")] ref SqlCommandHandler sql)
{
    cmd.CommandText = sql.GetCommandText();
    cmd.ExecuteNonQuery();
}

// Usage:
cmd.Execute($"UPDATE users SET email = {newEmail} WHERE id = {userId}");
// Safe: builds "UPDATE users SET email = @p0 WHERE id = @p1" + parameters.
```

This pattern — safe SQL by construction — is what makes handlers more than a perf knob.

## Alignment and format specifiers

Handlers get the format string and alignment from the interpolation hole:

```csharp
$"{amount,10:C2}"   // alignment = 10, format = "C2"
```

Compiler calls `AppendFormatted(T value, int alignment, string? format)` — overload your handler with that signature if you want to honor them:

```csharp
public void AppendFormatted<T>(T value, int alignment = 0, string? format = null)
{
    // …
}
```

`DefaultInterpolatedStringHandler` overloads for `string`, `ReadOnlySpan<char>`, `IFormattable`, and `ISpanFormattable` with every combination of alignment/format — that's why BCL formatting is zero-alloc in the common case.

## Overload resolution and handler selection

When a method has parameters of type `string` and an interpolated-string-handler type, the compiler picks the handler overload when given an interpolation:

```csharp
void Log(string msg)                                   { /* fallback */ }
void Log([InterpolatedStringHandlerArgument("")]
         ref MyLogHandler msg)                          { /* preferred */ }

Log("static");        // picks the string overload
Log($"hello {x}");    // picks the handler overload
```

This lets you evolve an API to become handler-aware without breaking existing callers.

## Performance measurement

Measure, always. The perf claims depend on:

- The types in the holes (`ISpanFormattable` wins vs. `object.ToString()`).
- Buffer growth — misestimating `literalLength + formattedCount` forces reallocation.
- How often the conditional path fires (a logger with info disabled saves all formatting work; one with info enabled gains nothing over `string.Format`).

Use BenchmarkDotNet with `[MemoryDiagnoser]`. Handlers should show zero allocations in the conditional-off path.

## Senior-level gotchas

- **`ref struct` is almost mandatory** for handlers so they can't escape. A handler that hits the heap defeats the allocation-avoidance design.
- **`ToStringAndClear()` on `DefaultInterpolatedStringHandler` releases the rented buffer.** Calling `ToString()` does **not** — and leaks the rental. Always use `ToStringAndClear()` for custom handlers that delegate to the default.
- **Compiler picks the handler overload when an interpolation is passed**, but a regular string variable (`string s = $"{x}"; Log(s);`) is already a `string` — you lose the handler path. For perf-sensitive APIs, keep the interpolation inline.
- **`AppendFormatted<T>` with `T : ISpanFormattable`** formats directly; boxing happens when `T` is a value type without `ISpanFormattable`. Check BCL types — some (`TimeSpan`) gained `ISpanFormattable` only in later framework versions.
- **Custom handlers with `[InterpolatedStringHandlerArgument("...")]`** can reference instance parameters (`"this"` for the receiver when the method is an extension, or `""` for the static context). Misspelling the name is CS8953.
- **Conditional evaluation is the big performance win.** If your hot-path log messages always format (because the level is enabled), handlers save you object boxing but not much else. The real gain is when a call is **filtered**.
- **Debugging inside a handler** is awkward — the `ref struct` can't cross `await`, can't appear in a watch window reliably, and the lifetime is tied to the single expression. Log from outside the handler, not from within.
- **Exceptions inside `AppendFormatted`** halt the build-up and the half-built buffer is not returned to the pool unless your `ToStringAndClear` / `Dispose` handles it. Wrap risky formatters or check `TryFormat` first.
- **Handlers don't compose** — you can't pass an interpolated string to a method that accepts a different handler type without first materializing to `string`. Each API that wants handlers declares its own.
- **Source generators can emit handlers**, but they need to target the exact compiler contract (constructor shape, attributes, overload set). Small mistakes fall back silently to the `string` overload — the only symptom is "my handler never gets called."
- **`StringBuilder.Append(ref StringBuilder.AppendInterpolatedStringHandler)`** exists for the same reason `string`-producing handlers do: build without intermediate `string` allocations. `$"…"` appended to a `StringBuilder` uses it automatically.
- **AOT / trimming**: handlers work under NativeAOT; generic `AppendFormatted<T>` may introduce code size if you're shipping many handlers for many types — prefer `ISpanFormattable` to keep dispatch virtual-ish.
