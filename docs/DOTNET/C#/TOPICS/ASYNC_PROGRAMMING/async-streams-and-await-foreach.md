# Async Streams and `await foreach`

_Targets .NET 10 / C# 14. See also: [ConfigureAwait](./configureawait-and-synchronizationcontext.md), [ValueTask](./valuetask.md), [Channels](./channels.md)._

Async streams are the async equivalent of `IEnumerable<T>`: a producer yields elements over time, a consumer pulls them one at a time, and **both sides may `await`**. Introduced in C# 8 / .NET Core 3.0, they are built on `IAsyncEnumerable<T>` + `await foreach` and power modern streaming APIs (EF Core, `Channel<T>.ReadAllAsync`, gRPC server streaming, SignalR streaming hubs).

## The shapes

```csharp
public interface IAsyncEnumerable<out T>
{
    IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken ct = default);
}

public interface IAsyncEnumerator<out T> : IAsyncDisposable
{
    T Current { get; }
    ValueTask<bool> MoveNextAsync();
}
```

Note `MoveNextAsync` returns `ValueTask<bool>` — per-element pulls are a hot path; see [ValueTask](./valuetask.md) for why.

## Producing a stream

```csharp
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    DateOnly from,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var conn = await _pool.OpenAsync(ct);
    await using var reader = await conn.ExecuteReaderAsync(
        "select * from orders where date >= @from", new { from }, ct);

    while (await reader.ReadAsync(ct))
    {
        ct.ThrowIfCancellationRequested();
        yield return Order.FromReader(reader);
    }
}
```

The compiler turns this into a state machine that implements `IAsyncEnumerable<T>` + `IAsyncEnumerator<T>`. You cannot mix `yield return` with `async Task` — that combination is a compile error, by design.

### `[EnumeratorCancellation]`

This attribute wires the `CancellationToken` a consumer passes via `WithCancellation(ct)` into the token parameter of your iterator method. Without it, consumer-supplied tokens are silently ignored.

```csharp
// Consumer side:
await foreach (var order in StreamOrdersAsync(from).WithCancellation(ct).ConfigureAwait(false))
{
    Process(order);
}
```

Roslyn analyzer **CA2016** warns when `IAsyncEnumerable`-returning iterators have a `CancellationToken` parameter not marked `[EnumeratorCancellation]` — turn it on in library projects.

## Consuming a stream

```csharp
await foreach (var line in ReadLinesAsync(path, ct))
{
    Console.WriteLine(line);
}
```

`await foreach` de-sugars to a loop calling `MoveNextAsync` and `Current`, disposing the enumerator via `DisposeAsync` in a `finally`. Cancellation is cooperative — the iterator has to observe the token.

### `WithCancellation(ct).ConfigureAwait(false)`

This is the fully-specified form for library code:

```csharp
await foreach (var x in source.WithCancellation(ct).ConfigureAwait(false))
    ...
```

Without `ConfigureAwait(false)` on the enumerable, the `MoveNextAsync` awaits capture the sync context on each iteration. In UI or .NET Framework library code that matters; in ASP.NET Core app code it does not. The outer method's `ConfigureAwait` does **not** propagate into `await foreach` — you must apply it on the enumerable.

## The anti-pattern: `IEnumerable<Task<T>>`

```csharp
// ❌ This is NOT streaming. It's a synchronous enumeration of cold tasks.
public IEnumerable<Task<string>> DownloadAll(string[] urls)
{
    foreach (var url in urls) yield return _http.GetStringAsync(url);
}
```

Pitfalls:
- Enumeration is synchronous; there's no back-pressure.
- The moment you touch an element, its task has already been *created* but not awaited — you lose the ability to order the awaits.
- Composing with LINQ (`.Select(x => x.Result)`) will block.

If you want concurrent fan-out, use `Task.WhenAll`. If you want streaming-as-it-arrives, use `IAsyncEnumerable<T>` or `Task.WhenAny` in a loop.

## Interop with other streaming primitives

### EF Core

```csharp
await foreach (var o in db.Orders.Where(o => o.Customer == id).AsAsyncEnumerable()
                                  .WithCancellation(ct))
{
    Process(o);
}
```

`AsAsyncEnumerable` defers materialization — useful for exports, batching, report pipelines. Avoid it on short result sets where a `ToListAsync` is simpler.

### Channels

```csharp
await foreach (var msg in channel.Reader.ReadAllAsync(ct))
    Handle(msg);
```

`Channel<T>.Reader.ReadAllAsync(ct)` is the idiomatic way to consume a channel — cleaner than the old `WaitToReadAsync` + `TryRead` loop.

### gRPC server streaming

```csharp
public override async Task StreamPrices(
    PriceRequest req, IServerStreamWriter<PriceTick> response, ServerCallContext ctx)
{
    await foreach (var tick in _feed.SubscribeAsync(req.Symbol).WithCancellation(ctx.CancellationToken))
        await response.WriteAsync(tick);
}
```

The `ctx.CancellationToken` fires when the client disconnects. Plumb it through `WithCancellation`.

## Back-pressure and buffering

`IAsyncEnumerable<T>` is inherently **pull-based**: the producer only advances when the consumer awaits `MoveNextAsync`. That gives you natural back-pressure — a slow consumer slows the producer.

For push-based scenarios (event streams, subscriptions) bridge through a `Channel<T>` and expose the reader via `ReadAllAsync`. See [channels.md](./channels.md).

## Combining with LINQ

`System.Linq.Async` (nuget `System.Linq.Async`) adds the usual operators (`Select`, `Where`, `Take`, `ToListAsync`) over `IAsyncEnumerable<T>`. C# 13+ included many of these as built-in extensions.

```csharp
var recent = await db.Orders
    .AsAsyncEnumerable()
    .Where(o => o.Total > 100)
    .Take(50)
    .ToListAsync(ct);
```

**Senior-level gotchas:**
- Forgetting `[EnumeratorCancellation]` means your iterator's `CancellationToken ct` parameter is *always* `default`, even if the caller passes one via `WithCancellation`. Silent — no exception, no warning without CA2016.
- `await foreach` disposes the enumerator via `DisposeAsync` in a generated `finally`. If your enumerator owns a disposable connection (e.g., the `DbDataReader` above), an exception mid-iteration cleans up correctly — but only if you use `await using` inside the iterator.
- The enumerable returned from an `async` iterator method is **cold**: calling the method does nothing until someone calls `GetAsyncEnumerator`. Storing it for later is safe; calling it twice starts two independent iterations (and two connections in the SQL example).
- `IAsyncEnumerable<T>` is not covariant in the awaiter — returning `IAsyncEnumerable<Derived>` from a method that declares `IAsyncEnumerable<Base>` is fine (the interface is `out T`), but the underlying enumerator type is invariant, so casting a materialized enumerator is not.
- `await foreach` does **not** capture `ExecutionContext` separately per iteration — the iterator state machine runs under the caller's EC.  Setting `AsyncLocal<T>` inside the iterator and observing it outside won't work.
- Consuming an `IAsyncEnumerable` twice in parallel with `.Skip(n)` / `.Take(n)` tricks will execute the iterator twice. For broadcasting one source to many consumers, fan out through a `Channel<T>` instead.
