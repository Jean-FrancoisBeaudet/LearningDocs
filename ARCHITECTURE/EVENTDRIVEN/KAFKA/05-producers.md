# Kafka Producers — Deep Dive

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- The producer is an **asynchronous, batching, in-memory-buffered** client. Records go into a partition-keyed `RecordAccumulator`, are drained by a background I/O thread, and flushed to the partition leader over a single TCP connection per broker.
- **Idempotence is ON by default** since Kafka 3.0 (KIP-679). That forces `acks=all`, `retries=Int32.MaxValue`, `max.in.flight.requests.per.connection <= 5`, and gives you exactly-once *per partition, per producer session* with zero config effort.
- Durability is controlled by `acks` (`0` / `1` / `all`). For any production system that cares about data loss, `acks=all` combined with broker-side `min.insync.replicas >= 2` is the only correct answer.
- Throughput is dominated by **batching** (`linger.ms`, `batch.size`) and **compression** (`zstd` for best ratio, `lz4`/`snappy` for lowest CPU). These are the two biggest tuning levers; everything else is second-order.
- Back-pressure is implicit: when `buffer.memory` fills, `Send` blocks up to `max.block.ms`, then throws. Treating the producer as fire-and-forget without understanding this is a common outage source.
- Partitioning since 2.4 uses the **sticky partitioner**; since 3.3 the **uniform-sticky (built-in)** partitioner is the default for keyless records (`partitioner.class=null`). Don't write a custom partitioner unless you truly need locality or co-partitioning across topics.

## Deep dive

### Anatomy of a send
A call to `IProducer<K,V>.Produce()` / `ProduceAsync()` does **not** perform any network I/O on the calling thread. It:

1. Serializes `key` and `value` (and headers).
2. Selects a partition via the configured partitioner.
3. Appends the record to the partition's in-memory batch inside the `RecordAccumulator` (bounded by `buffer.memory`, default **32 MiB**).
4. Returns immediately; the background `Sender` thread eventually drains ready batches and issues a `ProduceRequest` to the partition leader.

This is why `ProduceAsync` returns a `Task<DeliveryResult>` that only completes once the broker has acknowledged (per `acks`). The callback-based `Produce(...)` overload is faster (no `Task` allocation) and is recommended for high-throughput fire-and-forget style.

### `acks` — the durability knob
| Value | Leader waits for | Durability | Latency |
|-------|------------------|------------|---------|
| `0`   | Nothing — not even leader write | **Data loss on any failure** | Lowest |
| `1`   | Leader local write | Lost if leader crashes before replication | Medium |
| `all` (a.k.a. `-1`) | Leader + all in-sync replicas | Bounded by `min.insync.replicas` | Highest |

`acks=all` alone is **not** enough. You need broker-side `min.insync.replicas=2` (with RF=3) to tolerate a single broker loss *without* silently accepting writes on an under-replicated partition. `acks=all` + `min.insync.replicas=1` = lies.

When idempotence is enabled (default), the client **forces `acks=all`**. Setting `acks=1` with idempotence explicitly enabled throws `ConfigException`.

### Idempotent producer (KIP-98)
Enabled by default via `enable.idempotence=true`. Mechanics:

- On first send, the producer requests a **Producer ID (PID)** from the transaction coordinator.
- Each record carries `(PID, epoch, sequence_number)` per partition. The broker maintains a small cache (last 5 batches per partition) and rejects/deduplicates based on sequence number.
- On a retry after a transient network failure, the broker sees the same `(PID, seq)` and responds with `DUPLICATE_SEQUENCE_NUMBER` (internally treated as success, no duplicate appended).
- Scope: **single producer session, per partition**. A producer restart gets a new PID — so idempotence alone does **not** give you EOS across process restarts. That requires transactions (see `08-delivery-semantics.md`).

Gotcha: idempotence is bounded to 5 in-flight requests per connection because the broker sequence-number cache only tracks the last 5 batches. Setting `max.in.flight.requests.per.connection=6` while idempotence is on throws at producer construction.

### Batching: `linger.ms` and `batch.size`
The producer accumulates records into partition-level batches. A batch is drained when either:

- the batch reaches `batch.size` bytes (default **16 KiB**), OR
- `linger.ms` elapses since the first record landed in the batch (default **5 ms** in Kafka 3.x+; historically **0**).

Non-obvious behavior:

- `batch.size` is an **upper bound** on a single request per partition, not a minimum. With `linger.ms=0` you still get small batches under light load.
- Compression ratios scale with batch size. Sending 100-byte records with `batch.size=16KB` and `linger.ms=5` gives ~10x better compression than `linger.ms=0`.
- Setting `linger.ms=20–100` is normal for throughput-optimized pipelines. The worst-case added latency is bounded and predictable.
- `batch.size` of 64–256 KiB is common for log-ingestion workloads. Beyond ~1 MiB you're fighting `max.request.size` (default 1 MiB) and `message.max.bytes` on the broker.

### Compression
| Codec  | CPU cost | Ratio | Latency profile | When to use |
|--------|----------|-------|-----------------|-------------|
| `none` | 0 | 1.0x | Lowest | Tiny payloads, LAN-only |
| `snappy` | Low | ~2x | Low | Legacy default, fine |
| `lz4` | Lowest (fastest) | ~2x | Lowest | Real-time, CPU-constrained |
| `gzip` | High | ~4x | Highest | Archival, bandwidth-bound |
| `zstd` | Medium | **~4–5x** | Medium | **Modern default** — best overall (KIP-110) |

`zstd` was added in Kafka 2.1 and is the recommended default since 3.x. It beats `gzip` on both ratio and speed, and beats `snappy`/`lz4` dramatically on ratio. `compression.level` can be tuned for zstd (1–22, default 3).

Compression is applied **per batch**, end-to-end: the producer compresses the batch, the broker stores the compressed batch as-is (assuming `compression.type=producer` at topic level, the default), the consumer decompresses. This means setting a topic-level compression other than `producer` forces the broker to recompress on write — avoid unless you're migrating codecs.

### Partitioners
- **Keyed records**: `hash(key) mod numPartitions` (murmur2). This is a hash of the serialized bytes — changing serializers changes routing. Stable.
- **Null-key records**:
  - Before 2.4: round-robin — pathological, produced 1 batch per partition per record batch → tiny batches → terrible throughput.
  - 2.4–3.2: **sticky partitioner** (KIP-480) — stick to one partition until the current batch fills, then rotate. Dramatically better batching.
  - 3.3+: **uniform-sticky** (built-in default, KIP-794) — sticky, but rotation is weighted by broker latency/queue, avoiding hot-spotting a slow broker.
- **Custom partitioner**: implement when you need co-partitioning across topics (e.g., customer-id must hash identically across `orders` and `payments` topics) or locality (prefer same-rack partitions). In Confluent.Kafka, set `Partitioner` on `ProducerConfig` or use the `SetPartitioner` builder hook.

Setting `partitioner.class` explicitly overrides the sticky default — don't do this unless you have a reason. Also note that `partitioner.ignore.keys=true` (KIP-794) makes the partitioner ignore keys for sticky behavior even on keyed records — useful when your keys only matter for compaction, not routing.

### Back-pressure, buffer memory, blocking
- `buffer.memory` (default **32 MiB**): total bytes the producer can buffer across all partitions.
- `max.block.ms` (default **60000**): how long `Produce`/`Send` will block waiting for buffer space *or* metadata. After this, it throws.
- `delivery.timeout.ms` (default **120000**, KIP-91): total wall-clock from `Produce()` to successful ack, including retries and `request.timeout.ms`. This is the **only** timeout you should reason about for SLA purposes; `retries` and `retry.backoff.ms` are now implementation details under this umbrella.

Common failure mode: downstream broker slowdown → batches accumulate → `buffer.memory` fills → callers block on `Produce()` → upstream caller thread-pool starves → cascading failure. Mitigations:

- Set `buffer.memory` to reflect acceptable in-flight data loss on process crash (it's RAM — it dies with the process).
- Set `max.block.ms` **lower than** your upstream timeout. If upstream gives you 5 s, don't block for 60 s.
- Emit metrics from `Statistics` events (Confluent.Kafka `StatisticsHandler`): `queue.size`, `buffer.available`, `tx`, `tx_bytes`.

### `max.in.flight.requests.per.connection`
Default **5**. Interaction table:

| Scenario | Value | Effect |
|---|---|---|
| Idempotence on (default) | 1–5 | Ordering + no duplicates guaranteed |
| Idempotence on | `>5` | `ConfigException` at startup |
| Idempotence off, `retries>0` | `>1` | **Reordering possible** on retry — avoid |
| Strict ordering without idempotence | `1` | Safe but throttles throughput |

Historical advice said "set to 1 for ordering"; with idempotence this is obsolete — you get ordering at 5, with 5x the throughput.

### Headers
`Headers` are `IEnumerable<IHeader>` of `(string key, byte[] value)` pairs, serialized alongside each record. Typical uses: correlation IDs, tenant IDs, schema ID, content-type, tracing context (W3C `traceparent`). They bypass the key/value serializer and are **not** compacted on (no special treatment by log compaction).

### Serializers
Confluent.Kafka ships built-in serializers for `string`, `int`, `long`, `float`, `double`, `byte[]`, `null`, `Ignore`. For Avro/Protobuf/JSON Schema use `Confluent.SchemaRegistry.Serdes.*` with a Schema Registry client. Schema Registry integration:

- Caches schema IDs locally after first registration.
- Subject naming strategy defaults to `TopicNameStrategy` (`<topic>-key`, `<topic>-value`); override with `RecordNameStrategy` or `TopicRecordNameStrategy` if you have multiple event types per topic.
- `auto.register.schemas=false` in production — schemas should be registered out-of-band by CI.

## Config reference

| Key | Default | Notes |
|-----|---------|-------|
| `bootstrap.servers` | (required) | CSV of `host:port`; one is enough, more gives bootstrap redundancy |
| `acks` | `all` (when idempotence on) | `0`/`1`/`all`. Durability knob. |
| `enable.idempotence` | `true` | Since 3.0. Forces `acks=all`, `retries=MAX`, `mif<=5` |
| `retries` | `Int32.MaxValue` | Bounded in practice by `delivery.timeout.ms` |
| `delivery.timeout.ms` | `120000` | Total ack deadline — this is the SLA knob |
| `request.timeout.ms` | `30000` | Per-request timeout; must be ≤ `delivery.timeout.ms` |
| `linger.ms` | `5` | Time to wait to fill a batch |
| `batch.size` | `16384` (16 KiB) | Per-partition batch upper bound |
| `buffer.memory` | `33554432` (32 MiB) | Total producer buffer |
| `max.block.ms` | `60000` | Max block on full buffer / metadata wait |
| `compression.type` | `none` | Use `zstd` in production |
| `compression.level` | codec default | zstd: 1–22, default 3 |
| `max.in.flight.requests.per.connection` | `5` | ≤5 required with idempotence |
| `max.request.size` | `1048576` (1 MiB) | Must be ≤ broker `message.max.bytes` |
| `partitioner.class` | built-in uniform-sticky | Only override for co-partitioning |
| `partitioner.ignore.keys` | `false` | Set true to force sticky on keyed records |
| `client.id` | `""` | Set it — shows up in broker metrics, quotas |
| `transactional.id` | `null` | Turns on transactions; must be stable across restarts |
| `transaction.timeout.ms` | `60000` | See `08-delivery-semantics.md` |

## .NET / C# snippet (Confluent.Kafka)

```csharp
using Confluent.Kafka;
using System.Text;
using System.Text.Json;

var config = new ProducerConfig
{
    BootstrapServers      = "broker1:9092,broker2:9092,broker3:9092",
    ClientId              = $"orders-producer-{Environment.MachineName}",

    // Durability: idempotence is default-on in librdkafka 2.x, but be explicit.
    EnableIdempotence     = true,          // forces Acks=All + retries
    Acks                  = Acks.All,
    MessageSendMaxRetries = int.MaxValue,  // bounded by MessageTimeoutMs

    // Throughput: batch hard, compress with zstd.
    LingerMs              = 20,
    BatchSize             = 256 * 1024,    // 256 KiB
    CompressionType       = CompressionType.Zstd,
    CompressionLevel      = 3,

    // Back-pressure: bound blocking so upstream never stalls indefinitely.
    QueueBufferingMaxKbytes = 64 * 1024,   // 64 MiB (librdkafka name for buffer.memory)
    MessageTimeoutMs      = 30_000,        // librdkafka equivalent of delivery.timeout.ms

    // Ordering + idempotence: 5 in-flight is fine.
    MaxInFlight           = 5,

    // TLS/SASL omitted for brevity.
};

using var producer = new ProducerBuilder<string, byte[]>(config)
    .SetKeySerializer(Serializers.Utf8)
    .SetValueSerializer(Serializers.ByteArray)
    .SetErrorHandler((_, e) =>
    {
        // Fatal errors require recreating the producer (e.g., InvalidProducerEpoch on fencing).
        Console.Error.WriteLine($"[producer] {e.Code}: {e.Reason} (fatal={e.IsFatal})");
    })
    .SetLogHandler((_, log) => Console.WriteLine($"[{log.Level}] {log.Message}"))
    .Build();

// Graceful shutdown: flush on Ctrl+C / SIGTERM.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

async Task PublishOrderAsync(Order order, CancellationToken ct)
{
    var bytes = JsonSerializer.SerializeToUtf8Bytes(order);
    var msg = new Message<string, byte[]>
    {
        Key     = order.CustomerId,
        Value   = bytes,
        Headers = new Headers
        {
            { "content-type", Encoding.UTF8.GetBytes("application/json") },
            { "schema",       Encoding.UTF8.GetBytes("order.v1") },
            { "traceparent",  Encoding.UTF8.GetBytes(Activity.Current?.Id ?? "") }
        }
    };

    try
    {
        var dr = await producer.ProduceAsync("orders", msg, ct);
        Console.WriteLine($"delivered order={order.Id} → {dr.TopicPartitionOffset}");
    }
    catch (ProduceException<string, byte[]> ex) when (ex.Error.Code == ErrorCode.Local_QueueFull)
    {
        // buffer.memory exhausted — apply upstream back-pressure.
        throw new BackPressureException("producer queue full", ex);
    }
}

// On shutdown: give in-flight batches a chance to land.
producer.Flush(TimeSpan.FromSeconds(10));

public record Order(string Id, string CustomerId, decimal Total);
public sealed class BackPressureException(string msg, Exception inner) : Exception(msg, inner);
```

Notes on the snippet:
- `ProduceAsync` is awaited — fine for moderate throughput. For >50k msg/s, use the callback form `producer.Produce(topic, msg, deliveryHandler)` to avoid `Task` allocations.
- `librdkafka` (the C library underneath Confluent.Kafka) uses **slightly different config names** (`queue.buffering.max.ms` ≈ `linger.ms`, `queue.buffering.max.kbytes` ≈ `buffer.memory`, `message.timeout.ms` ≈ `delivery.timeout.ms`). The C# strongly-typed `ProducerConfig` maps most of these, but if you read docs from Apache Kafka Java docs, be aware of the alias.
- Always `Flush` before `Dispose`. `Dispose` alone will wait up to a fixed timeout then drop in-flight messages.

## Senior-level gotchas
- **Idempotence ≠ EOS across restarts.** PID is ephemeral. Use transactions if the producer restarts and you need dedup across sessions.
- **`acks=all` without `min.insync.replicas>=2` is a lie.** With RF=3 and `min.isr=1`, a lagging replica set collapses to "leader only" and you're back to `acks=1` durability silently.
- **`max.block.ms=60000` is too long for most online systems.** Drop it to ~2–5 s so back-pressure propagates fast.
- **Callback thread pool is single-threaded** in librdkafka. Never call `Produce` from inside a delivery report callback — you'll deadlock. Hand off to a `Channel<T>` or `Task.Run`.
- **Sticky partitioner can create hot partitions** under low cardinality keys. Monitor partition skew; consider `partitioner.ignore.keys=true` for compaction-only keys.
- **Schema Registry `auto.register.schemas=true` in prod** is how half the world has accidentally registered broken schemas. Turn it off.
- **Headers aren't free.** Each header is serialized per record; 20 headers × 50 bytes × 100k msg/s = ~100 MB/s of header overhead. Audit what you propagate.
- **zstd `compression.level` > 10 is rarely worth it** — diminishing returns and sharply rising CPU. Leave at default 3 unless benchmarking shows otherwise.
- **`transactional.id` must be stable per logical producer instance.** Generating a new one per process restart defeats fencing and you can get zombies. Pair with a stable pod/hostname, not a GUID.
- **Never share a producer across transactions.** One `IProducer` = one `transactional.id` = one in-flight transaction at a time.

## References
- [Apache Kafka 4.1 Producer Configs](https://kafka.apache.org/41/configuration/producer-configs/)
- [Confluent Platform Producer Config Reference](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html)
- [KIP-679: Enable idempotence by default](https://issues.apache.org/jira/browse/KAFKA-10619)
- [Apache Kafka Message Compression (Confluent blog)](https://www.confluent.io/blog/apache-kafka-message-compression/)
- [Confluent.Kafka .NET client docs](https://docs.confluent.io/kafka-clients/dotnet/current/overview.html)
- [KIP-480: Sticky Partitioner](https://cwiki.apache.org/confluence/display/KAFKA/KIP-480%3A+Sticky+Partitioner)
- [KIP-794: Strictly Uniform Sticky Partitioner](https://cwiki.apache.org/confluence/display/KAFKA/KIP-794%3A+Strictly+Uniform+Sticky+Partitioner)
