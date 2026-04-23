# `break`, `continue`, `return` (and `goto`, `yield break`)

_Targets .NET 10 / C# 14. See also: [Loops](./loops.md), [if, switch, switch expression](./if-switch-and-switch-expression.md), [try/catch/finally](../ERROR_HANDLING/try-catch-finally-catch-filters-throwing-exceptions.md)._

The escape hatches of structured code. Simple individually, but the interactions with `finally`, `using`, iterators, and async are where these trip people up.

## `break` — exit the innermost loop or `switch`

```csharp
foreach (var item in items)
{
    if (item.IsSentinel) break;   // leaves the foreach
    Process(item);
}

switch (status)
{
    case Ok:     Handle(); break; // leaves the switch
    case Error:  throw new InvalidOperationException();
}
```

Rules:
- **Innermost only**. `break` in a nested loop exits one level. C# has no `break outerLabel;`.
- **In a `switch` statement** each non-empty case must end with `break`, `return`, `throw`, `continue`, or `goto` — implicit fall-through is a compile error (that's the point).
- **Not valid inside a `switch` expression** — arms are expressions.

To break out of nested loops cleanly, two options:

```csharp
// Option 1 — goto + label (ugly but explicit)
foreach (var row in rows)
{
    foreach (var cell in row)
    {
        if (cell.IsTarget) { target = cell; goto Found; }
    }
}
Found:
Process(target);

// Option 2 — extract to a method and return (preferred)
Cell? FindTarget(IEnumerable<Row> rows)
{
    foreach (var row in rows)
        foreach (var cell in row)
            if (cell.IsTarget) return cell;
    return null;
}
```

Option 2 is the "right" answer in review: it converts the escape into a normal method return, which composes with `null`/`Result<T>`/exceptions. `goto` survives only in very hot generated code.

## `continue` — skip to the next iteration

```csharp
foreach (var user in users)
{
    if (!user.IsActive) continue;   // go to next iteration
    Notify(user);
}

for (int i = 0; i < n; i++)
{
    if (i % 10 == 0) continue;      // jumps to `i++`, then condition
    Work(i);
}
```

In `for`, `continue` jumps to the **increment step** first, then the condition. That detail matters if you write funky increments.

`continue` in a nested loop skips the **innermost** loop only, same as `break`.

## `return` — exit the method

```csharp
int Abs(int x)
{
    if (x >= 0) return x;
    return -x;
}
```

### `return` and `finally` / `using`

`return` doesn't skip cleanup. `finally` blocks run, `using`-scoped resources are disposed, `lock` monitors are released:

```csharp
int Read()
{
    using var file = File.OpenText("log.txt");
    try
    {
        return file.ReadLine()?.Length ?? 0;
    }
    finally
    {
        Console.WriteLine("cleanup runs before caller sees the return");
    }
}
```

Order of observable effects: body produces value → `finally` runs → `using`'s compiler-generated `finally` runs (dispose) → value returned. A `return` inside a `finally` block is a **compile error** (CS0157) — it would swallow in-flight exceptions.

### `return` in `async` methods

An `async` method's `return value;` feeds the `SetResult(value)` call on the method's builder (`AsyncTaskMethodBuilder<T>`). The caller's awaiting `Task` completes with that value.

```csharp
async Task<int> FetchAsync(CancellationToken ct)
{
    using var http = new HttpClient();
    var s = await http.GetStringAsync(url, ct);
    return s.Length;    // sets Task<int>.Result
}
```

`return;` in an `async Task` method (no `<T>`) just completes the returned `Task`. `async void` also supports `return;` but should only be used for event handlers — any exception goes to the current `SynchronizationContext`'s unobserved path and can crash the process.

### Exception wrapping

`await` unwraps `AggregateException` to its first inner exception. `Task.Result` / `.Wait()` do **not** unwrap. So:

```csharp
try { await t; }                      catch (InvalidOperationException) { }  // typical
try { var v = t.Result; }             catch (AggregateException ae)     { }  // wrapped
```

### `return ref` / `return ref readonly`

```csharp
ref int First(int[] arr) => ref arr[0];           // ref return
ref readonly int Peek(int[] arr) => ref arr[0];   // ref readonly return (C# 7.2+)
```

Ref returns let you hand back a live slot without copying. The caller can mutate via `ref var slot = ref First(arr); slot = 42;`. Only valid when the returned ref refers to something that outlives the callee — locals do not qualify; fields, array elements, and passed-in `ref`s do. See [ref return, ref struct](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md) when populated.

## `goto` — the rehabilitated one

C# has three `goto` forms:

```csharp
// 1) Label
Retry:
    if (!TryConnect()) { Thread.Sleep(100); goto Retry; }

// 2) goto case / goto default  — only in switch statements
switch (n) { case 0: goto default; default: …; break; }

// 3) goto case N to chain case bodies (see if-switch doc)
```

**When `goto` is acceptable:**
- Coded state machines where labels *are* the states.
- Fall-through in a `switch` statement — that's exactly what `goto case N` is for.
- Escaping deeply nested loops when extracting a method is genuinely worse (rare).

**When it's wrong:** anywhere you can write an early `return`, a `break`, or a method extraction.

You cannot `goto` **into** a `try`, `using`, or scope that would bypass initialization — the compiler blocks it (CS0159/CS0158).

## `yield return` / `yield break`

These are not loop controls — they're iterator-method controls. See [Loops → Iterators](./loops.md#iterators--yield-return--yield-break).

```csharp
IEnumerable<int> Evens(IEnumerable<int> xs)
{
    foreach (var x in xs)
    {
        if (x < 0) yield break;            // end the sequence
        if (x % 2 == 0) yield return x;    // emit, suspend
    }
}
```

`yield break` ends the sequence normally (not an exception). `return` in an iterator method is a compile error — you use `yield break` instead, because a plain `return value;` can't coexist with `yield return value;` (the method produces an enumerable, not a `T`).

## Interaction summary

| Construct | Effect of `break` | Effect of `continue` | Effect of `return` |
|---|---|---|---|
| `for` / `while` / `do-while` | Exit loop | Go to next iteration (for: increment first) | Exit method (cleanup runs) |
| `foreach` | Exit loop + dispose enumerator | Advance enumerator | Exit method + dispose enumerator + run `using`/`finally` |
| `switch` statement | Exit switch (not outer loop) | Continue **outer** loop, skip rest of switch | Exit method |
| `switch` expression | Not valid | Not valid | N/A — arms are expressions |
| `try` (inside a loop) | Exit loop after `finally` runs | Next iteration after `finally` | Exit method after `finally` |
| iterator method | Ends the enumerable (acts like `yield break`) | Same loop semantics | Compile error — use `yield break` |

## Senior-level gotchas

- **`break`/`return` don't skip `finally`.** Cleanup always runs. Placing `return` inside a `try` is fine — just don't write side-effecting code in a `finally` you didn't intend to run on every path.
- **`continue` in a `for` loop still runs the increment expression.** Easy to miss when you're debugging "why does `i` keep going up?"
- **`break` in a `switch` inside a loop exits the switch, not the loop.** Use a flag or extract to a method if you want the loop out.
- **`return` from inside a `lock` releases the monitor** (compiler wraps the body in `try/finally`). Same for `using`. No manual unwinding needed.
- **`async void` + exception = unobserved crash.** Use `async Task` except for event handlers. The rare event-handler case should wrap its body in a `try/catch` that logs rather than rethrows.
- **`await`-ing an already-faulted `Task` rethrows the original exception** (unwrapped). Synchronously reading `.Exception` gives you the `AggregateException` — different APIs, different unwrap behavior.
- **`yield break` is not a control-flow bug — it's how you end an iterator.** Don't refactor it into `return` hoping to "clean up" — iterators don't support `return value;`.
- **`goto` into a `try` block is a compile error.** `goto` out of one runs the `finally`. Useful mental model: `goto` is allowed wherever the cleanup unwinding is well-defined.
- **Ref returns don't extend local lifetime.** Returning `ref` to a stack local or a ref-struct field is a compile error — the compiler's escape analysis catches it.
