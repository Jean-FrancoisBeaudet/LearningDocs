# Log Compaction and Retention

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR

Kafka topics are append-only logs. Kafka reclaims disk space via a per-topic
**cleanup policy**:

- `cleanup.policy=delete` (default) — time- or size-based **retention**. Whole
  log segments are dropped when they age out (`retention.ms`) or when the
  partition exceeds a size cap (`retention.bytes`). This is the "queue-like"
  mode used for event streams, clickstreams, metrics, logs, etc.
- `cleanup.policy=compact` — **log compaction**. Kafka keeps at least the
  **latest value per key** indefinitely; old versions of the same key are
  garbage-collected by the background `log.cleaner` thread pool. This is the
  "changelog / table" mode used for CDC, Kafka Streams state stores, Connect
  offsets, `__consumer_offsets`, and any "current state per entity" topic.
- `cleanup.policy=compact,delete` — both: compact old keys **and** also expire
  records older than `retention.ms`. Useful for bounded-history changelogs
  (e.g. "latest value per user, but discard users inactive >90d").

Compaction runs asynchronously and does not block producers. It is
**eventually** consistent: there is always a "dirty" head of the log that
still contains duplicates; only the "clean" tail is guaranteed to have a
single record per key. Tombstones (records with `null` value) are how you
**delete** a key from a compacted topic; they are themselves retained for
`delete.retention.ms` (default 24 h) so consumers have time to observe the
deletion before the tombstone itself is cleaned away.

The most common operational pain is **the cleaner falling behind**: under
heavy write load or with small `log.cleaner.threads`, the cleaner cannot keep
up, partitions grow without bound, and memory / disk pressure builds on the
broker. KRaft does not change any of this — the cleaner is a broker-local
component independent of the metadata quorum.

---

## Deep dive

### 1. Retention (`cleanup.policy=delete`)

#### Time-based

```
retention.ms = 604800000   # 7 days (default)
```

- Kafka checks the **largest timestamp** in a closed segment against
  `now - retention.ms`. If the entire segment is older, the segment file is
  deleted.
- Retention operates on whole **segments**, never on individual records.
  A partition whose newest record is 6 days old will still hold segments
  from day 0 because the active segment has not rolled.
- Segments roll when they reach `segment.bytes` (default 1 GiB) or
  `segment.ms` (default 7 days). If your traffic is low, a segment may sit
  open for a long time and retention appears "stuck" — force rolling with
  `segment.ms` smaller than `retention.ms` (common rule: `segment.ms ≈
  retention.ms / 4` to `/10`).
- `message.timestamp.type` = `CreateTime` (producer-assigned) or
  `LogAppendTime` (broker-assigned). With `CreateTime` a misbehaving
  producer can send future-dated records and pin the segment forever. Use
  `LogAppendTime` if producers are untrusted.

#### Size-based

```
retention.bytes = -1       # default: unlimited
```

- Per-**partition** cap (not per-topic). A topic with 100 partitions and
  `retention.bytes=1GB` can keep up to 100 GiB.
- Applied **in addition to** `retention.ms` — whichever limit triggers
  first wins. Setting both lets you bound worst-case disk while keeping a
  time SLA.

#### Broker-level defaults vs topic overrides

The broker config uses `log.retention.ms`, `log.retention.bytes`,
`log.segment.bytes`, etc. Topic-level configs (`retention.ms`,
`retention.bytes`, `segment.bytes`) override them per topic and are the
ones you usually set. `kafka-configs.sh --alter --entity-type topics` is the
supported path.

### 2. Log compaction (`cleanup.policy=compact`)

#### Data model

A compacted topic is a **keyed changelog**. For every distinct key, Kafka
guarantees that the **latest record** (by offset) is retained. Intermediate
values for that key may be deleted at any time after compaction runs.

Ordering guarantee: per partition, offsets are still strictly increasing.
Compaction preserves offsets of surviving records (no reshuffling) — it
merely punches holes in the log. Consumers may therefore see non-contiguous
offsets; this is normal, do not panic on "offset gaps".

#### The log's dirty/clean split

Each partition's log is split in two regions:

```
|-------- clean ---------|--------- dirty (head) --------|
0                      cleanerCheckpoint            log end offset (LEO)
```

- **Clean tail**: already compacted. Every key appears at most once.
- **Dirty head**: everything appended since the last compaction pass; may
  contain duplicates.

The cleaner's job is to scan the dirty head, build an in-memory key->latest-
offset map (the "OffsetMap", bounded by `log.cleaner.dedupe.buffer.size`),
and rewrite segments so older duplicates vanish. The checkpoint advances.

#### When does compaction run?

The cleaner picks the partition with the **highest dirty ratio** above a
threshold:

```
dirtyRatio = dirtyBytes / (dirtyBytes + cleanBytes)
```

- `log.cleaner.min.cleanable.ratio` (default `0.5`) — minimum ratio to
  schedule the partition. Lower it (e.g. `0.1`) to compact more aggressively
  at the cost of more I/O.
- `min.compaction.lag.ms` (default `0`) — minimum time a record must sit
  in the log before it can be compacted. Lets consumers read recent history.
- `max.compaction.lag.ms` (default `Long.MAX_VALUE`) — KIP-354 hard upper
  bound: if a dirty record exceeds this age, the partition is scheduled
  regardless of dirty ratio. **Set this** for GDPR/PII: it caps how long
  a "deleted" key can survive before the tombstone takes effect.

#### The cleaner threads

- `log.cleaner.enable=true` (default).
- `log.cleaner.threads` (default `1`). Each thread cleans one partition at a
  time. For a broker with hundreds of compacted partitions you typically
  need 4–8.
- `log.cleaner.dedupe.buffer.size` (default 128 MiB, broker-wide, split
  across threads). The OffsetMap lives here; if a partition's unique-key
  count in the dirty head exceeds what fits, the cleaner processes the
  partition in multiple passes — slower but still correct.
- `log.cleaner.io.max.bytes.per.second` — throttle to protect producers.
- `log.cleaner.backoff.ms` (default 15 s) — sleep between idle scans.

#### Active segment is never compacted

The cleaner only touches **closed** segments. Records in the active segment
(the one currently being written) are always kept. This means a low-volume
compacted topic can still show duplicates for a long time — force rolling
with `segment.ms` (e.g. 1 hour for a slow CDC topic).

### 3. Tombstones and deletion

To delete key `K` from a compacted topic, produce `(K, null)`. This is a
**tombstone**. During compaction:

1. The tombstone supersedes any prior value for `K`.
2. The tombstone itself is kept for `delete.retention.ms` (default
   `86400000` = 24 h) **after** the segment containing it becomes clean.
3. After that window, the tombstone is also removed — the key is gone.

`delete.retention.ms` must be **long enough for every consumer to read it**.
Rule of thumb: `delete.retention.ms >= max consumer lag + safety margin`.
For Kafka Streams / Connect state rebuild, 24 h is usually fine; for slow
offline batch consumers, raise it.

Null **keys** are different — see gotchas.

### 4. `cleanup.policy=compact,delete`

Both policies run. Useful when:

- A CDC stream where you only need "current state per entity for the last
  30 days" and want to drop dormant entities.
- Streams changelog where you want to bound total history regardless of
  whether a key is still live.

Compaction + delete ordering: records are subject to size/time retention
first (entire segment deleted when old enough), and what remains is
compacted. You may lose the latest value of a key if the entire segment
carrying it is time-expired before a newer version is written — so only use
this policy when dropping cold keys is acceptable.

### 5. Use cases for compacted topics

- **CDC** (Debezium): each row's primary key -> latest row state.
  Tombstone = DELETE. Consumers can bootstrap a full snapshot by reading
  from offset 0.
- **Kafka Streams state stores**: every KTable / aggregate is backed by a
  compacted **changelog topic** (`<app-id>-<store>-changelog`). On failover,
  the new task replays the compacted changelog to rebuild RocksDB.
- **Connect offsets / configs / status**: `connect-offsets`,
  `connect-configs`, `connect-status` are all compacted.
- **`__consumer_offsets`** and **`__transaction_state`** are compacted.
- **Reference-data topics** for stream-table joins (customers, products).
- **Feature store / cache warm-up** for downstream services that want a
  "latest-known value" subscription.

### 6. Reading a compacted topic from offset 0

A new consumer with `auto.offset.reset=earliest` gets a **snapshot** of the
current state — one record per key in the clean tail, plus whatever is in
the dirty head. This is the foundation of "log-as-database" patterns. The
snapshot is **not transactionally consistent** (it's an eventually-
consistent merge of compaction passes), but for most state-sync use cases it
is fine.

### 7. Compaction and transactions / EOS

Transactional producers write commit/abort markers (control records) to
partitions. Compaction preserves these. EOS (exactly-once semantics) for
Streams relies on compacted changelogs + transactions; ensure
`min.compaction.lag.ms` is large enough that in-flight transactions are
never raced by the cleaner (Streams sets sane defaults, but check if you
author custom Processor API code).

---

## Config reference

### Topic-level

| Config | Default | Meaning |
|---|---|---|
| `cleanup.policy` | `delete` | `delete`, `compact`, or `compact,delete` |
| `retention.ms` | `604800000` (7 d) | Max age of a segment (delete policy) |
| `retention.bytes` | `-1` | Max bytes per partition (delete policy) |
| `segment.bytes` | `1073741824` (1 GiB) | Segment roll size |
| `segment.ms` | `604800000` (7 d) | Segment roll age — lower it on low-traffic topics |
| `min.compaction.lag.ms` | `0` | Min age before a record may be compacted |
| `max.compaction.lag.ms` | `Long.MAX_VALUE` | Hard ceiling; forces compaction (KIP-354) |
| `min.cleanable.dirty.ratio` | `0.5` | Dirty ratio trigger |
| `delete.retention.ms` | `86400000` (24 h) | Tombstone retention after cleaning |
| `message.timestamp.type` | `CreateTime` | Use `LogAppendTime` to prevent producer-side timestamp abuse |
| `message.timestamp.difference.max.ms` | `Long.MAX_VALUE` | Reject records with skewed timestamps |

### Broker-level (cleaner tuning)

| Config | Default | Meaning |
|---|---|---|
| `log.cleaner.enable` | `true` | Master switch |
| `log.cleaner.threads` | `1` | Cleaner thread pool size — raise to 4–8 on busy brokers |
| `log.cleaner.dedupe.buffer.size` | `134217728` (128 MiB) | Shared OffsetMap memory |
| `log.cleaner.io.buffer.size` | `524288` | Per-cleaner I/O buffer |
| `log.cleaner.io.max.bytes.per.second` | `Double.MAX_VALUE` | Throttle |
| `log.cleaner.min.cleanable.ratio` | `0.5` | Cluster-default dirty ratio |
| `log.cleaner.backoff.ms` | `15000` | Idle sleep |
| `log.cleaner.delete.retention.ms` | `86400000` | Cluster-default tombstone retention |

---

## .NET / C# snippet (Confluent.Kafka)

Compacted topics are a **broker-side** concern; the client just produces and
consumes. Below: creating a compacted topic via AdminClient, producing a
value, producing a tombstone, and consuming to rebuild in-memory state.

```csharp
using Confluent.Kafka;
using Confluent.Kafka.Admin;

var bootstrap = "localhost:9092";
var topic     = "customer-state";

// 1. Create the topic as compacted with tight compaction lag for fast
//    tombstone propagation. delete.retention.ms = 1h for this demo.
using (var admin = new AdminClientBuilder(
    new AdminClientConfig { BootstrapServers = bootstrap }).Build())
{
    await admin.CreateTopicsAsync(new[]
    {
        new TopicSpecification
        {
            Name              = topic,
            NumPartitions     = 6,
            ReplicationFactor = 3,
            Configs = new Dictionary<string, string>
            {
                ["cleanup.policy"]          = "compact",
                ["min.cleanable.dirty.ratio"] = "0.1",
                ["min.compaction.lag.ms"]   = "60000",       // 1 min
                ["max.compaction.lag.ms"]   = "3600000",     // 1 h  (GDPR bound)
                ["delete.retention.ms"]     = "3600000",     // 1 h
                ["segment.ms"]              = "600000",      // 10 min (low-volume)
            }
        }
    });
}

// 2. Produce a keyed value, then a tombstone.
var producerCfg = new ProducerConfig
{
    BootstrapServers = bootstrap,
    EnableIdempotence = true,
    Acks = Acks.All
};

using var producer = new ProducerBuilder<string, string?>(producerCfg).Build();

await producer.ProduceAsync(topic,
    new Message<string, string?> { Key = "customer-42", Value = "{\"tier\":\"gold\"}" });

// Tombstone -> delete customer-42 from the compacted topic
await producer.ProduceAsync(topic,
    new Message<string, string?> { Key = "customer-42", Value = null });

producer.Flush(TimeSpan.FromSeconds(5));

// 3. Bootstrap "latest state per key" by reading from the beginning.
var consumerCfg = new ConsumerConfig
{
    BootstrapServers = bootstrap,
    GroupId          = $"state-loader-{Guid.NewGuid()}",
    AutoOffsetReset  = AutoOffsetReset.Earliest,
    EnableAutoCommit = false
};

using var consumer = new ConsumerBuilder<string, string?>(consumerCfg).Build();
consumer.Subscribe(topic);

var state = new Dictionary<string, string>();

while (true)
{
    var cr = consumer.Consume(TimeSpan.FromSeconds(1));
    if (cr is null) break;                       // idle — assume caught up
    if (cr.Message.Value is null) state.Remove(cr.Message.Key);   // tombstone
    else state[cr.Message.Key] = cr.Message.Value;
}

Console.WriteLine($"Loaded {state.Count} live customers from compacted topic.");
```

Key implementation details:

- Use `string?` (or `byte[]?`) for the value type so you can actually send
  `null`. If you use a non-nullable serializer the tombstone will be encoded
  as an empty payload, which is **not** a tombstone — Kafka checks for a
  literal null value.
- With Avro/Protobuf/JSON schema serializers, pass `null` value directly;
  the Confluent serializers short-circuit and write a true null.
- Never use a tombstone with a **null key** — that record will be ignored by
  the cleaner (see gotcha below).

---

## Senior-level gotchas

1. **Null-key records on a compacted topic are a black hole.** The cleaner
   has no key to dedupe on; such records sit in the dirty head forever until
   the partition is force-expired. Either reject null keys at the producer,
   or set `cleanup.policy=compact,delete` so time-based retention sweeps
   them up.

2. **Active segment is never compacted.** On a low-volume CDC topic, your
   "deleted" key can live for the entire `segment.ms` before even becoming
   eligible. Set `segment.ms` in hours, not days, for compacted topics.

3. **Tombstone visibility vs tombstone lifetime.** `delete.retention.ms` is
   the window in which a downstream consumer must observe the tombstone.
   If a consumer is lagging by more than that, it will miss the delete and
   hold stale state. Monitor consumer lag on compacted topics as carefully
   as on normal ones.

4. **Cleaner falling behind.** Symptoms: partition dir grows unboundedly,
   `kafka.log.LogCleaner` JMX `max-dirty-ratio` climbs to 1.0, uncleanable
   partitions log warnings. Fixes: raise `log.cleaner.threads`, raise
   `log.cleaner.dedupe.buffer.size`, lower `min.cleanable.dirty.ratio`,
   investigate a single giant-cardinality topic hogging a thread. The
   cleaner is **single-threaded per partition**, so one pathological
   partition blocks one cleaner thread.

5. **Cleaner OOM / "buffer overflow" errors.** Means the unique-key count in
   a partition's dirty head exceeds what fits in its share of the dedupe
   buffer. Kafka will do multi-pass compaction, but throughput tanks.
   Either raise `log.cleaner.dedupe.buffer.size` or shorten
   `max.compaction.lag.ms` so the dirty head stays smaller.

6. **Offset gaps are normal.** After compaction consumers see
   `offset 101, 104, 108`. Do not write code that asserts contiguous
   offsets on compacted topics.

7. **Reading from offset 0 is not atomic.** Two consumers bootstrapping a
   compacted topic at slightly different times may see different snapshots
   because compaction is always in progress. For use cases needing a
   point-in-time snapshot, take it at a fixed offset and ignore newer
   records.

8. **GDPR / "right to be forgotten".** Compaction is the **only** way to
   remove a specific record from Kafka without deleting the whole topic.
   For compliance, use `cleanup.policy=compact` + tombstones + a bounded
   `max.compaction.lag.ms` (e.g. 24 h) + a bounded `delete.retention.ms`.
   Document the max time from tombstone-write to physical erasure; that is
   `max.compaction.lag.ms + delete.retention.ms` worst case.

9. **`retention.bytes` is per partition, not per topic.** Everyone misreads
   this. A 32-partition topic with `retention.bytes=10GB` holds up to
   320 GiB.

10. **Changing `cleanup.policy` on a live topic is allowed but dangerous.**
    Going `delete -> compact` will start compacting segments that may have
    null-key records — they become immortal. Going `compact -> delete` is
    cleaner but you will suddenly start dropping historical values; CDC
    consumers that assumed "bootstrap from 0 = full state" will break.

11. **`__consumer_offsets` is a compacted topic.** Its tuning
    (`offsets.retention.minutes`, default 7 days in recent versions) bounds
    how long an inactive group retains its committed offsets before they
    are tombstoned. Inactive consumers that come back after that window
    restart from `auto.offset.reset`.

12. **KRaft does not change cleaner behavior** — the metadata log itself is
    compacted via a different mechanism (snapshots), but user topics are
    handled by the same `log.cleaner` threads as on ZK-based clusters.

---

## References

- [Apache Kafka — Log Compaction design](https://kafka.apache.org/documentation/#compaction)
- [Confluent: Log Compaction](https://docs.confluent.io/kafka/design/log_compaction.html)
- [KIP-354: max.compaction.lag.ms](https://cwiki.apache.org/confluence/display/KAFKA/KIP-354:+Add+a+Maximum+Log+Compaction+Lag)
- [Aiven: Configure the log cleaner](https://aiven.io/docs/products/kafka/howto/configure-log-cleaner)
- [Conduktor: Kafka Log Compaction Explained](https://www.conduktor.io/glossary/kafka-log-compaction-explained)
- [Ted Naleid: Understanding Kafka Compaction](https://www.naleid.com/2023/07/30/understanding-kafka-compaction.html)
- [Cloudurable: Kafka Log Compaction 2025 Edition](https://cloudurable.com/blog/kafka-architecture-log-compaction-2025/)
- [Redpanda: Kafka log compaction configuration and troubleshooting](https://www.redpanda.com/guides/kafka-performance-kafka-log-compaction)
- [OneUptime: How to Handle Kafka Topic Compaction (2026)](https://oneuptime.com/blog/post/2026-01-24-handle-kafka-topic-compaction/view)
