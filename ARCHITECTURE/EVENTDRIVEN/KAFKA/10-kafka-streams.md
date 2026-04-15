# Kafka Streams (and .NET Alternatives)

> Version baseline: Apache Kafka 4.x (KRaft mode), Kafka Streams 4.x (JVM).
> .NET alternatives: Streamiz 1.6+, ksqlDB current. Last researched: 2026-04.

## TL;DR

**Kafka Streams** is Apache's JVM library for stateful stream processing on
top of Kafka. It turns a Kafka consumer+producer into a topology of
streaming operators (filter, map, aggregate, join, window) with local state
stores (RocksDB), automatic fault tolerance (changelog topics), and
exactly-once semantics — all without a separate cluster. It is just a Java
library that scales by running more instances of your app; Kafka itself is
the "cluster".

> **Kafka Streams is JVM-only.** There is no first-party .NET port. If you
> live in .NET you have three realistic choices:
>
> 1. **Streamiz** (<https://github.com/LGouellec/streamiz>) — community .NET
>    library built by a Confluent engineer that re-implements the Kafka
>    Streams DSL in C# on top of `Confluent.Kafka`. Same mental model,
>    RocksDB state, changelog topics, windowed joins. Not 100% feature
>    parity with JVM Streams, but covers the common cases.
> 2. **ksqlDB** — Confluent's SQL-over-Kafka engine. Runs on the JVM
>    server-side but is queryable from any language via REST/WebSocket. A
>    C# client (`ksqlDB.RestApi.Client`) exists. Good for "I want streaming
>    joins and aggregations but I do not want to operate a stream-processing
>    app".
> 3. **Plain `Confluent.Kafka` consumer + producer + your own state** —
>    explicit topology in C#. Use RocksDB via `RocksDbSharp` or an embedded
>    LiteDB/SQLite for state, and manually write a compacted changelog topic
>    for recoverability. Max control, maximum foot-gun.
>
> If you need Streams-grade fault tolerance on .NET today, Streamiz is the
> closest drop-in. For complex event-time windowed joins in production, many
> .NET shops still route through a JVM Streams app or ksqlDB.

Core Streams abstractions:

- **KStream** — unbounded stream of immutable facts (event log). Every
  record is an independent event. Appending.
- **KTable** — changelog-backed table: latest value per key. Updates
  overwrite. Backed by a state store + compacted changelog topic.
- **GlobalKTable** — fully replicated KTable — every Streams instance holds
  the entire dataset. Used for small, slow-changing reference data where
  you want **non-co-partitioned** joins (no repartitioning required).

---

## Deep dive

### 1. KStream vs KTable vs GlobalKTable

| Aspect | KStream | KTable | GlobalKTable |
|---|---|---|---|
| Semantics | Event stream (append) | Changelog (upsert) | Changelog, fully replicated |
| Partitioning | Sharded | Sharded (co-partitioned with input) | **Full copy on every instance** |
| Join key | Must match partition key (or be repartitioned) | Must be co-partitioned | Any FK — no co-partitioning required |
| Size | Unbounded | Large, sharded | Must fit on each instance |
| Bootstrap | No | Lazy, up to latest offset | Fully loaded **before** processing starts |
| Typical use | Clicks, orders, events | Running aggregate, per-key state | Country codes, product catalog, feature flags |

Canonical example:

```java
KStream<String, Order>        orders    = builder.stream("orders");
KTable<String, Customer>      customers = builder.table("customers");
GlobalKTable<String, Country> countries = builder.globalTable("countries");

orders.join(customers, (order, cust) -> enrich(order, cust))       // co-partitioned join
      .join(countries,
            (key, enriched) -> enriched.countryCode,               // FK extractor
            (enriched, country) -> enriched.withCountry(country))  // global join
      .to("orders-enriched");
```

### 2. State stores

Streams tasks own local **state stores**. Types:

- **RocksDB (persistent)** — default. Embedded key-value DB written to
  local disk. Survives process restart without replaying the changelog.
- **In-memory** — `Stores.inMemoryKeyValueStore(...)`. Faster, but on
  restart must fully replay the changelog — slow for large state.
- **Windowed** — key + window start -> value. Internally RocksDB with a
  composite key. Used by `.windowedBy(...)` aggregations.
- **Session** — stores session windows (key + session-start/end).

State stores are **local to a task**; Streams partitions the input topic and
each partition's data lives in exactly one task's store. Re-balancing moves
tasks (and their stores) between instances.

### 3. Changelog topics

Every state store has a **changelog topic** named
`<application.id>-<store-name>-changelog`. Properties:

- `cleanup.policy=compact` (so it keeps latest per key forever).
- Auto-created by Streams with sensible defaults; can be pre-created with
  custom retention.
- On task failover, the new owner replays the changelog into a fresh
  RocksDB. This is why compaction matters — without it, recovery would
  take hours.
- For windowed stores, Streams sets `cleanup.policy=compact,delete` with
  `retention.ms = window size + grace period` so old windows are purged.

### 4. Repartition topics

Any operation that changes the key (`selectKey`, `map` that returns a new
key, `groupBy`) forces Streams to **repartition**: records are written to
an internal `<app-id>-<node>-repartition` topic, then re-consumed. This is
expensive: 2x network, 2x disk. Senior rule: **plan your partition key at
ingest time** so downstream operators can use `groupByKey` (no repartition)
instead of `groupBy` (repartition).

Repartition topics are NOT compacted (they are delete-mode with short
retention) — they hold events, not state.

### 5. Windowing

Windows group records by key + time. Event-time is the default; wall-clock
(processing-time) is available but usually wrong for real data.

| Window type | Behavior | Example |
|---|---|---|
| **Tumbling** | Fixed, non-overlapping | 5-min buckets: `[0,5) [5,10) [10,15)` |
| **Hopping** | Fixed size, sliding by a step | size=5m, advance=1m -> overlapping buckets |
| **Session** | Dynamic; closes after `inactivityGap` of silence | User browsing session |
| **Sliding** (KIP-450) | Window of size N moves with every record — emits for record pairs within N | Fraud detection ("3 logins in 10 s") |

**Grace period** (`.grace(Duration)`) — how long after window-end to still
accept out-of-order records. After grace, the window is **closed** and
late records are routed to `DROPPED` metric / a dead-letter stream. Senior
tradeoff: larger grace = more state, more accuracy; smaller grace = less
memory, more drops.

**Retention** — stores keep window results for a bounded time (default
1 day past window-end). Interactive queries can only read within retention.

### 6. Joins

Four shapes:

1. **Stream-Stream** — windowed join over a time window. Both sides must
   be co-partitioned. Emits once per matching pair within the window.
   ```java
   clicks.join(impressions,
               (click, imp) -> new Attribution(click, imp),
               JoinWindows.of(Duration.ofMinutes(5)));
   ```

2. **Stream-Table** — non-windowed. Stream event is enriched with current
   table value. Left side drives; right side is a point-in-time lookup.
   Must be co-partitioned.

3. **Table-Table** — non-windowed. Maintains a joined KTable: every update
   on either side re-emits. Co-partitioned.

4. **FK Table-Table join** (KIP-213) — left KTable's value contains a FK
   into the right KTable. Streams handles the repartition machinery for
   you (under the hood: a "subscription" topic + reverse lookup). Very
   powerful but expensive — 4 internal topics per FK join.

5. **Stream/Table -> GlobalKTable** — non-windowed, no co-partition required
   because the entire GlobalKTable is local.

### 7. Interactive Queries (IQ / IQv2)

Streams apps can expose their local state stores **read-only** over an RPC
of your choice. Kafka Streams itself only gives you `store(..)` and
`streamsMetadataForKey(..)`; you implement the HTTP/gRPC server. IQv2
(KIP-796) adds a typed query abstraction so custom store types can plug in.

Pattern: "where does key K live?" -> `streams.metadataForKey(..)` returns
which instance owns it. If it is the local instance, read directly; if
remote, forward the HTTP call. This builds a serverless "distributed
database" out of your Streams app — no external DB needed.

### 8. Processor API

Lower-level than the DSL. You write a `Processor<K, V, K', V'>` that
receives records one by one, manages its own stores, and forwards output.
Use cases: custom punctuation (periodic timers), complex state transitions,
operations not expressible in DSL (e.g., multi-store lookups).

```java
public class FraudProcessor implements Processor<String, Tx, String, Alert> {
    private KeyValueStore<String, Long> counts;
    public void init(ProcessorContext<String, Alert> ctx) {
        counts = ctx.getStateStore("fraud-counts");
        ctx.schedule(Duration.ofMinutes(1), PunctuationType.WALL_CLOCK_TIME,
                     ts -> counts.all().forEachRemaining(...));
    }
    public void process(Record<String, Tx> r) { ... }
}
```

### 9. EOS in Streams

`processing.guarantee=exactly_once_v2` (the current, KIP-447 mode; replaces
the old `exactly_once` which used one producer per task).

Under the hood:

- Each Streams thread owns a transactional producer.
- For every input batch, Streams opens a transaction and inside it:
  writes to output topics, writes to changelog topics, and **commits input
  offsets** (via `sendOffsetsToTransaction`) — all atomically.
- On failure, the transaction aborts; neither outputs nor offset commits
  become visible. The thread re-reads the batch.
- Consumers downstream must set `isolation.level=read_committed`.

EOS requires:

- `min.insync.replicas >= 2` (practically 2) and `replication.factor >= 3`
  for the internal topics.
- `transaction.state.log.replication.factor >= 3`.
- Lower throughput (~10–30% overhead vs `at_least_once`).
- **Does not** protect against non-Kafka side effects (e.g., HTTP calls,
  DB writes outside a Kafka transaction).

---

## Config reference

### Core Streams config (JVM)

| Config | Default | Meaning |
|---|---|---|
| `application.id` | _required_ | Becomes consumer group id, changelog/repartition topic prefix |
| `bootstrap.servers` | _required_ | Kafka cluster |
| `processing.guarantee` | `at_least_once` | or `exactly_once_v2` |
| `num.stream.threads` | `1` | Threads per instance; each owns a set of tasks |
| `replication.factor` | `-1` (broker default) | For internal topics; set to 3 in prod |
| `state.dir` | `/tmp/kafka-streams` | RocksDB on-disk location — **change this**, `/tmp` is wiped |
| `commit.interval.ms` | `30000` (or `100` for EOS) | How often offsets + stores are committed |
| `cache.max.bytes.buffering` | `10485760` (10 MiB) | DSL record cache; `0` disables and forces every update to emit |
| `default.deserialization.exception.handler` | `LogAndFailExceptionHandler` | Set to `LogAndContinue...` or a DLQ handler for poison pills |
| `default.production.exception.handler` | `DefaultProductionExceptionHandler` | How to react to producer failures |
| `rocksdb.config.setter` | `null` | Hook for tuning RocksDB (block cache, write buffers) |
| `topology.optimization` | `all` | DSL optimizations (merge repartitions, reuse source topics) |

### Streamiz (.NET) equivalent

| JVM name | Streamiz name |
|---|---|
| `application.id` | `ApplicationId` |
| `bootstrap.servers` | `BootstrapServers` |
| `processing.guarantee` | `Guarantee` (`AT_LEAST_ONCE` / `EXACTLY_ONCE`) |
| `state.dir` | `StateDir` |
| `num.stream.threads` | `NumStreamThreads` |
| `commit.interval.ms` | `CommitIntervalMs` |

---

## .NET / C# snippet

### Option A — Streamiz (closest to Kafka Streams)

```csharp
// dotnet add package Streamiz.Kafka.Net
using Streamiz.Kafka.Net;
using Streamiz.Kafka.Net.SerDes;
using Streamiz.Kafka.Net.Stream;
using Streamiz.Kafka.Net.Table;

var config = new StreamConfig<StringSerDes, StringSerDes>
{
    ApplicationId    = "order-enricher",
    BootstrapServers = "localhost:9092",
    StateDir         = "./state",                 // RocksDB lives here
    NumStreamThreads = 2,
    Guarantee        = ProcessingGuarantee.EXACTLY_ONCE,
    CommitIntervalMs = 1000,
};

var builder = new StreamBuilder();

// KStream of orders
IKStream<string, string> orders = builder.Stream<string, string>("orders");

// KTable of customers (compacted topic)
IKTable<string, string> customers = builder.Table<string, string>("customers");

// GlobalKTable of countries — fully replicated on every instance
IGlobalKTable<string, string> countries =
    builder.GlobalTable<string, string>("countries");

// Stream-Table join (co-partitioned)
var enriched = orders.Join(customers,
    (orderJson, customerJson) => $"{{\"order\":{orderJson},\"cust\":{customerJson}}}");

// Windowed aggregation: orders per customer in 5-minute tumbling windows
orders
    .GroupByKey()
    .WindowedBy(TumblingWindowOptions.Of(TimeSpan.FromMinutes(5))
                                     .WithGrace(TimeSpan.FromMinutes(1)))
    .Count(InMemory.As<string, long>("order-counts-5m"))
    .ToStream()
    .Map((windowedKey, count) =>
        KeyValuePair.Create($"{windowedKey.Key}@{windowedKey.Window.StartMs}",
                            count.ToString()))
    .To("order-counts-5m-out");

enriched.To("orders-enriched");

var topology = builder.Build();
var stream   = new KafkaStream(topology, config);

Console.CancelKeyPress += (_, e) => { e.Cancel = true; stream.Dispose(); };
await stream.StartAsync();
```

### Option B — ksqlDB from C# (SQL over Kafka)

```csharp
// dotnet add package ksqlDB.RestApi.Client

using ksqlDB.RestApi.Client.KSql.Query.Context;
using ksqlDB.RestApi.Client.KSql.Linq;

var ksqlDbUrl = "http://localhost:8088";
await using var ctx = new KSqlDBContext(ksqlDbUrl);

// Server-side: CREATE STREAM orders ... and CREATE TABLE countries ...
// Then query as a push query from C#:
using var sub = ctx.CreateQueryStream<Order>("ORDERS_ENRICHED")
    .Where(o => o.Total > 100)
    .Subscribe(
        onNext: o  => Console.WriteLine($"Big order {o.Id} {o.Total}"),
        onError: e => Console.Error.WriteLine(e));
```

The aggregation/join/windowing runs on the ksqlDB JVM server. Your .NET app
just consumes the result stream. Operationally you still run a ksqlDB
cluster, but your app is pure C#.

### Option C — plain Confluent.Kafka consumer + RocksDB (DIY)

```csharp
// dotnet add package Confluent.Kafka
// dotnet add package RocksDbSharp

using Confluent.Kafka;
using RocksDbNet = RocksDbSharp.RocksDb;

var consumerCfg = new ConsumerConfig
{
    BootstrapServers   = "localhost:9092",
    GroupId            = "order-counter",
    EnableAutoCommit   = false,
    IsolationLevel     = IsolationLevel.ReadCommitted,
    AutoOffsetReset    = AutoOffsetReset.Earliest,
};

var producerCfg = new ProducerConfig
{
    BootstrapServers        = "localhost:9092",
    EnableIdempotence       = true,
    TransactionalId         = "order-counter-tx-1",   // per instance
};

using var db       = RocksDbNet.Open(new RocksDbSharp.DbOptions().SetCreateIfMissing(),
                                     "./state/order-counts");
using var consumer = new ConsumerBuilder<string, string>(consumerCfg).Build();
using var producer = new ProducerBuilder<string, long>(producerCfg).Build();

producer.InitTransactions(TimeSpan.FromSeconds(10));
consumer.Subscribe("orders");

while (!ct.IsCancellationRequested)
{
    var cr = consumer.Consume(TimeSpan.FromSeconds(1));
    if (cr is null) continue;

    producer.BeginTransaction();
    try
    {
        // Update local RocksDB state
        var key   = cr.Message.Key;
        var bytes = db.Get(System.Text.Encoding.UTF8.GetBytes(key));
        long count = bytes is null ? 0 : BitConverter.ToInt64(bytes) ;
        count++;
        db.Put(System.Text.Encoding.UTF8.GetBytes(key), BitConverter.GetBytes(count));

        // Write to changelog (compacted topic) for fault tolerance
        producer.Produce("order-counts-changelog",
            new Message<string, long> { Key = key, Value = count });

        // Emit to output
        producer.Produce("order-counts",
            new Message<string, long> { Key = key, Value = count });

        // Commit input offsets inside the transaction -> EOS
        producer.SendOffsetsToTransaction(
            new[] { new TopicPartitionOffset(cr.TopicPartition, cr.Offset + 1) },
            consumer.ConsumerGroupMetadata,
            TimeSpan.FromSeconds(10));

        producer.CommitTransaction();
    }
    catch
    {
        producer.AbortTransaction();
        throw;
    }
}
```

You are now re-implementing ~30% of Kafka Streams by hand: failover, state
rebuild from the changelog, repartitioning on key change, windowing,
punctuation, rebalance listeners, standby replicas. This is viable for
**simple** stateful pipelines; for anything with joins or windows, prefer
Streamiz or ksqlDB.

---

## Senior-level gotchas

1. **Streams is JVM-only.** There is no Anthropic-grade official .NET port.
   If your team is pure .NET, honestly evaluate Streamiz / ksqlDB before
   rolling your own — people underestimate how much Streams does for them.

2. **Streamiz is not 100% feature-complete.** At time of writing it lacks
   FK joins (KIP-213), full IQv2, and some windowed-store nuances. Check
   the current README before committing to it for a specific feature.

3. **Co-partitioning is mandatory for non-global joins.** Both sides must
   share partition count AND partitioner. Different partition count = join
   silently misses records. Streams will throw `TopologyException` at
   startup if it detects the mismatch — trust the error.

4. **GlobalKTable is fully replicated — size it.** It must fit in memory
   (or RocksDB) on every instance. A 50 GiB global table on 20 instances
   = 1 TiB across the fleet. Do not use it for facts; use it for slowly
   changing reference data only.

5. **Repartition topics are hidden cost.** A single `selectKey` followed by
   `groupBy` writes every record back to Kafka. On a 1 M msg/s pipeline
   that's 1 M msg/s of internal traffic. Use `groupByKey` whenever the
   partition key is already correct.

6. **`state.dir` defaults to `/tmp/kafka-streams`.** On Linux, `/tmp` is
   often wiped on reboot and sometimes tmpfs-backed (RAM). Always override
   to a persistent volume or you will replay the entire changelog on every
   restart.

7. **State rebuild time dominates cold-start latency.** For a 100 GiB
   RocksDB store, rebuilding from a compacted changelog can take 30+
   minutes. Mitigations: **standby replicas** (`num.standby.replicas`) so
   another instance is already warm, and **persistent volumes** (EBS,
   local NVMe with snapshot) so the store survives pod restart.

8. **Cache vs emit behavior.** `cache.max.bytes.buffering > 0` means
   intermediate updates to the same key are coalesced and not emitted.
   Looks like missing records if you compare input and output one-to-one.
   Set `cache.max.bytes.buffering=0` for deterministic downstream testing,
   then re-enable for prod throughput.

9. **Windows have retention; interactive queries respect it.** If you set
   a 5-minute window with default 1-day retention, you cannot query windows
   older than 1 day via IQ — they're purged. Configure
   `Materialized.withRetention(...)` explicitly for long-range queries.

10. **EOS v1 (deprecated) vs v2.** Old code may still set
    `processing.guarantee=exactly_once` — on Kafka 3.0+ brokers this is
    deprecated; use `exactly_once_v2`. v2 shares one producer per thread
    instead of one per task, dramatically lowering broker load for high-
    task-count apps.

11. **EOS does not cover external systems.** If your processor calls an
    HTTP API or writes to Postgres, that side effect is **not** in the
    Kafka transaction. Use the outbox pattern (write intent to a Kafka
    topic, then a separate sink connector applies it) to get atomicity.

12. **Poison pills kill the app by default.** A single deserialization
    error with `LogAndFailExceptionHandler` stops the stream thread.
    Switch to a custom handler that routes bad records to a DLQ topic, or
    at minimum `LogAndContinueExceptionHandler`.

13. **ksqlDB is not a drop-in for Streams.** It is strictly less expressive
    (no Processor API, limited windowing joins), runs as its own cluster
    you must operate, and introduces REST latency if queried live.

14. **Changelog topic retention matters.** By default Streams creates them
    with `cleanup.policy=compact` (forever). For bounded-state use cases,
    switch to `compact,delete` with a sensible `retention.ms` or you will
    never reclaim disk for dead keys.

---

## References

- [Apache Kafka — Streams Core Concepts (4.x)](https://kafka.apache.org/42/streams/core-concepts/)
- [Apache Kafka — Streams DSL](https://kafka.apache.org/documentation/streams/developer-guide/dsl-api.html)
- [Confluent — Streams Concepts](https://docs.confluent.io/platform/current/streams/concepts.html)
- [Confluent — Streams Architecture](https://docs.confluent.io/platform/current/streams/architecture.html)
- [Confluent — What is a KTable](https://developer.confluent.io/courses/kafka-streams/ktable/)
- [KIP-447 — EOS v2](https://cwiki.apache.org/confluence/display/KAFKA/KIP-447%3A+Producer+scalability+for+exactly+once+semantics)
- [KIP-213 — FK table-table joins](https://cwiki.apache.org/confluence/display/KAFKA/KIP-213+Support+non-key+joining+in+KTable)
- [KIP-450 — Sliding windows](https://cwiki.apache.org/confluence/display/KAFKA/KIP-450%3A+Sliding+Window+Aggregations+in+the+DSL)
- [Streamiz on GitHub](https://github.com/LGouellec/streamiz)
- [Streamiz documentation](https://lgouellec.github.io/streamiz/)
- [Streamiz samples](https://github.com/LGouellec/streamiz-samples)
- [Event streaming in .NET with Kafka (Streamiz intro)](https://dev.to/lgouellec/event-streaming-in-net-with-kafka-2c1)
- [ksqlDB.RestApi.Client on NuGet](https://www.nuget.org/packages/ksqlDB.RestApi.Client)
