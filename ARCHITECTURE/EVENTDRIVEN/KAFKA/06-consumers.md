# Kafka Consumers — Deep Dive

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- A Kafka consumer is a **single-threaded poll loop** by design. `Consume()` drives heartbeats, fetches, offset commits, and rebalance callbacks. If your loop blocks, the group thinks you're dead.
- Offsets are stored in the internal compacted topic **`__consumer_offsets`** (replication factor 3, 50 partitions by default). There is no per-record ack — offsets are a watermark per `(group, topic, partition)`.
- **Auto-commit (`enable.auto.commit=true`) is at-most-once or at-least-once depending on failure timing** — never exactly-once. Production systems should commit manually, after processing, synchronously on rebalance/shutdown and asynchronously otherwise.
- The **three timeouts you must understand**: `session.timeout.ms` (default **45 s**) — liveness window at the group coordinator; `heartbeat.interval.ms` (default **3 s**) — background heartbeat thread cadence; `max.poll.interval.ms` (default **5 min**) — max time between `Consume()` calls before the group considers you stuck and kicks you out.
- Fetch sizing (`fetch.min.bytes`, `fetch.max.wait.ms`, `max.partition.fetch.bytes`) controls throughput vs latency. `max.poll.records` (Java) / `MaxPollIntervalMs` (librdkafka) bounds loop work per iteration.
- Poison-pill handling is **your** job. The consumer will infinitely re-deliver a record it can't deserialize if you don't handle it explicitly. Pattern: catch `ConsumeException`, log + DLQ, advance offset.

## Deep dive

### The poll loop contract
The consumer is fundamentally a single thread that does the following on each `Consume(timeout)`:

1. Sends heartbeat if due (in librdkafka, a background thread also does this, but the Java client ties it to poll by default).
2. Fetches records from assigned partitions (pre-fetched in background).
3. Returns a `ConsumeResult<K,V>` (one record in .NET / librdkafka; a batch up to `max.poll.records` in Java).
4. Fires rebalance callbacks (partitions-assigned / partitions-revoked / partitions-lost) synchronously on the calling thread.
5. Commits offsets if `enable.auto.commit=true` and the interval elapsed.

**If your processing blocks longer than `max.poll.interval.ms`, you'll be kicked out of the group** and the next `Consume()` will trigger a rebalance. Long blocking work (DB writes, HTTP calls, image processing) must either finish under the deadline, or use the pause/resume + worker-thread pattern (below).

### Offset storage: `__consumer_offsets`
- Internal compacted topic, 50 partitions, RF=3 (configurable via `offsets.topic.replication.factor` / `offsets.topic.num.partitions`).
- Key: `(group, topic, partition)`. Value: `(offset, leader_epoch, metadata, commit_timestamp)`.
- Log compaction keeps only the latest offset per key. On broker restart / group coordinator failover, offsets are rebuilt by replaying the relevant partition of `__consumer_offsets`.
- `offsets.retention.minutes` (default **10080** = 7 days) controls how long offsets for an **inactive** group are retained. Go quiet for 8 days, come back, and your group may reset to `auto.offset.reset`.

### Commit strategies

| Strategy | Semantics | Throughput | Complexity |
|---|---|---|---|
| Auto-commit every `auto.commit.interval.ms` | Best-effort at-least-once (can lose or duplicate on crash) | Highest | Zero |
| Manual sync commit after every record | At-least-once | Low (round-trip per record) | Low |
| Manual sync commit every N records or T seconds | At-least-once (small window) | High | Low |
| Manual async commit + sync on rebalance/shutdown | At-least-once | Highest manual | Medium (this is the standard prod pattern) |
| External transactional commit (commit offsets to a DB alongside side effects) | Exactly-once (application-level) | Medium | High |
| Kafka transactions (read-process-write) | Exactly-once (Kafka-only) | Medium | High — see `08-delivery-semantics.md` |

**The canonical production pattern**: `enable.auto.commit=false`. Inside the poll loop, do async commits every few seconds or N records. Inside `PartitionsRevoked` handler **do a synchronous commit** so the next consumer picks up without reprocessing. On `Dispose`/shutdown, also commit sync.

Why not always sync? A sync commit is a round-trip (5–50 ms). At 100k msg/s, committing per record kills throughput. Async commits pipeline.

Why sync on rebalance? The rebalance callback is the last time *this instance* owns the partition. If you exit without committing, another instance picks up the old offset and reprocesses.

### Fetch sizing
The consumer pre-fetches in the background. Knobs:

| Config | Default | Effect |
|---|---|---|
| `fetch.min.bytes` | `1` | Broker waits until it has this many bytes (or `fetch.max.wait.ms` elapsed) before responding. Bump to 50 KB–1 MB to improve throughput / reduce broker CPU |
| `fetch.max.wait.ms` | `500` | Max time broker holds a fetch waiting for `fetch.min.bytes` |
| `fetch.max.bytes` | `52428800` (50 MiB) | Max bytes per fetch response across all partitions |
| `max.partition.fetch.bytes` | `1048576` (1 MiB) | Max bytes per partition per fetch |
| `max.poll.records` (Java) | `500` | Max records returned by a single `poll()` |
| `queued.max.messages.kbytes` (librdkafka) | `65536` | Local pre-fetch queue size per partition |

Latency-sensitive workloads: keep `fetch.min.bytes=1` and `fetch.max.wait.ms=100`.
Throughput workloads: `fetch.min.bytes=100000`, `fetch.max.wait.ms=500`.

`max.partition.fetch.bytes` must be ≥ the broker's `message.max.bytes` — otherwise a single oversized record blocks the partition forever (you get `MESSAGE_TOO_LARGE` repeatedly).

### `max.poll.interval.ms`, `session.timeout.ms`, `heartbeat.interval.ms`

These three are the most-misconfigured consumer settings.

- **`heartbeat.interval.ms`** (3 s): a background heartbeat thread pings the group coordinator. Decoupled from poll since KIP-62.
- **`session.timeout.ms`** (45 s in 3.x+, up from 10 s in older versions — KIP-735): if no heartbeat arrives for this long, the coordinator evicts the member. Rule of thumb: `session.timeout.ms >= 3 × heartbeat.interval.ms`.
- **`max.poll.interval.ms`** (5 min): if `Consume()` isn't called again for this long, the coordinator considers the member stuck (even if heartbeats are arriving) and evicts. This protects against deadlocked/hung processing threads.

Tuning:

- Workloads with heavy per-batch processing: raise `max.poll.interval.ms` to cover worst-case processing time, or use pause/resume + worker threads (see below).
- Kubernetes pods: `session.timeout.ms` should be > pod startup grace period for the graceful shutdown signal path.
- Cloud instances behind flaky NAT: `session.timeout.ms` shouldn't be so low that network blips cause rebalance storms.

Broker-side bound: `group.min.session.timeout.ms` (6 s) and `group.max.session.timeout.ms` (30 min). Your client setting is clamped to this range.

### Poison-pill handling
A **poison pill** is a record the consumer cannot deserialize (bad schema, corrupt payload, wrong type). Without handling, the consumer retries forever — the offset never advances — and the partition is stuck.

The .NET / librdkafka client surfaces this as a `ConsumeException` with a `ConsumerRecord` carrying the raw bytes. Pattern:

1. Catch `ConsumeException`.
2. Log `ex.ConsumerRecord.TopicPartitionOffset` and raw payload.
3. Produce the raw payload to a **dead-letter topic** (same key to preserve ordering for that key's downstream consumer).
4. **Manually commit the offset past the bad record** — this is the crucial step.
5. Continue.

Alternative: use a lenient deserializer that returns `Maybe<T>` / `Result<T>` and handle bad records as normal data-flow events rather than exceptions. This avoids the "throw + catch + commit" dance.

### Pause / resume and seek
`IConsumer<K,V>.Pause(IEnumerable<TopicPartition>)` stops pre-fetching from those partitions — the consumer still heartbeats, still owns the partitions, but `Consume()` won't return records from them. `Resume(...)` undoes it.

Use cases:
- **Slow downstream**: downstream (DB, HTTP) is struggling → pause → drain in-flight work → resume.
- **Async worker threads**: dispatch records to a `Channel<T>`, pause partition while backlog > N, resume when drained. Required pattern to do useful parallelism beyond "one thread per partition" and avoid `max.poll.interval.ms` eviction.

`Seek(TopicPartitionOffset)` moves the in-memory fetch position. It does **not** commit. Useful for:
- Reprocessing after a bug: `SeekToBeginning()` / `Seek(tp, timestamp)`.
- Consuming from a specific offset after external state restore.

Timestamp-based seek: `OffsetsForTimes(TopicPartitionTimestamp)` resolves a wall-clock timestamp to the first offset at or after that time. Hugely useful for debugging and replay.

### `auto.offset.reset`
Only applied when there is **no committed offset** for a group-partition:
- `earliest`: start from oldest retained record. Replay-friendly. Dangerous on huge topics.
- `latest` (default): start from tail. Miss records produced between group creation and first poll.
- `none`: throw `NoOffsetForPartitionException`. Safest for production — forces explicit decision.

Common bug: developer assumes `auto.offset.reset=earliest` means "always replay". It doesn't — once an offset is committed, it's respected; reset only fires for *new* groups or after retention expiry.

### `isolation.level`
- `read_uncommitted` (default): returns all records, including those written by open/aborted transactions.
- `read_committed`: returns only records from committed transactions; holds back records from in-flight transactions until commit/abort. Introduces **Last Stable Offset (LSO)** — reads stall at the LSO when a transaction is open.

Use `read_committed` in any EOS pipeline. See `08-delivery-semantics.md`.

## Config reference

| Key | Default | Notes |
|---|---|---|
| `bootstrap.servers` | (required) | CSV |
| `group.id` | (required for groups) | Logical consumer group identity |
| `group.instance.id` | `null` | Static membership (KIP-345); see `07-consumer-groups-and-rebalancing.md` |
| `enable.auto.commit` | `true` | Turn OFF in production |
| `auto.commit.interval.ms` | `5000` | Only if auto-commit on |
| `auto.offset.reset` | `latest` | `earliest` / `latest` / `none` |
| `session.timeout.ms` | `45000` | Liveness window (KIP-735 raised from 10s) |
| `heartbeat.interval.ms` | `3000` | Background heartbeat cadence |
| `max.poll.interval.ms` | `300000` (5 min) | Max time between polls before eviction |
| `max.poll.records` (Java) | `500` | Batch size per poll |
| `fetch.min.bytes` | `1` | Broker-side hold threshold |
| `fetch.max.wait.ms` | `500` | Max broker wait |
| `fetch.max.bytes` | `52428800` (50 MiB) | Per-response cap |
| `max.partition.fetch.bytes` | `1048576` (1 MiB) | Per-partition per-response cap |
| `isolation.level` | `read_uncommitted` | Use `read_committed` for EOS consumers |
| `partition.assignment.strategy` | `[RangeAssignor, CooperativeStickyAssignor]` | See `07-consumer-groups-and-rebalancing.md` |
| `enable.partition.eof` | `false` (librdkafka) | Emit sentinel when reaching partition end |
| `client.id` | `""` | Set it — shows up in quotas/metrics |
| `check.crcs` | `true` | Leave on; negligible cost |
| `allow.auto.create.topics` | `false` (since 2.4) | Should stay off in production |

## .NET / C# snippet (Confluent.Kafka)

```csharp
using Confluent.Kafka;
using System.Text.Json;

var config = new ConsumerConfig
{
    BootstrapServers        = "broker1:9092,broker2:9092,broker3:9092",
    GroupId                 = "order-projector",
    ClientId                = $"order-projector-{Environment.MachineName}",
    GroupInstanceId         = Environment.GetEnvironmentVariable("POD_NAME"), // static membership

    // Offsets: never auto-commit in prod.
    EnableAutoCommit        = false,
    AutoOffsetReset         = AutoOffsetReset.Earliest,

    // Liveness/stuck-detection.
    SessionTimeoutMs        = 45_000,
    HeartbeatIntervalMs     = 3_000,
    MaxPollIntervalMs       = 300_000,

    // Fetch tuning: moderate throughput with low latency.
    FetchMinBytes           = 1,
    FetchWaitMaxMs          = 100,
    MaxPartitionFetchBytes  = 1_048_576,

    // Cooperative sticky to minimize stop-the-world rebalances.
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.CooperativeSticky,

    // EOS downstream reads.
    IsolationLevel          = IsolationLevel.ReadCommitted,

    // Safety: don't let consumers silently auto-create topics.
    AllowAutoCreateTopics   = false,
};

using var consumer = new ConsumerBuilder<string, byte[]>(config)
    .SetKeyDeserializer(Deserializers.Utf8)
    .SetValueDeserializer(Deserializers.ByteArray)
    .SetErrorHandler((_, e) =>
        Console.Error.WriteLine($"[consumer] {e.Code}: {e.Reason} (fatal={e.IsFatal})"))
    .SetPartitionsAssignedHandler((c, partitions) =>
    {
        Console.WriteLine($"assigned: [{string.Join(", ", partitions)}]");
        // return the partitions you accept (or offsets to seek to)
    })
    .SetPartitionsRevokedHandler((c, partitions) =>
    {
        // CRITICAL: sync commit here before giving up ownership.
        try { c.Commit(); } catch (KafkaException ex) { Console.Error.WriteLine(ex); }
    })
    .SetPartitionsLostHandler((c, partitions) =>
    {
        // "Lost" means we were kicked out (session timeout). Don't try to commit — we no longer own these.
        Console.Error.WriteLine($"lost: [{string.Join(", ", partitions)}]");
    })
    .Build();

consumer.Subscribe("orders");

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

var uncommitted = 0;
try
{
    while (!cts.IsCancellationRequested)
    {
        ConsumeResult<string, byte[]>? cr;
        try
        {
            cr = consumer.Consume(TimeSpan.FromMilliseconds(500));
        }
        catch (ConsumeException ex) when (ex.Error.Code == ErrorCode.Local_ValueDeserialization)
        {
            // Poison pill: ship raw bytes to DLQ and advance offset.
            await DeadLetterAsync(ex.ConsumerRecord);
            consumer.Commit(ex.ConsumerRecord); // commit past the bad record
            continue;
        }

        if (cr is null || cr.IsPartitionEOF) continue;

        try
        {
            var order = JsonSerializer.Deserialize<Order>(cr.Message.Value)!;
            await ProjectOrderAsync(order, cts.Token);
        }
        catch (TransientException)
        {
            // e.g., downstream DB hiccup — pause and back off.
            consumer.Pause(new[] { cr.TopicPartition });
            await Task.Delay(TimeSpan.FromSeconds(5), cts.Token);
            consumer.Seek(cr.TopicPartitionOffset); // replay this record
            consumer.Resume(new[] { cr.TopicPartition });
            continue;
        }

        // Async commit every ~1000 records / ~5 seconds.
        if (++uncommitted >= 1000)
        {
            consumer.Commit(); // sync here is fine at 1000-record granularity
            uncommitted = 0;
        }
    }
}
finally
{
    try { consumer.Commit(); } catch { /* best-effort */ }
    consumer.Close(); // triggers cooperative leave-group
}

static Task ProjectOrderAsync(Order o, CancellationToken ct) => Task.CompletedTask;
static Task DeadLetterAsync(ConsumeResult<string, byte[]> r) => Task.CompletedTask;

public record Order(string Id, string CustomerId, decimal Total);
public sealed class TransientException : Exception { }
```

Notes:
- `consumer.Close()` is **not** the same as `Dispose()`. `Close()` performs a cooperative leave-group so the group rebalances immediately. `Dispose()` without `Close()` waits for session timeout — 45 seconds of stuck partitions during graceful shutdown. Always `Close()`.
- `Pause` + `Seek` + `Resume` is how you back off without losing the partition.
- In heavy-throughput pipelines, offload `ProjectOrderAsync` to a `Channel<T>` worker and track committed offsets explicitly (commit only the *highest contiguous processed* offset per partition).

## Senior-level gotchas
- **`Dispose` without `Close`** = 45 s of stuck partitions on shutdown. Always call `Close()` first.
- **Async commit callback exceptions are silently swallowed** unless you attach a `SetErrorHandler`. Your "successful commit" might not have happened.
- **`PartitionsLost` ≠ `PartitionsRevoked`**. Lost means you were evicted — don't commit (you'd overwrite a possibly-ahead offset from the new owner).
- **Auto-commit commits offsets you've *read*, not ones you've *processed***. A crash between poll and side-effect loses messages silently.
- **One poison pill blocks an entire partition indefinitely.** Always have a DLQ path and a "commit past" fallback.
- **`auto.offset.reset=earliest` doesn't mean "always replay"** — it only fires when there's no committed offset. Developers confuse "fresh group" with "resetting offsets".
- **Long processing + default `max.poll.interval.ms=5min`** = sporadic rebalance storms. Either raise it (with `session.timeout.ms` implications), or use pause/resume + worker threads.
- **`max.partition.fetch.bytes < broker message.max.bytes`** = one oversized record stalls the partition. Align them.
- **Rebalance callbacks run on the poll thread.** Long work there = next member times out = cascading rebalance. Keep callbacks < 1 s.
- **`group.instance.id` (static membership) requires unique, stable IDs across restarts.** Using random UUIDs defeats the purpose. Tie to pod name, VM hostname, or instance-id.
- **Consumer lag metrics must be based on `LSO` when `isolation.level=read_committed`**, not log-end-offset, or you'll perpetually report false lag.
- **`enable.partition.eof=true` can surprise integrations**: the consumer returns sentinel EOF records that aren't real messages. Null-check `IsPartitionEOF`.

## References
- [Apache Kafka 4.1 Consumer Configs](https://kafka.apache.org/41/configuration/consumer-configs/)
- [Confluent Platform Consumer Config Reference](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html)
- [Confluent blog: Can your Kafka consumer handle a poison pill?](https://www.confluent.io/blog/spring-kafka-can-your-kafka-consumers-handle-a-poison-pill/)
- [KIP-62: Decouple heartbeats from poll](https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread)
- [KIP-735: Increase default session.timeout.ms to 45 seconds](https://cwiki.apache.org/confluence/display/KAFKA/KIP-735%3A+Increase+default+consumer+session+timeout)
- [Strimzi: Optimizing Kafka consumers](https://strimzi.io/blog/2021/01/07/consumer-tuning/)
- [Confluent.Kafka .NET client reference](https://docs.confluent.io/kafka-clients/dotnet/current/overview.html)
