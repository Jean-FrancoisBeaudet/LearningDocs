# Topics, Partitions, Segments, and Partition Planning

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- A **topic** is a logical name; a **partition** is the unit of parallelism, ordering, and storage. Partitions are the thing you actually operate on.
- Each partition on disk is a set of **segment files** (`.log`), each paired with a sparse **offset index** (`.index`) and a **time index** (`.timeindex`). Brokers roll segments by size (`log.segment.bytes`, default 1 GiB) or time (`log.roll.ms`, default 7d).
- Two cleanup policies: **delete** (drop segments older than `retention.ms` / over `retention.bytes`) or **compact** (keep only the latest record per key). They can also be combined (`cleanup.policy=compact,delete`).
- Partition count planning is a **4-dimensional tradeoff**: target throughput, consumer parallelism ceiling, rebalance cost, and per-broker resource cost (open file handles, memory, metadata load). More partitions ≠ always better.
- Rule of thumb in 2026: aim for **≤4,000 partitions per broker**, **≤200,000 per cluster** (KRaft makes higher feasible but still painful), and enough partitions that each partition absorbs **5–10 MB/s** at your target load.
- **Partition key choice is irreversible in practice.** Keyed records hash by `murmur2(key) % numPartitions`; if you later change partition count, the hash mapping shifts and keyed-ordering guarantees break for existing data. Pick the key like you're picking a primary key — because you are.

## Deep dive

### Anatomy of a partition on disk

A broker's `log.dirs` contains one directory per hosted partition:

```
/var/lib/kafka/data/orders-7/
  00000000000000000000.log
  00000000000000000000.index
  00000000000000000000.timeindex
  00000000000000000000.snapshot
  00000000000012345678.log          ← next segment, base offset 12345678
  00000000000012345678.index
  00000000000012345678.timeindex
  leader-epoch-checkpoint
  partition.metadata
```

- **`.log`** — the actual records, framed in the Kafka record batch format (v2). Immutable once not-active.
- **`.index`** — sparse offset→byte-position map. Entries are 8 bytes (4-byte relative offset + 4-byte position). Broker writes one entry every `log.index.interval.bytes` (default 4 KiB). To find offset O in a segment: binary-search the index to get the closest position ≤ O, then scan forward in the `.log` a few KiB.
- **`.timeindex`** — sparse timestamp→offset map (12 bytes/entry: 8-byte timestamp, 4-byte relative offset). Needed for consumer `offsetsForTimes()` and for time-based retention.
- **`leader-epoch-checkpoint`** — tracks leader epochs for correct truncation after a leader change (KIP-101). Replaces the naive "just trust the high watermark" algorithm.
- **`.snapshot`** — producer-id snapshot for idempotence/transactions recovery.

Only the **active segment** (highest base offset) is being written. All older segments are read-only — that's what enables safe zero-copy reads and makes retention trivial (delete = `rm` of an entire segment).

### Segment rolling

A new segment opens when any of these fire:
- Active segment reaches `segment.bytes` (default **1 GiB**).
- Active segment age exceeds `segment.ms` (default **7 days**).
- Active segment's index or timeindex fills up (`segment.index.bytes`, default 10 MiB).

Big segments = fewer file handles and less metadata, but retention/compaction act at segment granularity, so a 1-GiB segment containing one stale record keeps that record alive until the whole segment rolls and ages out. If your compacted topic has low write rate, **smaller segments make compaction more timely** — drop `segment.bytes` to e.g. 100 MiB.

### Retention: `cleanup.policy=delete`

Delete policy drops whole segments. Two triggers:
- **Time-based**: `retention.ms` (default 7 days). A segment is deletable when its largest timestamp is older than now − `retention.ms`. **Timestamp, not offset** — this is why "retention time lies" when producers set wonky timestamps.
- **Size-based**: `retention.bytes` (default -1, unlimited per partition). When the partition's total log size exceeds this, oldest segments are deleted until it doesn't.

Both can be set; whichever triggers first wins. Deletion is done by the **LogCleaner / log retention thread** (`log.retention.check.interval.ms`, default 5 min). A segment is never partially deleted.

Gotcha: `retention.ms` applies only to **closed** segments. The active segment is never eligible. If your topic writes slowly and `segment.bytes=1GiB` with `segment.ms=7d`, data can live ~2× `retention.ms` in the worst case.

### Retention: `cleanup.policy=compact`

Compaction keeps the **latest value per key**, forever (or until it's overridden or tombstoned). Use cases:
- Stateful materialized views, Kafka Streams `KTable`s.
- Change-log backup of a keyed state store.
- Configuration topics (e.g. `__consumer_offsets` itself is compacted).

Mechanics:
1. The log cleaner thread scans "dirty" segments (those after the last compaction point).
2. It builds an in-memory offset map of `key → highest-offset`.
3. It then rewrites old segments, dropping any record whose offset is not the highest for its key.
4. **Tombstones** (records with null value) are retained for `delete.retention.ms` (default 24h) to give downstream compacted consumers time to observe deletes, then dropped.

Critical configs:
- `min.cleanable.dirty.ratio` (default **0.5**) — don't compact until half of the log is dirty. Lower means more aggressive compaction, more IO.
- `min.compaction.lag.ms` — minimum age before a record is eligible for compaction. Useful so downstream consumers can see intermediate states.
- `max.compaction.lag.ms` — maximum time a dirty record can go uncompacted. Cap this if you need bounded recovery size.

**Hybrid**: `cleanup.policy=compact,delete` gives you "keep latest per key, but also drop any record older than N days regardless". Useful when keys are unbounded (e.g. per-user state) and you want eventual GC.

### Picking the partition count

This is the single most consequential topic-level decision. There are four forces:

**1. Throughput ceiling.** A single partition is a single leader broker writing to a single file. On reasonable hardware a partition absorbs **5–30 MB/s** produce and serves 2–3× that on fetch. Your target topic throughput divided by per-partition throughput gives a floor on partition count. **Confluent's classic formula**: `partitions = max(T/P, T/C)` where T is target throughput, P is per-partition produce throughput, C is per-partition consume throughput.

**2. Consumer parallelism ceiling.** Within a consumer group, one partition is owned by exactly one consumer. If you have 8 partitions you can scale to 8 parallel consumers; a 9th sits idle. **Always pick partition count ≥ max expected consumer fan-out**, with slack for growth.

**3. Rebalance cost.** A rebalance (consumer joins/leaves, coordinator failure, partition count change) redistributes partition ownership. With the old eager protocol this was stop-the-world. With **cooperative-sticky** (default since 2.4, the only sensible choice in 4.x) and **KIP-848** (new group protocol, default for new groups in 4.0), rebalances are incremental — only moved partitions pause. Still: **more partitions = larger state to move around**. In practice, aim for partition count that keeps rebalance duration < 30s.

**4. Per-broker resource cost.**
- **Open file descriptors**: each partition opens its active `.log`, `.index`, `.timeindex` = 3 fds per partition per broker (plus replicas). 10k partitions × 3 ≈ 30k fds. Raise the nofile ulimit (65536 minimum).
- **Memory**: each partition has in-memory structures (offset index, producer state map, leader-epoch cache). Order of ~10 KB/partition steady state, but tail latency gets worse as GC pressure climbs.
- **Metadata**: every partition is a record in `__cluster_metadata`. KRaft scales further than ZK, but the controller still has to replicate and apply those records on failover.
- **Replication fan-out**: each partition's leader maintains a follower fetch session per in-sync replica. RF=3 × 10k partitions = 20k fetch sessions per broker.

**2026 rules of thumb:**
- ≤ **4,000 partition replicas per broker** (leader + follower).
- ≤ **200,000 partitions per cluster** for comfortable operations (KRaft can go higher; Confluent Cloud runs at >1M but at significant operational complexity).
- Size so **each partition gets 5–10 MB/s** of produce at target load. Headroom for 2× spikes.
- Pick partition count in **powers of 2 or multiples of consumer count** so that assignments distribute evenly.

### Choosing the partition key

Default partitioner (4.x, since KIP-480 sticky partitioner is now the default for `null` keys):

- **Non-null key** → `murmur2(key) % numPartitions`. Deterministic routing. Same key always goes to the same partition (as long as partition count doesn't change).
- **Null key** → **sticky partitioner**: batches go to one partition until the batch is closed (by `linger.ms` or `batch.size`), then it rotates. This gives better batching than pure round-robin.

Key selection rubric:
- **What needs strict ordering?** The key must be "whatever the consumer groups by when order matters". If you need per-order event ordering, key by `orderId`. If you need per-customer ordering across all of a customer's orders, key by `customerId`.
- **Is the key's cardinality high enough?** If 90% of traffic is for one mega-tenant, that tenant's partition becomes a hot shard. Consider **composite keys** (`customerId + hash(eventId) % 4`) to fan the hot tenant across 4 partitions while preserving partial order.
- **Is the key stable?** Don't key by anything that changes (e.g. status). It breaks routing.
- **Think about future partition count changes.** If you ever add partitions, the key→partition mapping changes for new writes only; old data stays where it is. Consumers that rely on "this key always lives on partition X" will break. Avoid relying on that invariant.

### Operational patterns

- **Pre-create topics via IaC** (Terraform `confluent_kafka_topic`, or `kafka-topics.sh --create` in a pipeline). Never let `auto.create.topics.enable` do it.
- **Default RF=3, min.insync.replicas=2** for any topic that matters.
- **Tiered storage (KIP-405, GA in 4.0)**: set `remote.storage.enable=true` on a topic and configure an S3/ADLS plugin. Old segments offload to object store; local disk holds the hot tail. Dramatically lowers broker disk cost and lets retention go from days to years without paying for local SSD.

## Config reference

### Topic-level (set with `kafka-topics.sh --config ...` or `kafka-configs.sh`)

| Key | Default | Notes |
|---|---|---|
| `cleanup.policy` | `delete` | Or `compact`, or `compact,delete`. |
| `retention.ms` | 604800000 (7 d) | Time-based retention. |
| `retention.bytes` | -1 | Per-partition size cap. |
| `segment.bytes` | 1073741824 (1 GiB) | Roll trigger. |
| `segment.ms` | 604800000 (7 d) | Time roll trigger. |
| `min.cleanable.dirty.ratio` | 0.5 | Compaction aggressiveness. |
| `min.compaction.lag.ms` | 0 | Delay before compaction eligibility. |
| `max.compaction.lag.ms` | Long.MAX | Force compaction deadline. |
| `delete.retention.ms` | 86400000 (24 h) | Tombstone retention (compact topics). |
| `min.insync.replicas` | 1 | Durability floor — see file 04. |
| `compression.type` | `producer` | Recompress server-side if set. |
| `message.timestamp.type` | `CreateTime` | Or `LogAppendTime` (broker overwrites). |
| `remote.storage.enable` | false | Tiered storage (KIP-405). |
| `local.retention.ms` | -2 (follows retention.ms) | Hot-tier retention when tiered. |

### Broker-level

| Key | Default | Notes |
|---|---|---|
| `log.segment.bytes` | 1 GiB | Cluster-wide default. |
| `log.retention.hours` | 168 | Cluster-wide default. |
| `log.cleaner.threads` | 1 | Compaction parallelism. Bump on heavy compact workloads. |
| `log.cleaner.dedupe.buffer.size` | 128 MiB | Offset map buffer. |
| `log.index.interval.bytes` | 4096 | Sparsity of offset index. |
| `num.partitions` | 1 | Default for auto-created topics. |

## .NET / C# snippet (Confluent.Kafka)

Pick a partition explicitly, produce with a meaningful key, and handle failures.

```csharp
using Confluent.Kafka;
using System.Text.Json;

var config = new ProducerConfig
{
    BootstrapServers      = "broker-1:9092,broker-2:9092,broker-3:9092",
    ClientId              = "orders-producer",
    EnableIdempotence     = true,        // exactly-once-per-partition producer
    Acks                  = Acks.All,
    CompressionType       = CompressionType.Zstd,
    LingerMs              = 10,          // small batching window
    BatchSize             = 64 * 1024,
    MessageMaxBytes       = 1_000_000,
    // Partitioner.Murmur2Random is the default Java-compatible partitioner
    Partitioner           = Partitioner.Murmur2Random,
};

using var producer = new ProducerBuilder<string, string>(config)
    .SetErrorHandler((_, e) => Console.Error.WriteLine($"[producer] {e.Reason}"))
    .Build();

var order = new { OrderId = "O-42", CustomerId = "C-17", Amount = 199.95m };

// Key by customerId so all events for a customer stay ordered on one partition.
var msg = new Message<string, string>
{
    Key     = order.CustomerId,
    Value   = JsonSerializer.Serialize(order),
    Headers = new Headers { { "schema", "orders.v1"u8.ToArray() } },
};

try
{
    var dr = await producer.ProduceAsync("orders", msg);
    Console.WriteLine($"Produced to {dr.TopicPartitionOffset}");
}
catch (ProduceException<string, string> ex)
{
    // ex.Error.IsFatal => broker rejected (e.g. auth, invalid topic); abort.
    // Retryable errors are already retried internally by librdkafka.
    Console.Error.WriteLine($"Produce failed: {ex.Error.Reason}");
    throw;
}

producer.Flush(TimeSpan.FromSeconds(10));
```

Two points a mid-level dev misses:
1. `EnableIdempotence=true` implies `Acks=All`, `MaxInFlight <= 5`, and `Retries=int.MaxValue`. Don't fight it.
2. `Partitioner.Murmur2Random` matches the Java default so cross-language producers agree on partition assignment for the same key. If you mix Java and .NET producers writing to the same topic and you use a non-default partitioner on one side, keys diverge.

## Senior-level gotchas

- **You can add partitions but never remove them.** And adding them changes the hash distribution — existing keyed ordering guarantees break for any new messages of existing keys that now hash to a different partition. Treat partition count as immutable in practice.
- **Retention is timestamp-based, and producers control timestamps by default.** A buggy client setting a future timestamp will keep data alive forever. Defend with `message.timestamp.type=LogAppendTime` on topics where you don't trust producer timestamps (loses `CreateTime` semantics though).
- **Compaction + transactions**: compacted topics can have transactional markers. The compactor preserves them correctly, but if you're writing a custom consumer that reads past tombstones you must handle abort markers.
- **`__consumer_offsets` is itself a compacted topic** with 50 partitions by default (`offsets.topic.num.partitions`). If your offset commit rate is enormous, this one topic can bottleneck consumer group operations.
- **Hot partitions silently.** Monitor per-partition produce rate. A skewed key distribution (one mega-customer) will pin 90% of traffic to one partition and you'll blame "Kafka latency" when it's your key choice.
- **Timeindex files fill faster than offset indexes** (12 vs 8 bytes per entry, 50% larger). On high-rate topics, timeindex can trigger segment rolls earlier than expected.
- **Don't compact what you should delete.** Compaction is expensive CPU + IO. If you don't need "latest per key" semantics, use delete.
- **Tiered storage changes your latency profile for historical reads.** Hot-tier reads are page-cache fast; cold-tier reads hit S3 with 50–200 ms latency per segment. Consumers doing a from-beginning read of a year-old topic will feel this. Plan batch jobs accordingly.
- **Segment files are memory-mapped in some paths.** `vm.max_map_count` on Linux matters for high-partition brokers. Default of 65536 is often too low.

## References
- [Conduktor — Kafka Storage Internals: Segments, Indexing, and Why Retention Time Lies](https://www.conduktor.io/blog/understanding-kafka-s-internal-storage-and-log-retention)
- [Strimzi blog — Deep dive into Apache Kafka storage internals](https://strimzi.io/blog/2021/12/17/kafka-segment-retention/)
- [Confluent — How to Choose the Number of Topics/Partitions](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
- [Confluent — Apache Kafka Partition Strategy](https://www.confluent.io/learn/kafka-partition-strategy/)
- [Confluent docs — Log Compaction](https://docs.confluent.io/kafka/design/log_compaction.html)
- [KIP-405: Kafka Tiered Storage](https://cwiki.apache.org/confluence/display/KAFKA/KIP-405%3A+Kafka+Tiered+Storage)
- [AWS — Best practices for right-sizing your Apache Kafka clusters](https://aws.amazon.com/blogs/big-data/best-practices-for-right-sizing-your-apache-kafka-clusters-to-optimize-performance-and-cost/)
