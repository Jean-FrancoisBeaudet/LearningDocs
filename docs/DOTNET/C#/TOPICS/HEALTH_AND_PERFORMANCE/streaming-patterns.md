# Streaming patterns

_Targets .NET 10 / C# 14. See also: [System.IO.Pipelines](./system-io-pipelines.md), [Channel&lt;T&gt;](./channel-of-t.md), [Backpressure handling](./backpressure-handling.md), [Pagination strategies](./pagination-strategies.md), [Async streams and `await foreach`](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md), [Relational database query optimization](./relational-database-query-optimization.md), [Serialization overhead](./serialization-overhead.md)._

The streaming principle in one sentence: **peak memory should be bounded by chunk size, not by input size**. Materializing a full result set into a `List<T>`, a `MemoryStream`, or a `string` before handing it to the next layer is the anti-pattern that turns "small request returning a lot of data" into RSS spikes, GC pressure, and time-to-first-byte cliffs. .NET 6+ ships a coherent toolkit (`IAsyncEnumerable<T>`, `PipeReader`, `Channel<T>`, `JsonSerializer.DeserializeAsyncEnumerable`) for streaming end-to-end; the engineering work is keeping the stream intact across every layer.

## The materialization smell

```csharp
public async Task<List<Order>> GetAllAsync(CancellationToken ct) {
    return await _db.Orders.ToListAsync(ct);            // <-- materialization
}

public async Task<string> ReadFileAsync(string path) =>
    await File.ReadAllTextAsync(path);                  // <-- materialization

public async Task<MyDto[]> ParseJsonAsync(Stream s) =>
    (await JsonSerializer.DeserializeAsync<MyDto[]>(s))!; // <-- materialization
```

Each line above forces peak memory to **the size of the entire input**. For a 100k-row table or a 50 MB JSON document, that's an LOH allocation, a Gen-2 promotion, and a TTFB equal to "load it all then send it." The streaming alternative for each is one method call away.

## The four streaming surfaces

| Surface | Use for | Source-side | Sink-side |
|---|---|---|---|
| `IAsyncEnumerable<T>` | Already-parsed objects, pull-based | `yield return` in `async` method | `await foreach` |
| `Stream` / `PipeReader` | Bytes, byte-stream parsers | `Pipe`, network/file `Stream` | Manual frame parsing or `JsonSerializer.DeserializeAsyncEnumerable` |
| `Channel<T>` | Decoupled producer/consumer of objects | `WriteAsync` in producer | `await foreach (... ReadAllAsync(ct))` in consumer |
| Sourced `IAsyncEnumerable<T>` | Stream from a stream-aware source | EF Core `AsAsyncEnumerable`, `JsonSerializer.DeserializeAsyncEnumerable<T>`, `File.ReadLinesAsync` | `await foreach` |

The first one is the base abstraction; the others are specializations or adapters.

## Producing an `IAsyncEnumerable<T>`

```csharp
public async IAsyncEnumerable<Row> ReadAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    while (await reader.ReadLineAsync(ct) is { } line)
    {
        yield return Row.Parse(line);   // hands one item to consumer; resumes on next pull
    }
}
```

Two non-obvious pieces:

- **`[EnumeratorCancellation]`** is required so that the token passed to `WithCancellation(ct)` on the consumer's side actually flows into your `async` method. Without it, the consumer's token is silently ignored.
- The compiler rewrites the method into a state machine that suspends on every `yield return` and resumes on `MoveNextAsync`. **No `List<T>` ever exists** — peak memory is one `Row` plus the read buffer.

## Consuming with `await foreach`

```csharp
await foreach (var row in source.ReadAsync(path, ct).ConfigureAwait(false))
    await Process(row, ct);
```

`.ConfigureAwait(false)` on the **enumerable**, not on each item — `IAsyncEnumerable<T>` exposes `ConfigureAwait` directly because the foreach expansion needs to apply the configuration to every implicit `await`. Library code should always use it for the same reason it does on `await`.

## ASP.NET Core: stream all the way to the wire

```csharp
app.MapGet("/orders", (CancellationToken ct, OrderService svc) =>
    svc.StreamAllAsync(ct));   // returns IAsyncEnumerable<Order>

// Or in a controller:
[HttpGet]
public IAsyncEnumerable<Order> Get(CancellationToken ct) =>
    _svc.StreamAllAsync(ct);
```

ASP.NET Core's `System.Text.Json` integration knows how to serialize `IAsyncEnumerable<T>` element-by-element directly to the response body. The wire bytes start flowing on the first yielded element — TTFB is the time to fetch one row, not the time to fetch all of them. Memory peaks at one `Order` plus the JSON writer buffer, no matter how many rows the source produces.

The same shape works in **gRPC server streaming** (`IAsyncEnumerable<TResponse>` return type) and **SignalR streaming hubs** (`ChannelReader<T>` or `IAsyncEnumerable<T>`). All three runtimes consume the abstraction natively.

## DB layer

```csharp
// EF Core: row-by-row, no buffer.
public IAsyncEnumerable<Order> StreamAllAsync(CancellationToken ct) =>
    _db.Orders.AsNoTracking().AsAsyncEnumerable();

// Dapper: explicit unbuffered query.
public async IAsyncEnumerable<Order> StreamAllAsync(
    [EnumeratorCancellation] CancellationToken ct = default) {
    await using var conn = await _ds.OpenConnectionAsync(ct);
    await foreach (var o in conn.QueryUnbufferedAsync<Order>("select * from orders").WithCancellation(ct))
        yield return o;
}
```

Both bypass the row-buffer that the default `ToListAsync` / `Query<T>` materialize. The DB connection stays open for the duration of consumption — pair with `AsNoTracking()` (EF) and a short consumer to avoid holding a connection longer than the connection pool can tolerate.

## JSON streaming

```csharp
// Root must be a JSON array; each element is yielded as it parses.
await foreach (var item in JsonSerializer
    .DeserializeAsyncEnumerable<MyDto>(stream, options, ct))
{
    await Handle(item, ct);
}
```

`DeserializeAsyncEnumerable<T>` parses incrementally — peak memory is one `MyDto` plus the parser's lookahead buffer. The constraint: the JSON root must be an array. Object roots with an embedded array property require manual `Utf8JsonReader` driving (see the BCL docs).

For **producing** a streaming JSON response from an `IAsyncEnumerable<T>`, ASP.NET Core's response writer handles it for free. For ad-hoc serialization, `JsonSerializer.SerializeAsync(stream, asyncEnumerable, options, ct)` writes element-by-element.

## File / network

```csharp
// Lines, not bytes — the stream stays open.
await foreach (var line in File.ReadLinesAsync(path, ct))
    await Process(line, ct);

// Bytes, with explicit buffer size.
await source.CopyToAsync(destination, bufferSize: 81920, ct);
```

`File.ReadLinesAsync` (.NET 7+) is the streaming counterpart to `File.ReadAllLinesAsync` — same arguments, fundamentally different memory profile. `Stream.CopyToAsync(destination, bufferSize)` is the standard byte-stream pump; default buffer is 81920 bytes, deliberately under the LOH threshold.

## Composition

LINQ over `IAsyncEnumerable<T>` is **not built into the BCL**. Either:

- Add the `System.Linq.Async` NuGet package — gives you `Where`, `Select`, `Take`, `GroupBy`, etc. on `IAsyncEnumerable<T>`.
- Or write `await foreach` loops with explicit yields — fine for small compositions, awkward for chains.

```csharp
await foreach (var batch in source
    .Where(x => x.Active)
    .Select(x => Transform(x))
    .Buffer(100)                  // System.Linq.Async or Reactive
    .WithCancellation(ct))
{
    await SaveBatch(batch, ct);
}
```

Avoid `await foreach` over a `Where(x => SomeAsync(x))`-style filter that itself awaits — `System.Linq.Async` has `WhereAwait` for that case; using the sync `Where` with a captured `Task` triggers a closure allocation per item AND breaks back-pressure.

## When to materialize

Streaming is a default; materialization is a deliberate choice:

| Materialize when | Why |
|---|---|
| You need the full count up front | `Count()` on a stream consumes it |
| You need to iterate twice | Streams are single-use |
| You need to sort across the whole set | Sorting requires materialization |
| The result fits in a fixed small budget | Stream overhead exceeds the alloc |
| You'll cache the result | Caching a stream is meaningless |
| The next layer can't consume `IAsyncEnumerable<T>` | Boundary forces materialization |

For "small budget," small means ~1 KB total or a known fixed N (paging size, top-N query). For everything else, default to streaming.

## Anti-patterns

| Anti-pattern | What breaks |
|---|---|
| `await foreach (var x in source) list.Add(x); return list;` | Re-materializes after the source streamed it. Just return `source`. |
| `var list = await stream.ToListAsync(ct);` then iterate | Defeats streaming entirely; pay alloc + GC for nothing |
| Server endpoint that returns `List<T>` after a long query | TTFB = full query time; bytes flow only once everything is fetched |
| `IEnumerable<T>` middle layer between two `IAsyncEnumerable<T>` stages | Synchronous boundary forces buffering / blocking — kills the stream |
| `JsonSerializer.Deserialize<T[]>(stream)` for large payloads | Materializes; use `DeserializeAsyncEnumerable<T>` |
| `File.ReadAllLinesAsync` for a multi-GB log file | Allocates `string[]` covering the whole file — likely OOM |
| Streaming through an EF `DbContext` for the lifetime of an HTTP response | Connection pool starvation; the DbContext outlives its scope |
| Returning an `IAsyncEnumerable<T>` from a method that opens a stream synchronously, no `[EnumeratorCancellation]` | Token doesn't flow; cancellation silently dropped |

## Cancellation: the `[EnumeratorCancellation]` idiom

```csharp
public async IAsyncEnumerable<Row> StreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    // ct here receives the token passed via WithCancellation()
}

// Consumer side — pass a new token, not the one captured at iterator creation.
await foreach (var row in svc.StreamAsync().WithCancellation(ct))
    ...
```

Without `[EnumeratorCancellation]`, the parameter token is bound at iterator construction and the consumer's `WithCancellation(ct)` is silently ignored. This is the single most common bug in `IAsyncEnumerable` code — and the compiler does not warn.

## Observability

Streaming changes which numbers matter:

| Metric | Streaming target | Materializing target |
|---|---|---|
| Time to first byte | One row's processing time | Full query/parse time |
| Peak working set | Chunk size + buffer overhead | Result size |
| Allocation rate | Per-item, low constant | One large alloc per request (often LOH) |
| GC counts during request | Steady Gen-0 | Gen-2 promotion under load |
| End-user latency (large payloads) | Bytes overlap with processing | Sum of fetch + send |

Compare the streaming and materializing variants of the same endpoint with `dotnet-counters monitor System.Runtime` under load: `alloc-rate`, `gen-2-gc-count`, `working-set`, plus your own per-endpoint `Histogram<double>` for TTFB. The streaming variant flatlines the GC counters while the materializing one spikes them on every request.

## Senior-level gotchas

- **`[EnumeratorCancellation]` is the silent failure mode** of `IAsyncEnumerable<T>` methods. Forget it and the consumer's `WithCancellation(ct)` does nothing — the iterator runs to completion regardless of cancellation requests. There is no compiler warning. The Roslyn analyzer **CA2016** does flag the omission; enable it.
- **Streaming through an `IEnumerable<T>` boundary collapses the stream.** The synchronous `foreach` materializes whatever's behind it — even if the source was `IAsyncEnumerable<T>`. Boundaries between async and sync streams are buffering points; design pipelines without them.
- **Streaming an `IAsyncEnumerable<T>` from EF Core holds the DB connection open for the lifetime of consumption.** A slow consumer means a long-held connection — the pool starves. Either pull pages with `Skip/Take` and a closed connection between pages, or accept the open connection and bound the consumer's processing time.
- **`Stream.CopyToAsync` defaults to 81920 bytes** — chosen specifically to stay under the 85k LOH threshold for `byte[]`. If you override the buffer size, stay under it or rent from `ArrayPool<byte>` and pass the resulting array's `Length`.
- **`File.ReadAllLinesAsync` and `File.ReadAllBytesAsync` are landmines on large inputs.** They look streaming because they're async, but they materialize the entire file. Use `File.ReadLinesAsync` and `Stream`-based reads.
- **`JsonSerializer.DeserializeAsyncEnumerable<T>` requires the root to be a JSON array.** For `{"items": [...]}`-shaped responses you have to drive a `Utf8JsonReader` manually to descend to the array, then yield from there. The BCL doesn't ship a "root path" overload.
- **An `await foreach` consumer that throws abandons the iterator without disposing pooled resources** — but only if the iterator's `using` / `await using` blocks ran. The `IAsyncDisposable` pattern handles this correctly; hand-rolled iterators must guarantee `try`/`finally` around any rented buffers.
- **Streaming endpoints disable response buffering.** If your hosting platform (IIS, some reverse proxies) enables buffering by default, the streaming benefit evaporates at the proxy. Set `[DisableResponseBuffering]` (.NET 7+) or `Response.Headers.ContentLength = null` and verify in production with a packet capture.
- **`IAsyncEnumerable<T>` is single-iterate.** A consumer that calls `await foreach` twice on the same instance triggers the iterator twice — which usually means re-running the underlying query. The contract is single-iterate; signal it in API names (`StreamAsync`, not `GetAllAsync`).
- **`System.Linq.Async`'s `Buffer(n)` is the standard batching primitive over a stream** — yields `IList<T>` of size n. Don't hand-roll batching with a `List<T>` buffer in the consumer; the operator handles the partial-final-batch case correctly.
- **Streaming a JSON response with `WriteIndented = true` undoes the throughput win** — the writer flushes more often, the bytes-per-element grows, and the consumer's parser does more work. Default to `WriteIndented = false` on streaming endpoints.
- **`ChannelReader<T>.ReadAllAsync(ct)` is itself an `IAsyncEnumerable<T>`** — Channels and async streams compose for free. The choice between them is decoupling: if producer and consumer share a method, use `IAsyncEnumerable<T>`; if they live in different services / `BackgroundService`s, use `Channel<T>`.
- **TTFB is the user-visible win, not bytes.** A streaming endpoint sends fewer bytes per second under heavy load than a materializing one (no pre-fetch concurrency). The point is **time** — first byte arrives in ms instead of seconds. Choose streaming for latency-critical large payloads, not for raw throughput.
