# Backpressure handling

_Targets .NET 10 / C# 14. See also: [Channel&lt;T&gt;](./channel-of-t.md), [System.IO.Pipelines](./system-io-pipelines.md), [Streaming patterns](./streaming-patterns.md), [Channels (API surface)](../ASYNC_PROGRAMMING/channels.md), [Synchronization bottlenecks](./synchronization-bottlenecks.md), [I/O saturation](./io-saturation.md), [Thread contention](./thread-contention.md)._

Back-pressure is the discipline of **propagating slowness backwards** through a pipeline so that a fast producer does not run a slow consumer (or downstream service) into the ground. Without it, an unbounded queue between the two converts CPU/IO mismatch into memory growth — and the failure mode is OOM, hours after the rate divergence began. .NET ships four in-process primitives that implement back-pressure correctly; the engineering work is picking the right one and instrumenting it.

## The shape of the problem

```
producer ──► [ queue ] ──► consumer
   1k/s                       100/s
```

Without bounding the queue: working set climbs by ~900 items/sec, latency through the queue climbs in step. A few hours later: OOM, or `OutOfMemoryException` on a totally unrelated allocation that finally tipped the runtime over. The producer was never warned; the consumer never noticed. **An unbounded queue is the absence of back-pressure.**

The four in-process primitives below all share one structural pattern: a bounded buffer + an asynchronous wait point that returns to the producer when the buffer is full. That wait is the back-pressure.

## The four mechanisms

### 1. Bounded `Channel<T>` with `FullMode = Wait` — the default for objects

```csharp
private readonly Channel<Work> _queue = Channel.CreateBounded<Work>(
    new BoundedChannelOptions(capacity: 1024) {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = true,
    });

public async ValueTask EnqueueAsync(Work w, CancellationToken ct) =>
    await _queue.Writer.WriteAsync(w, ct);   // awaits when full — back-pressure
```

`WriteAsync` becomes a `ValueTask` that completes immediately when there's capacity, and **suspends** when full until the consumer drains below threshold. The producer naturally absorbs the consumer's slowness as wait time. See [`channel-of-t.md`](./channel-of-t.md) for sizing and full-mode semantics.

### 2. `System.IO.Pipelines` — the default for byte streams

```csharp
var pipe = new Pipe(new PipeOptions(
    pauseWriterThreshold:  64 * 1024,
    resumeWriterThreshold: 32 * 1024,
    minimumSegmentSize:    4096,
    useSynchronizationContext: false));

// Producer:
await writer.FlushAsync(ct);    // doesn't complete until consumed bytes < pause threshold
```

`pauseWriterThreshold` / `resumeWriterThreshold` give you back-pressure across a byte stream without semaphores. Kestrel, gRPC, and SignalR all sit on this. Detail in [`system-io-pipelines.md`](./system-io-pipelines.md).

### 3. `SemaphoreSlim` — a concurrency limiter

```csharp
private readonly SemaphoreSlim _gate = new(initialCount: 16, maxCount: 16);

public async Task<Result> CallAsync(Request r, CancellationToken ct) {
    await _gate.WaitAsync(ct);   // blocks when 16 are in flight
    try { return await _downstream.SendAsync(r, ct); }
    finally { _gate.Release(); }
}
```

Bounds the **in-flight count** of a resource (HTTP connections to one host, DB connections, expensive CPU work). Producers wait at the semaphore. Pair with a timeout (`WaitAsync(timeout, ct)`) so the wait itself can fail fast under sustained overload — otherwise you've replaced "queue grows" with "request threads pile up at the gate."

### 4. TPL Dataflow `BoundedCapacity` — multi-stage graphs

```csharp
var transform = new TransformBlock<Raw, Parsed>(Parse,
    new ExecutionDataflowBlockOptions { BoundedCapacity = 256, MaxDegreeOfParallelism = 4 });
var sink = new ActionBlock<Parsed>(SaveAsync,
    new ExecutionDataflowBlockOptions { BoundedCapacity = 64 });

transform.LinkTo(sink, new DataflowLinkOptions { PropagateCompletion = true });

await transform.SendAsync(raw, ct);   // awaits when transform is full
```

Use Dataflow when you have a **graph** with per-stage parallelism, joining, broadcasting. For a simple producer → consumer, a bounded `Channel<T>` is lighter and faster.

### Picking matrix

| Topology | Mechanism |
|---|---|
| In-process objects, single producer + single consumer | Bounded `Channel<T>` + `Wait` |
| Byte stream parsing (TCP, framed protocols) | `System.IO.Pipelines` |
| Bounding concurrency to a downstream resource (DB, HTTP host) | `SemaphoreSlim` |
| Multi-stage graph (parse → transform → sink) | TPL Dataflow with `BoundedCapacity` per block |
| Pull-based, single consumer naturally paces producer | `IAsyncEnumerable<T>` (no extra primitive needed) |
| Cross-process (network boundary) | HTTP 429 / gRPC `RESOURCE_EXHAUSTED` / Service Bus delivery limits |

## Worked example: HTTP ingest → background processor

```csharp
// Endpoint: enqueue with a bounded budget, surface 503 on sustained overload.
app.MapPost("/events", async (Event e, IIngestQueue q, CancellationToken ct) => {
    using var timeout = CancellationTokenSource.CreateLinkedTokenSource(ct);
    timeout.CancelAfter(TimeSpan.FromMilliseconds(200));   // wait budget
    try {
        await q.EnqueueAsync(e, timeout.Token);
        return Results.Accepted();
    }
    catch (OperationCanceledException) when (!ct.IsCancellationRequested) {
        return Results.StatusCode(503);    // queue-full backoff signal
    }
});
```

Design intent:

- The producer (request thread) **awaits** the channel up to a bounded budget (200 ms).
- If the budget elapses, return **503 Service Unavailable** so the load balancer / client retries against another instance or backs off.
- The background consumer drains at its natural rate; its slowness becomes producer wait time, then 503s — never unbounded memory.

The trade is explicit: under sustained overload, some requests fail fast instead of all requests slowing down. That's almost always the right call — failing fast is a recoverable signal; latency cliff isn't.

## `IHttpClientFactory` + Polly bulkhead — cross-instance back-pressure

```csharp
services.AddHttpClient<DownstreamClient>(c => c.BaseAddress = new("https://api.partner.com"))
    .AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, retry => TimeSpan.FromMilliseconds(100 * retry)))
    .AddPolicyHandler(Policy.BulkheadAsync<HttpResponseMessage>(
        maxParallelization: 32, maxQueuingActions: 64));
```

The **bulkhead** caps in-flight calls to one downstream service from this process. Once `maxParallelization + maxQueuingActions` is reached, new calls fail fast with `BulkheadRejectedException` — back-pressure the caller can react to. Combine with HTTP 429 / `Retry-After` from the downstream side to compose process-local and remote signals.

Note: `Microsoft.Extensions.Http.Resilience` (the modern resilience pipeline replacing classic Polly) provides the same primitive via `RateLimiter` integration; either is fine, but stick with one in a service.

## `IAsyncEnumerable<T>` — pull-based natural back-pressure

```csharp
await foreach (var item in producer.ReadAsync(ct))
    await Process(item, ct);
```

The consumer pulls — `MoveNextAsync` doesn't return until it asks. The producer can't run ahead. This is the cleanest back-pressure model when you have **one** consumer and don't need decoupling: no buffer, no sizing, no full-mode choice. See [`streaming-patterns.md`](./streaming-patterns.md).

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| `Channel.CreateUnbounded<T>` + ignoring queue depth | Working set grows with rate divergence; OOM in the night |
| `ConcurrentQueue<T>` with no upper bound | Same problem — back-pressure has to be *bounded* |
| `Task.Run(async () => Process(item))` per produced item | Thread-pool saturation; "async slowness" that's actually queue length |
| `BoundedChannelFullMode.DropWrite` without metering drops | Silent data loss indistinguishable from a working system |
| Adding `SemaphoreSlim` without a wait timeout | Threads pile up at the gate; "fast fail" replaced by "slow fail" |
| `lock (...)` over async work | `Monitor` doesn't release across awaits — incorrect AND defeats back-pressure |
| Buffering at the network boundary (`MemoryStream` for the request body) | Bypasses Pipelines back-pressure; client uploads at line rate, server holds it all |
| Producer ignores `503` / `429` and keeps hammering | Cross-process back-pressure exists only if both sides honor it |

## Observability

Without metrics, back-pressure is invisible — by design, the producer just waits a bit longer. Instrument three quantities:

```csharp
private static readonly Meter s_meter = new("MyApp.Ingest");
private static readonly Histogram<double> s_waitMs = s_meter.CreateHistogram<double>("ingest.write_wait_ms");
private static readonly Counter<long> s_dropped = s_meter.CreateCounter<long>("ingest.dropped");
private static readonly UpDownCounter<long> s_depth = s_meter.CreateUpDownCounter<long>("ingest.depth");
```

| Metric | What it tells you |
|---|---|
| `write_wait_ms` (p50/p99) | Producer wait time at the bounded primitive. Climbing = consumer falling behind. |
| `dropped` | Items lost on full (only meaningful for Drop modes). Non-zero = capacity loss. |
| `depth` | Approximate queue depth (enqueued − dequeued). Trend matters more than instantaneous value. |
| `consumer_latency_ms` | Per-item processing time. Multiplied by depth = recovery ETA. |

Alert on `write_wait_ms p99 > consumer_budget_ms` — that's the early warning, **before** producers start failing fast or memory starts climbing.

## Cross-process back-pressure

Within a process you await; across the network, awaiting wastes resources on both sides. The cross-process equivalent is **fail fast with a structured signal** the caller can act on:

| Boundary | Signal | Caller action |
|---|---|---|
| HTTP | `503 Service Unavailable`, `429 Too Many Requests` + `Retry-After` | Polly retry-with-backoff + jitter |
| gRPC | `StatusCode.ResourceExhausted`, `Unavailable` | Polly + `RpcException.Trailers` for backoff hint |
| Service Bus / RabbitMQ | Max delivery count, dead-letter on lock expiry | Consumer concurrency limit + DLQ inspection |
| Kafka | Consumer lag (`consumer_lag` metric) | Add partitions or scale consumers |

The architectural rule: **a process never accepts work it can't bound the latency of**. If accepting more work means unbounded queuing, return the back-pressure signal at the boundary instead.

## Senior-level gotchas

- **Unbounded is the only wrong default.** `Channel.CreateUnbounded`, `ConcurrentQueue` without a size cap, `MemoryStream` over a request body — every one converts a rate mismatch into an OOM. If you find yourself reaching for an unbounded buffer "just for now," put a bound on it now and tune the size based on production traffic.
- **`SemaphoreSlim.WaitAsync()` without a timeout is back-pressure that cannot fail.** Threads accumulate at the gate without bound. Always pair with a `CancellationToken` carrying a deadline; otherwise you've moved the queue from your channel into the SemaphoreSlim's wait list — same OOM, different stack trace.
- **`BoundedChannelFullMode.DropOldest` is a measurement requirement, not a feature.** If you can't tell a stakeholder how many items per minute are dropped, don't ship Drop semantics. The right default for "I don't know" is `Wait`.
- **Back-pressure is end-to-end or it isn't.** A bounded channel between stages 2 and 3 doesn't help if stage 1 buffers in a `List<T>`. Audit every queue / buffer in the pipeline; the weakest link is the actual capacity.
- **`Task.Run` per work item is anti-back-pressure.** It hands the item to the thread pool's unbounded global queue. Use `Parallel.ForEachAsync` (.NET 6+) with `MaxDegreeOfParallelism`, or a bounded channel + N consumers.
- **Don't combine `lock` and `await`.** `Monitor` doesn't release across an await — the lock is held by the wrong thread on resume. Use `SemaphoreSlim(1, 1)` for async mutual exclusion. This is also what makes `Monitor` unusable as a back-pressure primitive.
- **Producer side timeout vs. consumer side timeout are different errors.** A producer that times out at the gate should return 503 to its caller. A consumer that times out processing one item should not drop the next 1,000 — the back-pressure mechanism upstream handles that. Don't conflate the two.
- **Bulkhead patterns serialize on the bulkhead's queue.** `Polly.Bulkhead` queues requests in arrival order; head-of-line blocking is a feature for fairness but a bug for latency-sensitive paths. For latency-sensitive code, use a smaller `maxQueuingActions` or no queue at all (reject immediately).
- **`SendAsync` on `BufferBlock<T>` returns `bool` — `false` means "not accepted."** A producer that ignores the return value silently drops items. The Dataflow analog of `TryWrite`'s anti-pattern.
- **Cross-process back-pressure requires the caller to honor the signal.** A retry policy that ignores `Retry-After` is just a faster DDoS. Polly's `WaitAndRetry` honors it; hand-rolled retries usually don't.
- **Latency under back-pressure is *intentional***. p99 climbing into the producer wait band is the system working as designed. Alarming on it is alarming on the safety mechanism. Alarm on the *gradient* (wait time growing) and on `dropped > 0`.
- **The HTTP request body is back-pressure-aware via `PipeReader`** — using `HttpContext.Request.BodyReader` instead of `Request.Body` propagates flow control to the client's TCP window. `Request.Body` re-buffers; the upload happens at line rate regardless of how fast you process it.
- **`ConfigureAwait(false)` doesn't relate to back-pressure**, but its absence in library code that runs in a SyncCtx (WPF / WinForms / older ASP.NET) bottlenecks all continuations on one thread — a back-pressure-shaped symptom from a completely different bug. Rule out the SyncCtx pin first.
