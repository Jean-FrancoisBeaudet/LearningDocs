# Iterators and `IAsyncDisposable`

_Targets .NET 10 / C# 14. See also: [Async streams, `await foreach`](../ASYNC_PROGRAMMING/async-streams-await-foreach.md), [using, lock](../CONSTRUCTS/using-lock.md), [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md)._

Iterators (`yield return` / `yield break`) are a compiler-generated state machine that implements `IEnumerable<T>` / `IEnumerator<T>` (or the async versions) from imperative code. `IAsyncDisposable` is the async complement to `IDisposable` — for resources whose cleanup has to `await` (DB connections closing a network session, async flushing, `AsyncServiceScope`). Async iterators sit at the intersection: they implement **both** `IAsyncEnumerator<T>` and `IAsyncDisposable`.

## Sync iterators — how `yield return` works

```csharp
public static IEnumerable<int> RangeLazy(int start, int count)
{
    for (int i = 0; i < count; i++)
        yield return start + i;
}
```

The C# compiler rewrites the method body into a nested class implementing `IEnumerable<int>`, `IEnumerator<int>`, `IDisposable`. Key properties:

- **Lazy evaluation.** Nothing runs until `MoveNext()` is called.
- **`GetEnumerator()` returns `this`** for the first consumer; subsequent calls allocate a new state machine. Cheap but not free.
- **Single-pass.** Re-enumerating an iterator re-executes the body from the top.
- **Thread-safe for `GetEnumerator()` callers** on iterators produced by iterator methods with no shared state — each caller gets its own machine.

### `yield break` and implicit cleanup

A `yield break` exits the iterator cleanly. A thrown exception from inside the iterator propagates to the `MoveNext` caller. `finally` blocks inside the iterator **do run** when the consumer disposes the enumerator — including via `foreach` which calls `Dispose` for you:

```csharp
IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);   // disposed when foreach ends
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}
```

If the consumer abandons the iterator (breaks out of `foreach`), the compiler-generated `Dispose` is still called — that's what `foreach` does implicitly. Manual iteration via `MoveNext`/`Current` without a `try/finally` leaks.

## `IEnumerable<T>` vs `IAsyncEnumerable<T>`

| Concept              | Sync                      | Async                              |
|----------------------|---------------------------|------------------------------------|
| Producer method      | `IEnumerable<T> X()`      | `async IAsyncEnumerable<T> X()`    |
| Yield token          | `yield return`            | `yield return` (inside `async`)    |
| Consumer             | `foreach`                 | `await foreach`                    |
| Cleanup              | `IDisposable.Dispose`     | `IAsyncDisposable.DisposeAsync`    |
| Cancellation         | not built in              | `[EnumeratorCancellation]`         |
| Added in             | C# 2                      | C# 8 / .NET Core 3.0               |

## Async iterators

```csharp
public static async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    while (!reader.EndOfStream)
    {
        ct.ThrowIfCancellationRequested();
        var line = await reader.ReadLineAsync(ct);
        if (line is null) yield break;
        yield return line;
    }
}

await foreach (var line in ReadLinesAsync("big.log", cts.Token))
    Console.WriteLine(line);
```

Key points:

- Method is `async`, return type is `IAsyncEnumerable<T>`. You cannot `return` a value.
- `yield return` **and** `await` are both legal. They can interleave.
- `[EnumeratorCancellation]` on the parameter marks it as the token the consumer passes through `WithCancellation(token)`:

```csharp
await foreach (var line in ReadLinesAsync("big.log").WithCancellation(cts.Token))
{ /* ... */ }
```

  The compiler threads the caller's token into the iterator parameter even if the caller invoked `ReadLinesAsync("big.log")` without passing one.

- The generated enumerator implements `IAsyncEnumerator<T>` + `IAsyncDisposable`. `await foreach` calls `DisposeAsync` after the loop body exits (normally or via exception).

### Cancellation semantics

Three tokens can be in play simultaneously:

1. The one passed as an **argument** when the iterator was invoked.
2. The one passed via **`.WithCancellation(token)`** at enumeration time.
3. Any token created **inside** the iterator.

With `[EnumeratorCancellation]`, the compiler creates a **linked** token so that `ThrowIfCancellationRequested` inside the iterator responds to all three. Without the attribute, only the argument token is honored, and `WithCancellation` does nothing useful.

## `IAsyncDisposable`

```csharp
public sealed class AsyncResource : IAsyncDisposable
{
    private SqlConnection? _conn;

    public async ValueTask DisposeAsync()
    {
        if (_conn is not null)
        {
            await _conn.DisposeAsync();
            _conn = null;
        }
        GC.SuppressFinalize(this);
    }
}
```

Rules:

- Method signature **must** be `public ValueTask DisposeAsync()`. Not `Task`, not `async void`, not generic.
- Idempotent by convention — calling twice should be safe.
- Implement **`IDisposable` too** when the resource is usable in synchronous callers. BCL types (SQL connection, Pipe, FileStream) do both so consumers can pick.
- `GC.SuppressFinalize(this)` if your type has a finalizer. Otherwise the object still gets promoted to the F-reachable queue.

### `await using` — sync vs async scope

```csharp
await using var resource = new AsyncResource();   // uses DisposeAsync at scope exit
using var file = File.OpenRead("a.txt");          // uses Dispose at scope exit
await using var conn = new SqlConnection(cs);     // uses DisposeAsync (SqlConnection has both)
```

`await using` is required when the disposable is only `IAsyncDisposable`. If a type implements both, `await using` uses `DisposeAsync`; `using` uses `Dispose`. Prefer `await using` in async code paths — sync disposal can block on I/O.

### `ConfigureAwait` on disposal

```csharp
await using (resource.ConfigureAwait(false))
{
    // ...
}
```

`ConfigureAwait(false)` on an `await using` pattern is available via extension methods (`Microsoft.Bcl.AsyncInterfaces` / `Microsoft.Extensions`); standard library has it on `ConfiguredAsyncDisposable`. In library code, always `ConfigureAwait(false)` for async disposal to avoid capturing sync contexts.

## Async iterators implement `IAsyncDisposable`

The compiler-generated enumerator of an `async IAsyncEnumerable<T>` method implements `IAsyncDisposable`. `await foreach` takes care of calling `DisposeAsync`, but **manual enumeration** must:

```csharp
var e = ReadLinesAsync("big.log").GetAsyncEnumerator(cts.Token);
try
{
    while (await e.MoveNextAsync())
        Use(e.Current);
}
finally
{
    await e.DisposeAsync();
}
```

Missing the `DisposeAsync` leaks whatever the iterator has open (`FileStream`, `SqlConnection`, `HttpClient` streams).

## `IDisposable` vs `IAsyncDisposable` — which to implement

- **Both**, when realistically callable from sync and async code. Expose each cleanup independently; don't call `Dispose()` from `DisposeAsync()` or vice versa with `.GetAwaiter().GetResult()` — that's a classic deadlock/sync-over-async anti-pattern.
- **Only `IAsyncDisposable`** when cleanup genuinely requires async work and sync disposal would have to block. `DbContext` in EF Core 3+, `ServiceProvider` as `AsyncServiceScope`, Kafka consumers.
- **Only `IDisposable`** when cleanup is synchronous — files, mutexes, memory rentals, unmanaged pointers.

## Exception flow in async iterators

A throw inside the iterator propagates through `MoveNextAsync` / `Current`:

```csharp
async IAsyncEnumerable<int> Tricky()
{
    yield return 1;
    throw new InvalidOperationException("boom");
}

await foreach (var x in Tricky())  // throws on the second MoveNextAsync
    Console.Write(x);                // prints 1 first
```

`await foreach` honors the exception as with sync `foreach` — enumerator is disposed, exception propagates. `finally` blocks inside the iterator run during `DisposeAsync`.

## Interleaving `yield` and `await`

```csharp
async IAsyncEnumerable<Page> FetchAll(HttpClient client, string url,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    string? next = url;
    while (next is not null)
    {
        var page = await client.GetFromJsonAsync<Page>(next, ct);
        yield return page!;
        next = page?.NextLink;
    }
}
```

Each `yield return` suspends the state machine. The next `MoveNextAsync` resumes from that point. `await` does the same. The compiler uses a single state machine class that tracks where to resume.

Allocation note: the state machine is a `class` (reference type) because it implements `IAsyncEnumerator<T>`. Unlike `async Task` methods whose state machine stays on the stack in the fast path, async iterators always allocate on first yield.

## Senior-level gotchas

- **`[EnumeratorCancellation]` is the default you almost always want.** Without it, `WithCancellation(token)` is silently ignored — a nightmare to debug, because cancellation "just doesn't work." Analyzer CS8425 warns if your iterator accepts a `CancellationToken` and doesn't mark it.
- **Manual enumeration without `finally` + `DisposeAsync`** leaks the state machine's captured resources. Always prefer `await foreach`; if you must pull manually, wrap in try/finally.
- **`yield` cannot appear in a `try` with a `catch`** (only `try/finally` is allowed). This is a compiler restriction — rearrange to `try/finally` + outer `try/catch`.
- **`yield` is illegal in `async void`.** Async iterators must return `IAsyncEnumerable<T>` / `IAsyncEnumerator<T>`.
- **Re-enumerating an async iterator re-runs it.** Each call to `GetAsyncEnumerator()` starts fresh. Cache materialized results (e.g., `.ToListAsync()`) if you need to walk twice.
- **The state machine is a `class`, allocated on first yield.** Micro-optimizing a hot iterator for zero-alloc is a dead end; consider returning `ValueTask<ReadOnlyMemory<T>>` batches instead.
- **`ConfigureAwait(false)` on `await foreach`** — use `EnumeratorCancellation`'s cousin: `foo.WithCancellation(ct).ConfigureAwait(false)`. Library code should always opt out of the sync context.
- **`DisposeAsync` in a hot path** can stall — library authors sometimes issue blocking network calls in disposal. Measure; move heavy work out of disposal or offer an explicit `CloseAsync`.
- **Mixing `IDisposable` + `IAsyncDisposable` with `await using`**: if both are implemented, `await using` calls `DisposeAsync`. `using` calls `Dispose`. Consumers in sync contexts must know which path they're on — type `DbContext` documents this explicitly.
- **Returning an iterator as `IEnumerable<T>` with deferred exceptions** is a smell. A caller iterating tomorrow sees an exception from five minutes ago. Validate eagerly at the method top (wrap in a non-iterator helper that checks args and calls the iterator).
- **`ConfiguredCancelableAsyncEnumerable<T>`** is the type you get from `.WithCancellation(ct).ConfigureAwait(false)`. Expose APIs that return `IAsyncEnumerable<T>` — let callers choose the configured wrapper.
- **Source-generator caveats**: analyzers targeting `MethodDeclarationSyntax` often miss iterator methods because the compile-generated body lives elsewhere. If you're authoring a generator that inspects `yield return`, walk the `SyntaxTree`, not semantic return types.
- **`await using var` with a null disposable** is a runtime NullReference. The null-check is on the consumer, not the syntax. Guard with `await using var x = maybeNull ?? NullAsyncDisposable.Instance;` when the disposable might be absent.
- **NativeAOT** handles async iterators fine, but the state machine's generic instantiation can produce surprisingly large binaries if you have many `async IAsyncEnumerable<T>` methods — measure with `--verbose` trim reports.
