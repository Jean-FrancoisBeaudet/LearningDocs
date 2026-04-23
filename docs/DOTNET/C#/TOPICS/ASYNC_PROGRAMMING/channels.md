# Channels

_Targets .NET 10 / C# 14. See also: [Async streams and `await foreach`](./async-streams-and-await-foreach.md), [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [ValueTask](./valuetask.md), [CancellationTokenSource and TaskCompletionSource](./cancellationtokensource-and-taskcompletionsource.md)._

`System.Threading.Channels` is the modern producer/consumer primitive in .NET: a thread-safe, async-aware, optionally-bounded queue with configurable back-pressure. It replaces `BlockingCollection<T>` + `ThreadPool.QueueUserWorkItem` for nearly every in-process pipeline scenario, at a fraction of the allocation cost.

## The shape

```csharp
Channel<WorkItem> ch = Channel.CreateUnbounded<WorkItem>();
// or
Channel<WorkItem> ch = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 1024)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = true,
    SingleWriter = false,
});

ChannelWriter<WorkItem> writer = ch.Writer;
ChannelReader<WorkItem> reader = ch.Reader;
```

A channel is a split between a writer end and a reader end, like a `Pipe`. Both are thread-safe by default, but you can opt into single-reader / single-writer fast paths.

## Producer side

```csharp
// Async, back-pressured: awaits if the channel is full (BoundedChannelFullMode.Wait).
await writer.WriteAsync(item, ct);

// Fast path, no async: succeeds only if there's capacity.
if (!writer.TryWrite(item))
{
    // Full — decide: await, drop, or log.
    await writer.WriteAsync(item, ct);
}

// When the producer is done — any awaiting readers will see completion.
writer.Complete();        // or TryComplete()
writer.Complete(new IOException("upstream died"));  // fault the channel
```

`WriteAsync` returns `ValueTask` — usually completes synchronously when there's capacity, only allocates when it has to wait. That's why `Channel` APIs return `ValueTask` almost everywhere. See [ValueTask](./valuetask.md).

`Complete` is **terminal and idempotent-only via `TryComplete`**. After completion, no writer can enqueue; after the buffer drains, the reader sees the end.

## Consumer side

```csharp
// The idiomatic consumption pattern — await foreach over ReadAllAsync.
await foreach (var item in reader.ReadAllAsync(ct))
{
    await Handle(item, ct);
}
```

`ReadAllAsync` internally loops on `WaitToReadAsync` + `TryRead`. If you need the lower-level shape — e.g., batching N items at a time:

```csharp
while (await reader.WaitToReadAsync(ct))
{
    while (reader.TryRead(out var item))
        batch.Add(item);

    if (batch.Count > 0)
    {
        await Flush(batch, ct);
        batch.Clear();
    }
}
```

`WaitToReadAsync` returns `false` once the writer has completed **and** the buffer is empty. That's your exit condition — no sentinel values needed.

## Bounded channel options

```csharp
new BoundedChannelOptions(capacity)
{
    FullMode        = BoundedChannelFullMode.Wait,      // or DropOldest, DropNewest, DropWrite
    SingleReader    = true,
    SingleWriter    = false,
    AllowSynchronousContinuations = false,
}
```

### `FullMode` semantics

| Mode | Behavior when full |
|---|---|
| `Wait` | `WriteAsync` asynchronously blocks; `TryWrite` returns `false`. The producer feels the back-pressure. |
| `DropOldest` | The oldest buffered item is discarded; the new item enqueues. `TryWrite` **always returns `true`**. |
| `DropNewest` | The most-recent buffered item is discarded; the new item enqueues. |
| `DropWrite` | The new item is discarded. `WriteAsync` completes immediately with no effect. |

The three "Drop" modes exist for telemetry / UI-event pipelines where newer data is more valuable than older data, and the producer must not block. For anything that needs delivery guarantees, **use `Wait`**.

### `SingleReader` / `SingleWriter`

Opt-in fast paths. Saying `SingleReader = true` and then reading from two threads is a data race that the channel **does not check for you**. Only set these flags when you control the topology and want the measurable wins (fewer locks, fewer volatile reads).

### `AllowSynchronousContinuations`

Default `false` — continuations run on the thread pool. `true` gives you inline continuations and better throughput but re-introduces the TCS-style trap: the awaiting coroutine resumes on the writer's thread. Enable only in tight, known-safe consumers.

## Bounded vs unbounded

| When | Choose |
|---|---|
| Producer rate is inherently bounded (HTTP request handler, CLI command). | `CreateUnbounded` |
| Producer can outrun the consumer (Kafka reader → DB writer, sensor stream). | `CreateBounded` |
| Tests / one-shot fixtures. | `CreateUnbounded` |
| You want back-pressure to push through to an upstream source. | `CreateBounded` |

Unbounded channels can consume unbounded memory. That is the point of bounded. For any long-running in-process pipeline, start bounded and size the capacity based on measured latency × throughput.

## Canonical pipeline pattern

```csharp
public sealed class ParseAndSaveService(IRepo repo, ILogger<ParseAndSaveService> log)
    : BackgroundService
{
    private readonly Channel<string> _in = Channel.CreateBounded<string>(
        new BoundedChannelOptions(1024) { FullMode = BoundedChannelFullMode.Wait, SingleReader = true });

    public ChannelWriter<string> Writer => _in.Writer;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var raw in _in.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                var parsed = Parse(raw);
                await repo.SaveAsync(parsed, stoppingToken);
            }
            catch (Exception ex)
            {
                log.LogError(ex, "failed to process {Raw}", raw);
                // decide: continue, complete with fault, etc.
            }
        }
    }
}
```

Register the writer as a singleton, inject it into HTTP handlers, and you have a decoupled ingestion → processing pipeline with back-pressure, cancellation, and graceful shutdown for free.

## Channel vs ... everything else

| Alternative | When it's better | When Channel wins |
|---|---|---|
| `BlockingCollection<T>` | Legacy code already on it. | Every async scenario — `BlockingCollection` consumes a real thread blocked in `Take()`. |
| `IAsyncEnumerable<T>` | Single producer, single consumer, pull-based. | Multiple producers, or when producer and consumer lifecycles are decoupled. |
| `TPL Dataflow` (`BufferBlock<T>`, etc.) | Multi-stage graphs with per-stage parallelism and joining. | Simple N-to-M pipelines; Dataflow is heavier and noisier. |
| `ConcurrentQueue<T>` + `SemaphoreSlim` | Hand-rolled when you need very specific semantics. | Everyday pipelines — Channel is the pre-built, tested version. |
| `MediatR` / in-memory bus | Request/response, synchronous-feeling APIs. | Streaming work items, back-pressure, fire-and-forget. |

## Bridging to `IAsyncEnumerable<T>`

`reader.ReadAllAsync(ct)` *is* an `IAsyncEnumerable<T>`. That means you can compose it with LINQ (`System.Linq.Async`), pipe it through `foreach` loops, feed gRPC server streaming, or SignalR streaming hubs. For fan-out from one source to many consumers, bridge from an `IAsyncEnumerable<T>` into N channels:

```csharp
await foreach (var item in source.WithCancellation(ct))
    foreach (var w in writers)
        await w.WriteAsync(item, ct);
```

## Completion and faulting

```csharp
try
{
    await foreach (var raw in reader.ReadAllAsync(ct))
        await Handle(raw);
}
catch (ChannelClosedException ex)
{
    // The writer called Complete(exception). Re-throw or log.
    log.LogWarning(ex.InnerException, "upstream failed");
}
```

Completing with an exception is how producers signal fatal errors to consumers. Cancellation via `ct` is separate — throws `OperationCanceledException`, not `ChannelClosedException`.

**Senior-level gotchas:**
- `ChannelReader<T>` does not have `Count`. You can't "peek the queue depth" portably — by design, because the answer is stale as soon as you read it. Use your own counters if you need telemetry.
- Forgetting `writer.Complete()` leaves consumers hanging forever on `WaitToReadAsync`. Always pair completion with shutdown (either explicitly in a `finally` or via a `BackgroundService`'s `StopAsync`).
- `SingleReader`/`SingleWriter` flags are *declarative optimizations*, not enforcement. Violating them is a silent data race. The channel will not throw — you'll just get torn reads.
- `BoundedChannelFullMode.DropOldest` is racy with slow consumers that are also holding the item they just read — you can drop an item, log "dropped", and then have the slow consumer still processing the previously-read one. Back-pressure is what you usually actually want.
- `WriteAsync` can still block (asynchronously) even on an **unbounded** channel if the channel has been **completed** — it immediately throws `ChannelClosedException`. Don't assume "unbounded = never awaits."
- `await foreach` over `ReadAllAsync` captures the sync context at each `MoveNextAsync` unless you add `.ConfigureAwait(false)` on the enumerable. For library code, specify it: `.ReadAllAsync(ct).ConfigureAwait(false)`.
- Channels don't do priority, deduplication, or persistence. They are an in-memory FIFO. Reach for a real queue (Service Bus, RabbitMQ, Kafka) the moment any of those requirements appear.
- `Channel<T>` is allocation-light for `ValueTask`-returning APIs but each `WriteAsync` that **waits** still allocates a pooled state object. For zero-alloc steady state, ensure your sizing + consumer rate keep the channel below capacity.
