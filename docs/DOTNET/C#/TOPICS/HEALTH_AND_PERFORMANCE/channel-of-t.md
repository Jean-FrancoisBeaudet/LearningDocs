# Channel&lt;T&gt;

_Targets .NET 10 / C# 14. See also: [Channels (API surface)](../ASYNC_PROGRAMMING/channels.md), [ValueTask](../ASYNC_PROGRAMMING/valuetask.md), [System.IO.Pipelines](./system-io-pipelines.md), [Backpressure handling](./backpressure-handling.md), [Streaming patterns](./streaming-patterns.md), [Synchronization bottlenecks](./synchronization-bottlenecks.md), [async/await performance](./async-await-performance.md)._

`System.Threading.Channels` is the modern in-process producer/consumer primitive. The API surface and idiomatic usage live in [`channels.md`](../ASYNC_PROGRAMMING/channels.md); this note is the **performance and health** view — allocation profile, fast paths, sizing, observability, and the failure modes a senior .NET engineer needs to recognize on sight.

## Allocation profile

```csharp
ValueTask WriteAsync(T item, CancellationToken ct);   // ValueTask, not Task
bool TryWrite(T item);                                // sync fast path
ValueTask<bool> WaitToReadAsync(CancellationToken ct);
bool TryRead(out T item);
```

The whole API is `ValueTask`-based for a reason: in steady state, `WriteAsync` to a non-full bounded channel and `TryRead` to a non-empty buffer **complete synchronously**, returning a `ValueTask` wrapping a literal — no heap allocation.

When the channel is full and `WriteAsync` has to wait, the wait state is satisfied from a small internal pool of `IValueTaskSource<bool>` instances. So the truly hot path looks like:

| Path | Per-call allocation |
|---|---|
| `TryWrite` to non-full | **0** |
| `WriteAsync` to non-full | **0** (sync-completed `ValueTask`) |
| `WriteAsync` to full, awaits | pooled state object — ~zero amortized |
| `TryRead` from non-empty | **0** |
| `await foreach (... ReadAllAsync(ct))` | one enumerator per call (then 0 per item) |

The only way to leak allocations is to violate the `ValueTask` single-await rule (see [`valuetask.md`](../ASYNC_PROGRAMMING/valuetask.md)) — `Task.WhenAll(channel.Writer.WriteAsync(...).AsTask())` allocates per call. Don't do it on hot paths; use `TryWrite` + a fallback awaiting branch.

## `SingleReader` / `SingleWriter` fast paths

```csharp
Channel.CreateBounded<Work>(new BoundedChannelOptions(1024) {
    SingleReader = true,
    SingleWriter = false,
});
```

These flags switch the channel to specialized internal implementations:

- `SingleReader = true` removes the lock around `TryRead` / `WaitToReadAsync` — the read state is owned by one consumer, no CAS contention.
- `SingleWriter = true` does the same on the producer side.

Both flags are **declarative optimizations**, not runtime checks. Violating them is a silent data race — torn reads, lost items, no exception. Set them only when you control the topology (one `BackgroundService` consumer; one HTTP middleware producer).

Measured wins: 20-40% throughput improvement on tight producer/consumer loops; larger when the alternative path involves repeated CAS contention. Confirm with BenchmarkDotNet before committing to the constraint.

## `AllowSynchronousContinuations`

```csharp
new BoundedChannelOptions(capacity) {
    AllowSynchronousContinuations = true,    // default false
}
```

Defaults to `false`: continuations queue to the thread pool (one queue hop, predictable latency). With `true`, the awaiting consumer resumes **on the writer's thread**, inline with `WriteAsync`. That's a throughput win — no queue hop, no context switch — but the writer is now blocked on consumer code.

When `true` is right:

- Both sides of the channel are part of the same internal pipeline you control.
- The consumer is fast and predictable (no I/O, no `await`).
- You've measured the queue-hop cost.

When `true` is wrong:

- The consumer awaits an external resource. The writer thread now blocks on a network round trip.
- Multiple producers contend for the same channel. One slow consumer transitively serializes all producers.

Default to `false`. Flip after profiling proves the queue hop is the bottleneck.

## Capacity sizing

A bounded channel's capacity is `peak burst × consumer latency budget`, in items:

```
capacity = (peak items per second) × (consumer p99 latency in seconds)
```

If the consumer takes 50 ms p99 to handle one item and the producer can burst 2,000/sec, `capacity = 2_000 × 0.050 = 100`. Round up for safety. Sized too small, the producer blocks frequently and the upstream system feels back-pressure (see [`backpressure-handling.md`](./backpressure-handling.md)). Sized too large, you're using the channel as a bug-amplifier — the producer keeps running ahead of the failing consumer until the buffer eats a meaningful chunk of working set.

For variable-rate workloads, prefer **moderate capacity + measurement** over guessing high. Capacity `1024-8192` covers most scenarios; over `100_000` you should be reaching for an external queue (Service Bus, RabbitMQ, Kafka) instead.

## `BoundedChannelFullMode` choice

| Mode | When | Trade-off |
|---|---|---|
| `Wait` | Delivery guarantees matter (work queues, ingestion) | Producer feels back-pressure; upstream slowdown propagates |
| `DropOldest` | Newer data is more valuable (telemetry, UI ticks, sensor streams) | Silent loss; old items vanish |
| `DropNewest` | Recent items spam-suppressed in favor of stable data | Silent loss; new items vanish |
| `DropWrite` | Producer cannot block, consumer's behind = drop on floor | Silent loss; `WriteAsync` returns immediately with no effect |

The three Drop modes are silent — `TryWrite` returns `true` either way. **Always meter the drop count**; otherwise you've shipped silent data loss. The standard shape:

```csharp
private static long _dropped;

if (writer.TryWrite(item))
    Interlocked.Increment(ref _enqueued);
else
    Interlocked.Increment(ref _dropped);   // only meaningful if FullMode == DropWrite
```

For `DropOldest`/`DropNewest` you have to instrument the channel internals or wrap the writer in a counting decorator — the channel itself does not surface what it dropped.

## Health metrics

`ChannelReader<T>` deliberately does **not** expose `Count`. The depth is stale the moment you read it, and the omission forces you to instrument intent rather than guess from snapshots. The standard instrumentation:

```csharp
private static readonly Meter s_meter = new("MyApp.Channel");
private static readonly Counter<long> s_enqueued = s_meter.CreateCounter<long>("channel.enqueued");
private static readonly Counter<long> s_dequeued = s_meter.CreateCounter<long>("channel.dequeued");
private static readonly Histogram<double> s_waitMs = s_meter.CreateHistogram<double>("channel.write_wait_ms");

public async ValueTask EnqueueAsync(T item, CancellationToken ct) {
    if (!_writer.TryWrite(item)) {
        var sw = Stopwatch.StartNew();
        await _writer.WriteAsync(item, ct);
        s_waitMs.Record(sw.Elapsed.TotalMilliseconds);
    }
    s_enqueued.Add(1);
}
```

`enqueued - dequeued` gives a moving estimate of depth without polling. `write_wait_ms` rising is the early signal that the consumer is falling behind — well before the producer starts blocking outright. Wire both to your OpenTelemetry exporter; alert on `write_wait_ms p99 > consumer_budget_ms`.

## Channel vs alternatives

| Alternative | When it's better | When `Channel<T>` wins |
|---|---|---|
| `BlockingCollection<T>` | Legacy code already on it | Every async scenario — `BlockingCollection.Take()` parks a real thread |
| `IAsyncEnumerable<T>` | Single producer / single consumer, pull-based, lifecycles aligned | Producer and consumer have decoupled lifecycles, or N-to-M topologies |
| `TPL Dataflow` (`BufferBlock<T>`, `ActionBlock<T>`) | Multi-stage graphs with per-stage parallelism, joining, broadcasting | Simple N-to-M pipelines — Dataflow is heavier and noisier for the common case |
| `ConcurrentQueue<T>` + `SemaphoreSlim` | Hand-rolled when you need very specific semantics | Everyday pipelines — `Channel<T>` is the pre-built, tested version |
| `System.IO.Pipelines` | Bytes (parsing protocols) | Already-parsed objects |
| In-process bus (MediatR, MassTransit-in-memory) | Request/response, synchronous-feeling APIs | Streaming, fire-and-forget, back-pressured fan-in |

## Common anti-patterns

| Anti-pattern | What breaks |
|---|---|
| Forgetting `writer.Complete()` | Consumers block on `WaitToReadAsync` forever; `BackgroundService.StopAsync` hangs |
| `CreateUnbounded` with no rate limit upstream | Producer outruns consumer, working set grows unbounded, OOM |
| Sharing a channel across multiple intended-single readers | Silent data race — items disappear, no exception |
| Using `DropOldest` without metering drops | Silent data loss in production, indistinguishable from a working system |
| Awaiting `WriteAsync(...).AsTask()` in `Task.WhenAll` | Loses the `ValueTask` sync-completion path, allocates per call |
| Holding the channel writer past producer shutdown | Consumer can never exit; `await foreach` hangs |
| `lock`-ing around `TryWrite` / `TryRead` | Defeats the channel's own thread-safety; adds contention you'll diagnose as "channel is slow" |
| Treating channel as a persistent queue | It's an in-memory FIFO — process restart drops the buffer; use Service Bus / RabbitMQ for durability |

## Graceful shutdown

```csharp
// Producer side: complete after the last write.
public async Task IngestAsync(CancellationToken ct) {
    try {
        await foreach (var raw in _source.ReadAllAsync(ct))
            await _writer.WriteAsync(raw, ct);
    }
    finally {
        _writer.Complete();    // signals consumer to drain and exit
    }
}

// Consumer side: drain naturally on completion.
protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
    await foreach (var item in _reader.ReadAllAsync(stoppingToken))
        await Handle(item, stoppingToken);
    // returns once the buffer drains AND the writer has completed
}
```

`stoppingToken` (from `BackgroundService.ExecuteAsync`) handles forced shutdown — the `await foreach` throws `OperationCanceledException` and the buffer is **discarded**. For drain-on-shutdown semantics, complete the writer in `StopAsync` and don't pass the token to the consumer's `ReadAllAsync`. That's the standard "process the last items, then exit" pattern.

## Senior-level gotchas

- **`ChannelReader<T>` has no `Count`** by design. The number is meaningless concurrently. If you need depth, instrument enqueue/dequeue counters yourself — and accept that the value is an estimate.
- **`SingleReader`/`SingleWriter` are silent contracts.** Violating them is a data race, not an exception. The channel will quietly tear reads or lose writes; you'll diagnose it as "occasional missing items" months later. Only set them when you genuinely own the topology.
- **`AllowSynchronousContinuations = true` lets a slow consumer block the producer's thread.** The throughput win evaporates the moment the consumer awaits anything substantial. Default `false` is correct for any consumer that does I/O.
- **`WriteAsync` on an unbounded channel can still throw `ChannelClosedException`** — `Complete` is terminal. "Unbounded" guarantees no awaiting, not "always succeeds."
- **`DropOldest` is racy with a slow consumer that's already holding an item it just read.** You can drop an item, log "dropped: 1," and then the consumer is *still* processing the previously-read one — net effect is one item processed, one dropped, in an order the metrics suggest is reversed. Back-pressure (`Wait`) is what most callers actually want.
- **Forgetting `Complete()` is the #1 cause of `BackgroundService` not shutting down.** The consumer's `await foreach` waits indefinitely on `WaitToReadAsync`. Pair every long-lived producer with a `try`/`finally` that completes the writer.
- **`Channel<T>` is in-memory FIFO** — no priority, no deduplication, no persistence. The moment any of those requirements appear, you need a real broker. Treating channels as durable is the start of an outage.
- **`await foreach (var x in reader.ReadAllAsync(ct))` in library code should append `.ConfigureAwait(false)`** on the enumerable. The continuation captures the SyncCtx otherwise — a silent perf bug for code that may run anywhere.
- **The pool of `IValueTaskSource<T>` instances is bounded.** Under extreme contention (millions of awaiting writers), the pool can saturate and fall back to per-call allocation. If your "alloc-free channel" is allocating in production, the cause is usually pool saturation — reduce concurrency or batch writes.
- **`Channel.CreateBounded<T>(capacity: 0)`** is a special "rendezvous" channel — every write must hand off directly to a waiting reader. Useful for synchronous-feeling pipelines, but each `WriteAsync` blocks until a reader appears. Don't pick it accidentally.
- **`ChannelClosedException` from `WriteAsync` after `Complete` is the writer's fault, not the consumer's.** Producers must check `_writer.TryComplete()` (idempotent) when they own the lifecycle, not call `Complete()` from multiple sites.
- **Channels don't observe ambient `ActivitySource` contexts.** If you rely on OpenTelemetry distributed traces across producer → consumer, capture `Activity.Current` into the queued item and restore it on the consumer side. The channel is a black box to the trace.
