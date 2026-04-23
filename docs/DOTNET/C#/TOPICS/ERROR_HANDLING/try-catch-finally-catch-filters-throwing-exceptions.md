# `try`, `catch`, `finally`, catch filters, throwing exceptions

_Targets .NET 10 / C# 14. See also: [Pattern matching & expressions](../EXPRESSIONS/member-access-type-testing-pattern-matching-collection-expressions.md), [using and lock](../CONSTRUCTS/using-and-lock.md), [async/await](../CONSTRUCTS/async-await.md), [CancellationTokenSource & TaskCompletionSource](../ASYNC_PROGRAMMING/cancellationtokensource-and-taskcompletionsource.md), [Task continuation, cancellation, multiple tasks](../ASYNC_PROGRAMMING/task-continuation-cancellation-and-multiple-tasks.md), [CallerArgumentExpression](../GENERAL_TOPICS/caller-argument-expression.md)._

Exception handling looks simple — put stuff in `try`, write a `catch`, done. The hard parts are: (1) the **two-pass CLR model** that filters care about, (2) the stack-preservation rules around rethrow, (3) the interaction with `async` unwrapping and cancellation, and (4) the modern guard helpers that retired a decade of hand-written null checks.

## The shape

```csharp
try
{
    DoWork();
}
catch (IOException ex) when (ex.HResult == unchecked((int)0x80070020)) // sharing violation
{
    _log.LogWarning(ex, "File busy, skipping");
}
catch (IOException ex)
{
    _log.LogError(ex, "I/O failure");
    throw;
}
finally
{
    _telemetry.Flush();
}
```

Rules at a glance:
- Zero-or-more `catch` arms, matched **top to bottom by type**, then by `when` filter.
- At most one `finally`. `finally` always runs on normal exit *or* exception unwinding (with caveats — see §5).
- A `try` must have at least one `catch` or `finally`.
- `catch (T e)` requires `T` to derive from `System.Exception`. A bare `catch { }` is legal C# and is equivalent to `catch (Exception)` in modern .NET (since `RuntimeCompatibility(WrapNonExceptionThrows=true)` is the default on the current SDK).

## The two-pass CLR model — why filters matter

The CLR does exception dispatch in **two passes**:

1. **First pass (find handler)**: walk up the stack evaluating `catch` clauses and their `when` filters. No frames are unwound yet. If a filter throws, the exception is swallowed and treated as "filter did not match" — another reason filters should be boolean-pure.
2. **Second pass (unwind)**: once a handler is chosen, unwind the stack running each `finally` on the way, then jump into the chosen handler.

Compare:

```csharp
// ❌ Catch-inspect-rethrow: first pass chose this handler; stack is partially unwound by the time you throw again.
try { Work(); }
catch (DbException ex)
{
    if (!IsTransient(ex)) throw;       // rethrow — but the original context above the try is already gone
    Retry();
}

// ✅ Filter: first pass *rejects* this handler; stack remains frozen at the throw site for the next frame up.
try { Work(); }
catch (DbException ex) when (IsTransient(ex))
{
    Retry();
}
```

A post-mortem dump taken from inside the `catch` body shows partial unwinding; a dump triggered *after* a filter rejects shows the full original stack at the throw point. Debuggers, SOS, `!pe`, ETW — all benefit.

Side-effect trick — the "log everything without catching" filter:

```csharp
static bool Log(Exception ex) { Telemetry.Record(ex); return false; }

try { Work(); }
catch (Exception ex) when (Log(ex)) { /* unreachable — filter returned false */ }
```

The filter runs during pass 1, logs, returns `false`, and the runtime keeps searching. Stack stays intact for whichever frame above actually handles the exception.

## `catch` — type, variable, filter

```csharp
try { … }
catch (FileNotFoundException ex)         { /* most specific first */ }
catch (IOException ex) when (ShouldRetry(ex))
                                         { /* filter, runs in pass 1 */ }
catch (IOException)                      { /* no variable — fine if you don't need one */ }
catch (Exception ex) when (Log(ex))      { /* observe-and-keep-searching */ }
```

- **Order matters**: compile warning `CS0160` if an earlier arm already matches everything a later arm would. Put the most-derived type first.
- **Multi-catch isn't a thing in C#** (unlike Java's `catch (A | B)`). Use a filter: `catch (Exception ex) when (ex is A or B) { }`.
- **Bare `catch`** catches *any* thrown object, including pre-.NET 2.0 C++/CLI non-CLS exceptions, which the CLR wraps in `RuntimeWrappedException`. In practice you never need bare catch — use `catch (Exception)`.

## Catch filters (`when`) — patterns welcome

`when` takes any boolean expression, and C# 7+ pattern matching is first-class inside it:

```csharp
try { await http.SendAsync(req, ct); }
catch (HttpRequestException { StatusCode: HttpStatusCode.TooManyRequests } ex)
{
    await Task.Delay(RetryAfter(ex), ct);
}
catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)
{
    // any 5xx
}
```

Guidance:
- Keep filters **pure and cheap** — they run during pass 1, before any cleanup. A filter that throws is silently swallowed.
- Property patterns (`{ StatusCode: 429 }`) on the caught type are legal in the *catch type clause itself* — you don't need to duplicate them in a `when`.
- Don't combine `when` with catch-all `Exception` just to write `if/else` — write normal code in the body instead. `when` earns its place by keeping the first-pass stack intact.

## `finally` — guarantees and limits

Runs on:
- Normal exit from the `try` block (including `return`, `break`, `continue`, `goto`).
- Exception propagation through the `try` block, after pass 2 selects a handler further up.
- **Even when no matching handler exists** in the process — the CLR unwinds each frame's `finally` during pass 2, then the process is terminated if nothing catches.

Does **not** run on:
- `StackOverflowException` (since .NET 2.0 — uncatchable, process dies).
- `Environment.FailFast()` — the whole point is "skip cleanup, crash now".
- The async method that never resumes (e.g. awaiting a task that never completes — the state machine is garbage-collected; any `finally` in its body is abandoned).
- Unhandled exceptions on a *background* thread pre-.NET 4 (today they fault the process).
- `ThreadAbortException` — **not applicable** on .NET Core / .NET 5+; `Thread.Abort` throws `PlatformNotSupportedException`.

Don't rely on `finally` for security invariants — use `SafeHandle` and the Dispose pattern, which cooperate with the GC's critical finalization path.

## `using` and `lock` as syntactic `try`/`finally`

Both keywords compile to `try`/`finally` (see [using and lock](../CONSTRUCTS/using-and-lock.md)). Prefer them over hand-written cleanup:

```csharp
// Hand-rolled — verbose, easy to get disposal order wrong
var s = File.OpenRead(p);
try { … } finally { s.Dispose(); }

// Idiomatic
using var s = File.OpenRead(p);
```

Same for mutual exclusion: `lock (gate) { … }` lowers to `Monitor.Enter`/`try`/`finally`/`Monitor.Exit`. Don't do this dance by hand.

## Throwing — `throw` vs `throw ex` vs `throw new`

```csharp
catch (Exception ex)
{
    throw;           // ✅ preserves the original stack trace (IL: rethrow)
    throw ex;        // ❌ resets stack to this IL offset (IL: throw ldloc.0)
    throw new InvalidOperationException("wrap", ex);  // ✅ new exception, original chained via InnerException
}
```

The IL difference is one instruction: `rethrow` vs `throw`. `rethrow` preserves the captured `StackTrace`; `throw` builds a new one starting here. Analyzer `CA2200` flags `throw ex;`.

When you genuinely want to *translate* an exception (hide a DB exception behind a domain one, for example), `throw new T("…", ex);` is the right move — the caller gets your stable exception type and can still dig into `InnerException` for diagnostics.

## `ExceptionDispatchInfo` — rethrow preserving the stack across boundaries

`throw;` only preserves the stack when you're still inside the `catch` block. If you capture the exception and rethrow later (retry queue, TPL continuation, bridge to another thread), use `ExceptionDispatchInfo`:

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? failure = null;

try { await DoAsync(ct); }
catch (Exception ex) { failure = ExceptionDispatchInfo.Capture(ex); }

// …later, possibly another thread…
failure?.Throw();   // original stack + type preserved, appended with "--- End of stack trace from previous location ---"
```

Since .NET 5, `ExceptionDispatchInfo.SetCurrentStackTrace(ex)` lets you attach "here's where I'm surfacing it" context to a pre-built exception without a `try/catch` dance. Handy when your factory method produces the exception in advance.

## `throw` expression (C# 7+)

`throw` can appear where an expression is expected:

```csharp
public Order(Customer customer, decimal total)
{
    Customer = customer ?? throw new ArgumentNullException(nameof(customer));
    Total    = total >= 0 ? total : throw new ArgumentOutOfRangeException(nameof(total));
}

public string Label => _label ?? throw new InvalidOperationException("Label not set");
```

Valid positions: right-hand side of `??`, ternary arms, expression-bodied members, lambdas with an expression body, switch-expression arms.

## Exception hierarchy & guidance

```
Object
└── Exception
    ├── SystemException      (runtime-thrown; don't derive from this)
    │   ├── ArgumentException
    │   │   ├── ArgumentNullException
    │   │   └── ArgumentOutOfRangeException
    │   ├── InvalidOperationException
    │   │   └── ObjectDisposedException
    │   ├── NotSupportedException
    │   ├── FormatException
    │   ├── OperationCanceledException
    │   │   └── TaskCanceledException
    │   ├── IOException, …
    │   └── NullReferenceException, IndexOutOfRangeException, … (runtime-only, don't throw manually)
    └── ApplicationException (deprecated split — don't derive from this)
```

Rules:
- **Throw the most specific existing type** before inventing your own.
- `ApplicationException` vs `SystemException` was a .NET 1.x design that `.NET` itself gave up on — don't base your own hierarchy on it.
- Never throw `NullReferenceException` or `IndexOutOfRangeException` manually — they imply a runtime bug to callers.

## Modern guard helpers (.NET 6 → .NET 8)

Prefer the BCL static `ThrowIf…` helpers — they use `[CallerArgumentExpression]` to generate the parameter name automatically (see [CallerArgumentExpression](../GENERAL_TOPICS/caller-argument-expression.md)):

```csharp
public Order Place(Customer customer, string couponCode, int quantity, Connection connection)
{
    ArgumentNullException.ThrowIfNull(customer);                           // .NET 6
    ArgumentException.ThrowIfNullOrEmpty(couponCode);                      // .NET 7
    ArgumentException.ThrowIfNullOrWhiteSpace(couponCode);                 // .NET 8
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);           // .NET 8
    ArgumentOutOfRangeException.ThrowIfGreaterThan(quantity, MaxQty);      // .NET 8
    ObjectDisposedException.ThrowIf(connection.IsClosed, connection);      // .NET 7

    …
}
```

All of these produce the exact same error messages as hand-rolled `if (x is null) throw new ArgumentNullException(nameof(x))` but they're shorter, consistent across the BCL, and the parameter name comes from the caller site rather than a hand-written `nameof`.

## Custom exceptions

Create one only when callers need to **programmatically** distinguish the failure from existing BCL types. Idiomatic shape on modern .NET:

```csharp
public sealed class OrderPolicyException : Exception
{
    public OrderPolicyException() { }
    public OrderPolicyException(string message) : base(message) { }
    public OrderPolicyException(string message, Exception inner) : base(message, inner) { }

    public string PolicyCode { get; init; } = "";
}
```

Notes:
- `sealed` unless you explicitly design a hierarchy.
- Three constructors (parameterless / message / message+inner) covers framework expectations for `Activator.CreateInstance` and reflection-based deserialization.
- **Skip the `SerializationInfo`/`StreamingContext` constructor.** `BinaryFormatter` is obsolete (`SYSLIB0011`) and the `protected Exception(SerializationInfo, StreamingContext)` ctor itself was marked `[Obsolete]` in .NET 8 (`SYSLIB0051`). Only keep it if you must interoperate with legacy binary serialization.
- Prefer record-like `init` properties for extra context — message, error code, correlation ID.

## Inner exceptions and `AggregateException`

```csharp
try { await LoadAsync(); }
catch (RepositoryException ex) when (ex.InnerException is DbException db && db.ErrorCode == -2)
{
    // SQL timeout, peeled from the wrap
}
```

`AggregateException` is how parallel/TPL surface multiple failures:

```csharp
var all = Task.WhenAll(tasks);
try
{
    await all;   // await unwraps to the *first* exception
}
catch (Exception)
{
    // To see all failures, look at the Task.Exception.InnerExceptions:
    foreach (var e in all.Exception!.Flatten().InnerExceptions)
        Log(e);
}
```

`AggregateException.Flatten()` collapses nested aggregates into a single-level one — useful when `Task.WhenAll` aggregates of aggregates (nested continuations) pile up.

## Cancellation — `OperationCanceledException` / `TaskCanceledException`

The cancellation protocol:

```csharp
public async Task<Report> BuildAsync(CancellationToken ct)
{
    ct.ThrowIfCancellationRequested();
    var data = await _repo.LoadAsync(ct);
    return Shape(data);
}
```

- `CancellationToken.ThrowIfCancellationRequested()` throws `OperationCanceledException` carrying the token. If the token is already cancelled when the method is called, the exception fires synchronously.
- `Task`-producing APIs throw `TaskCanceledException : OperationCanceledException` to distinguish "the task itself was cancelled" from "the current operation was cancelled" — but the `catch` pattern below treats them uniformly.
- **Framework code re-throws `OperationCanceledException` unchanged** — don't swallow it, don't wrap it.

Distinguish user-requested cancellation from a stray cancellation:

```csharp
try { await WorkAsync(ct); }
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // Clean exit — the caller asked to stop
}
catch (OperationCanceledException ex)
{
    // Some *other* token cancelled us — likely a bug or a stale linked token
    _log.LogWarning(ex, "Unexpected cancellation");
    throw;
}
```

More on this in [CancellationTokenSource & TaskCompletionSource](../ASYNC_PROGRAMMING/cancellationtokensource-and-taskcompletionsource.md).

## Async exception behavior (recap)

Fast version — depth in [async/await](../CONSTRUCTS/async-await.md):

- Exceptions thrown inside an `async Task` method surface on `await`, not synchronously at the call site.
- The returned `Task` becomes faulted; observing it via `await` unwraps one exception; `.Result`/`.Wait()` wraps in `AggregateException`.
- For **eager** validation use the iterator-wrapper idiom:

```csharp
public Task<Account> GetAsync(string id, CancellationToken ct)
{
    ArgumentException.ThrowIfNullOrEmpty(id);   // fires synchronously
    return Core(id, ct);

    static async Task<Account> Core(string id, CancellationToken ct) { … }
}
```

## First-chance vs second-chance exceptions

```csharp
AppDomain.CurrentDomain.FirstChanceException += (_, e) =>
    Telemetry.RecordThrow(e.Exception);
```

First-chance fires **when the exception is constructed and thrown**, *before* any handler runs — so even caught-and-handled exceptions fire it. Second-chance ("unhandled") is `AppDomain.UnhandledException` / `TaskScheduler.UnobservedTaskException`, which fire only when no handler wants it.

Use first-chance for observability only. Noisy `try/catch` (e.g. parsing `int.TryParse` equivalents the manual way) pollutes diagnostics — every throw still costs and still shows up.

## Performance — exceptions are not control flow

A thrown exception:
- Constructs an `Exception` (allocation + stack walk to fill `StackTrace`, which is lazy but eventually paid).
- Runs the CLR's two-pass EH, which is an order of magnitude slower than a branch.
- Disables some JIT optimizations in methods with many protected regions.

Order-of-magnitude guideline: a throw/catch pair costs microseconds; a branch costs nanoseconds. Recent CoreCLR releases (notably .NET 7/8) reduced the constant, but the ratio remains.

Rule: **`TryXxx` for expected failures, exceptions for exceptional ones**. `int.TryParse`, `Dictionary<K,V>.TryGetValue`, `ConcurrentDictionary.TryRemove`, `FrozenDictionary.TryGetValue` — all preserve the anti-pattern escape hatch. In hot loops, reach for those first.

## `throw` inside `finally`

```csharp
// ❌ anti-pattern
try { Work(); }
finally { Cleanup(); }   // if Cleanup throws, the original exception is lost
```

An exception escaping a `finally` **replaces** the one currently propagating. If `Cleanup` can realistically fail, catch-log-suppress inside it:

```csharp
try { Work(); }
finally
{
    try   { Cleanup(); }
    catch (Exception ex) { _log.LogError(ex, "Cleanup failed (suppressed)"); }
}
```

## Filters vs catch-and-rethrow — side by side

| Property | `catch (T) when (…)` | `catch (T) { if (!…) throw; }` |
|---|---|---|
| Runs in | Pass 1 (discovery) | Pass 2 (after unwind to this frame) |
| Stack at filter/body | Intact up to throw site | Partially unwound |
| Visible in post-mortem | Yes — throw site preserved | Throw site lost |
| Cost of rejection | A boolean + continue search | A full rethrow + second dispatch |
| Preferred for observability | ✅ | ❌ |

## Senior-level gotchas

- **`throw ex;` resets the stack** — `CA2200` flags it. `throw;` preserves. `throw new T(msg, ex);` is a *translation*, not a rethrow.
- **`catch (Exception)` without a filter + rethrow** is almost always wrong — it still unwinds to this frame on pass 2, losing post-mortem fidelity. Use a `when` filter or skip the catch.
- **Filters must be pure and cheap** — a filter that throws is silently swallowed ("filter didn't match"), and side-effecting filters run on every pass-1 probe.
- **`finally` doesn't run on stack overflow or `FailFast`** — don't use it for security invariants. Use `SafeHandle` for critical cleanup.
- **`Task.WhenAll` faults aggregate, but `await` unwraps only the first.** Inspect `Task.Exception.InnerExceptions` or `AggregateException.Flatten()` to see all.
- **`OperationCanceledException` is "user cancelled" only when your `ct.IsCancellationRequested`** — otherwise some *other* token cancelled you, which is usually a bug or a stale linked token.
- **`ExceptionDispatchInfo.Throw()` is the right tool** when rethrowing from a different context (background thread, retry queue) — plain `throw ex;` loses the stack, plain `throw;` isn't available outside the original catch.
- **BinaryFormatter-based serialization ctors on exceptions are `[Obsolete]` in .NET 8** (`SYSLIB0051`). Drop them on new types.
- **`NullReferenceException` and `IndexOutOfRangeException` from your own code are bugs**, not failure modes to catch. Fix the root cause.
- **Eager argument validation in an `async Task` method surfaces on `await`, not at the call site** — wrap with a non-async outer method if you need synchronous throw.
- **Exception filters are the only way to inspect an exception without unwinding** — material for dump-friendly diagnostics.
- **Don't base custom exceptions on `ApplicationException`** — the BCL itself gave up on that split decades ago.
- **Avoid `try/catch` in hot paths** — even caught exceptions fire first-chance handlers and cost microseconds. Use `TryXxx` APIs.
