# Batch processing

_Targets .NET 10 / C# 14. See also: [Bulk operations](./bulk-operations.md), [Channel&lt;T&gt;](./channel-of-t.md), [Backpressure handling](./backpressure-handling.md), [Streaming patterns](./streaming-patterns.md), [Inter-service call optimization](./inter-service-call-optimization.md), [Aggregation efficiency](./aggregation-efficiency.md), [I/O saturation](./io-saturation.md)._

**Batch processing** is the discipline of *grouping items in-process before they leave it*, so that a fixed per-call cost (RPC round-trip, DB transaction, log flush, serialization preamble) is amortized across many items instead of paid once per item.

> Batching trades a fixed amount of latency (the linger window) for a multiplicative win on throughput and a divisional win on cost. The engineering work is sizing both knobs against the actual cost equation — and accepting that some items will sit in the buffer for a few milliseconds doing nothing.

Don't confuse it with [bulk operations](./bulk-operations.md) (the *downstream* API that accepts N rows in one call) or [streaming](./streaming-patterns.md) (which preserves per-item latency by definition). Batch processing is the producer-side queue + flush; bulk operations are what it ships to. They compose.

## The cost equation

For a per-call fixed cost `F` and per-item cost `c`:

```
sequential: N × (F + c)
batched:    ⌈N/B⌉ × F + N × c
```

Batching wins when `F` is non-trivial relative to `c`. Some rough numbers from real systems:

| Operation | F (per-call) | c (per-item) | Break-even batch size |
|---|---|---|---|
| SQL Server INSERT (in-VPC) | ~0.5 ms RTT + 1 ms server | tiny | 8–32 |
| Cross-region HTTP POST | ~30 ms RTT + TLS | tiny | 50–200 |
| Azure Service Bus send | ~10 ms + auth | tiny | 100–500 (subject to 256 KB cap) |
| App Insights / OTLP export | ~5 ms + serialization | bytes | 100–1000 |
| File flush to disk | sync(): 1–5 ms | append cost | 50–500 |

If `F` is comparable to `c` — e.g. an in-process function call, a memory write, a hot-cache lookup — **don't batch**. You're adding linger latency for no throughput gain. Likewise, don't batch CPU-bound work that's already running on a hot loop; batching there just turns a steady stream into bursts.

## The two-trigger flush rule

A correct batcher flushes when **either** of two conditions is met, whichever comes first:

1. **Size**: the buffer has reached `maxBatchSize`.
2. **Age**: the *oldest* item has waited `maxLinger`.

Linger-only flush starves a low-traffic system (one item never leaves). Size-only flush adds unbounded latency to the last item before a quiet period. Both triggers, always.

The age trigger watches the **oldest** item, not "time since last flush" — those two are different at low traffic, and the latter lets the first item of a batch sit there forever as long as new items keep arriving.

## Worked example: a `BatchingChannel<T>`

```csharp
public sealed class BatchingChannel<T> : IAsyncDisposable
{
    private readonly Channel<T> _channel;
    private readonly int _maxBatchSize;
    private readonly TimeSpan _maxLinger;
    private readonly Func<IReadOnlyList<T>, CancellationToken, Task> _sink;
    private readonly CancellationTokenSource _stop = new();
    private readonly Task _drainLoop;

    public BatchingChannel(
        int maxBatchSize,
        TimeSpan maxLinger,
        Func<IReadOnlyList<T>, CancellationToken, Task> sink)
    {
        _maxBatchSize = maxBatchSize;
        _maxLinger = maxLinger;
        _sink = sink;
        _channel = Channel.CreateBounded<T>(new BoundedChannelOptions(maxBatchSize * 8)
        {
            FullMode = BoundedChannelFullMode.Wait,   // back-pressure to the producer
            SingleReader = true,
        });
        _drainLoop = Task.Run(DrainAsync);
    }

    public ValueTask EnqueueAsync(T item, CancellationToken ct) =>
        _channel.Writer.WriteAsync(item, ct);

    private async Task DrainAsync()
    {
        var buffer = new List<T>(_maxBatchSize);
        var reader = _channel.Reader;

        while (await reader.WaitToReadAsync(_stop.Token).ConfigureAwait(false))
        {
            // First item: starts the linger clock.
            if (!reader.TryRead(out var first)) continue;
            buffer.Add(first);
            using var lingerCts = CancellationTokenSource.CreateLinkedTokenSource(_stop.Token);
            lingerCts.CancelAfter(_maxLinger);

            // Drain until full OR linger expires.
            while (buffer.Count < _maxBatchSize)
            {
                try
                {
                    if (!await reader.WaitToReadAsync(lingerCts.Token).ConfigureAwait(false))
                        break;  // channel completed
                    while (buffer.Count < _maxBatchSize && reader.TryRead(out var next))
                        buffer.Add(next);
                }
                catch (OperationCanceledException) when (lingerCts.IsCancellationRequested
                                                       && !_stop.IsCancellationRequested)
                {
                    break;  // linger expired — flush what we have
                }
            }

            try { await _sink(buffer, _stop.Token).ConfigureAwait(false); }
            catch (Exception ex) { /* log, DLQ, decide retry policy */ }
            finally { buffer.Clear(); }
        }
    }

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.TryComplete();
        await _drainLoop.ConfigureAwait(false);
        _stop.Dispose();
    }
}
```

Wiring it up to ship to a bulk endpoint:

```csharp
var batcher = new BatchingChannel<TelemetryEvent>(
    maxBatchSize: 200,
    maxLinger: TimeSpan.FromMilliseconds(50),
    sink: (batch, ct) => _otlpClient.ExportAsync(batch, ct));

await batcher.EnqueueAsync(@event, ct);
```

A 200-item / 50 ms window is a reasonable starting point for telemetry exporters: the linger adds ≤50 ms to the very last event before a quiet period, and burst traffic flushes on size before the timer fires.

## Failure model: what to do with a half-full batch that fails

| Strategy | When it's right | Trap |
|---|---|---|
| Retry the whole batch | Idempotent sinks (dedup key per item) | One bad item poisons the batch forever |
| Split on failure (binary search) | Mostly-good batches with a rare poison item | Adds latency on failures; only use if poison-pill is rare |
| Drop the batch + DLQ each item | Telemetry, logs, anything where loss < unbounded retry | Silent loss if DLQ inspection isn't wired |
| Per-item retry within the batch handler | Small batches, cheap retries | Defeats the bulk API; consider why you're batching |

The default for new code: **retry the whole batch with jittered exponential backoff, max N attempts, then DLQ each item**. Any other policy needs a written justification.

Idempotency: assign an idempotency key **per item**, not per batch. If the batch retries with one new item appended, the sink must dedupe — server-side, not client-side. Otherwise retries double-write the items that succeeded the first time.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| Linger-only flush (no size cap) | Burst traffic = giant batches = OOM on the sink |
| Size-only flush (no linger) | Low-traffic items wait forever; p99 = ∞ |
| Time-since-last-flush instead of age-of-oldest | Slow leak: batch can be hours old under steady traffic |
| Unbounded buffer (`Channel.CreateUnbounded`) | Producer outruns sink → working set climbs to OOM |
| `Task.Run(() => sink(item))` per arrival | Defeats batching entirely; thread-pool scheduling cost > the bulk savings |
| Per-item retry inside the batch sink | Hides partial failures; observability becomes guesswork |
| Same `List<T>` reused across batches without `Clear()` | Items from batch N leak into batch N+1 |
| `lock` around `await sink(...)` inside the drainer | `Monitor` doesn't release across an await — a quietly-broken batcher |
| Batching CPU-bound items already on a hot loop | All linger, no win |
| Batch size set in config without knowing the sink's max | One day you exceed 256 KB Service Bus limit and silently drop |

## Observability

Three signals are mandatory:

```csharp
private static readonly Meter s_meter = new("MyApp.Batching");
private static readonly Histogram<int> s_size = s_meter.CreateHistogram<int>("batch.size");
private static readonly Histogram<double> s_linger = s_meter.CreateHistogram<double>("batch.linger_ms");
private static readonly Histogram<double> s_flush = s_meter.CreateHistogram<double>("batch.flush_ms");
private static readonly Counter<long> s_failed = s_meter.CreateCounter<long>("batch.failed");
```

| Metric | What it tells you |
|---|---|
| `batch.size` p50 / p99 | If p50 ≪ maxBatchSize, you're linger-bound (timer fires first). If p99 = max, you're size-bound (good — size is the cap). |
| `batch.linger_ms` | Time from first-item arrival to flush. Should track `maxLinger` at low traffic; ≪ `maxLinger` at high traffic. |
| `batch.flush_ms` | Sink cost. If > linger, raise linger or you'll never catch up. |
| `batch.failed` | DLQ rate. Non-zero for sustained periods = capacity loss. |

Tune from data: if `batch.size` p50 = 5 and `maxBatchSize` = 200, the cost equation says reduce `maxLinger` (you're paying for nothing). If p99 = 200 and `flush_ms` is climbing, raise `maxBatchSize` or scale concurrency on the sink.

## Senior-level gotchas

- **Linger is added p99 by design.** A user-visible ingest endpoint that batches at 50 ms has *added* 50 ms to its tail latency. Don't batch synchronously-awaited work unless the user can absorb the linger; batch the *background* publish, not the request path.
- **`PeriodicTimer` inside the batcher is wrong.** A timer that fires every 50 ms is unrelated to the age of the *oldest item*. Use a per-batch `CancellationTokenSource.CancelAfter` keyed off the first item's arrival, as in the worked example.
- **`maxBatchSize × concurrency = downstream load`.** If you have 4 batchers in the same process each at 200 × 50 ms, the sink sees up to 16k items every 50 ms. Coordinate with the sink's bulk endpoint limits and connection pool.
- **Idempotency keys are per item, not per batch.** A batch-level key is meaningless under split-and-retry: the second attempt has a different shape. Server-side dedup must be per item.
- **Ordering across batches is not guaranteed.** Two batchers in parallel ship in arrival order *within* a batch but interleave *across* batches. If the consumer cares about ordering, partition by key into single-batcher streams (one per key shard).
- **Don't batch the request path.** A POST that returns `202 Accepted` after enqueue is fine. A POST that waits for the batch flush to return `200` has built a request-coupled queue with worse semantics than a bare RPC.
- **`async void` in the drain loop swallows failures.** The drain loop must be `Task.Run(async () => ...)` and observed via `await _drainLoop` on shutdown. An unobserved drain loop that crashes silently leaves the channel filling forever.
- **Bounded channel + `Wait` mode = back-pressure.** This is the right default and the reason the buffer in the example is `maxBatchSize × 8`, not unbounded. If the producer is faster than the sink, `EnqueueAsync` awaits — see [backpressure-handling.md](./backpressure-handling.md).
- **`reader.TryRead` after `WaitToReadAsync` can still fail** if a competing reader exists. Set `SingleReader = true` so the channel can use the fast path and the contract is enforced.
- **Shutdown is not graceful by default.** `_channel.Writer.TryComplete()` lets the drain loop finish the current pull; cancelling `_stop` aborts mid-batch and loses items unless the sink supports DLQ-on-cancel. Pick: drain-on-shutdown (slower stop), or DLQ-on-shutdown (loss-free but visible delay).
- **Batching telemetry is a feedback loop.** If your batcher emits a metric per batch, and the metric goes through a batcher, and that batcher emits a metric... measure the *outer* counter via `Counter<long>` (cheap, lock-free) and the *inner* via histograms only when sampling. Otherwise the observer dominates the workload.
- **`Channel<T>` is not free.** A bounded channel has heap allocation per write (for the linked-list node) and a synchronization cost on the WaitToRead/Read split. For ultra-high-throughput in-process batching (>1M items/sec), reach for `System.Threading.Channels` benchmarks or a ring-buffer (`Disruptor.NET`).
- **The first batch after startup is always small.** Tune dashboards to ignore the first 30 seconds, or you'll alert on warm-up. Same for the last batch before a graceful shutdown.
- **Don't store mutable state in the batch items.** If item N's processing inside the sink mutates item N+1 (shared reference, common in DTO reuse), you'll see Heisenbugs that depend on batch composition. Make items immutable records.
