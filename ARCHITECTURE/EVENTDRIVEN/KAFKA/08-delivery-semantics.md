# Delivery Semantics — Deep Dive

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- Kafka supports three delivery semantics — **at-most-once**, **at-least-once**, **exactly-once** — and the choice is primarily a producer+consumer config combination, not a broker feature.
- **At-least-once is the default** for any honest modern configuration (idempotent producer + manual commit after processing). At-most-once requires deliberately choosing `acks=0` or pre-processing commits — almost never what you want.
- **Exactly-once within Kafka** is achieved by the **idempotent producer (KIP-98)** + **transactions (KIP-98 / KIP-129)** + **EOS v2 (KIP-447)**, with consumers reading at `isolation.level=read_committed`. The boundary is "within Kafka" — reads from DB and sends to HTTP aren't magically transactional.
- The canonical pattern is **read-process-write**: consume → process → produce result + `SendOffsetsToTransaction` + `CommitTransaction`, all atomic. Offsets become part of the transaction; a crash rolls back both the output *and* the consumer offset movement.
- **EOS v2 (KIP-447)** removed the 1:1 requirement between input partitions and transactional producers. One producer can now atomically write outputs derived from many input partitions, making Streams applications massively cheaper.
- External-world exactly-once (Kafka → DB, Kafka → HTTP) requires **idempotent consumers** — a dedupe table or idempotent target operation. Kafka's EOS doesn't cross its boundary.

## Deep dive

### The three semantics

| Semantic | Failure outcome | Typical producer/consumer setup |
|---|---|---|
| **At-most-once** | Possible loss, no duplicates | `acks=0` OR commit offsets *before* processing. Rare in practice |
| **At-least-once** | Duplicates possible, no loss | `acks=all` + idempotent producer + manual commit *after* processing |
| **Exactly-once** | No loss, no duplicates (within Kafka) | Transactions + `read_committed` consumer + SendOffsetsToTransaction |

"Exactly-once" is a property of the **end-to-end pipeline** you build, not of Kafka itself. Kafka provides the primitives; correct usage gives the guarantee.

### Idempotent producer internals (KIP-98)
Enabled by default since 3.0. Mechanism recap (from `05-producers.md`):

- Producer registers with the transaction coordinator (the broker hosting its partition of `__transaction_state`) to get a **Producer ID (PID)**.
- Each `(PID, partition)` gets a **monotonic sequence number**. Every record batch carries `(PID, epoch, seq_base, seq_count)`.
- The **leader replica** maintains a 5-entry LRU of recent batch sequence numbers per partition per PID. Out-of-range sequences are rejected with `OUT_OF_ORDER_SEQUENCE_NUMBER`; duplicates with `DUPLICATE_SEQUENCE_NUMBER` (treated as success — no-op).
- This dedup survives leader failover: the sequence cache is persisted in the replica's log (via `ProducerStateSnapshot` files). A new leader rebuilds it from the log.

Scope: **one producer session, one partition**. Process restart → new PID → no cross-session dedup. That's where transactions come in.

### Transactions (KIP-98 + KIP-129)
A **transaction** is an atomic unit spanning multiple partitions and (optionally) consumer-offset commits.

Components:

- **`transactional.id`**: stable, user-provided string. Identifies a logical transactional producer. **Must survive process restart** — that's how fencing works. Typical values: `orders-processor-pod-0`, `payment-reconciler-vm-a1`.
- **Transaction coordinator**: broker hosting a partition of `__transaction_state` (hashed from `transactional.id`). Owns the transaction lifecycle.
- **Producer epoch**: 16-bit counter incremented every time someone calls `InitTransactions()` for a `transactional.id`. The coordinator aborts any in-flight transaction from the old epoch and refuses future writes from it. This is **zombie fencing**.
- **Control records**: special markers written to partition logs at `CommitTransaction` / `AbortTransaction` time. The consumer at `read_committed` uses these to decide whether to surface records from a given PID.
- **`__transaction_state`**: internal compacted topic (RF 3 by default) holding transaction metadata.

Lifecycle:

1. `InitTransactions()` — blocks until coordinator confirms PID/epoch. Aborts any in-flight transaction from a previous instance of this `transactional.id`.
2. `BeginTransaction()` — in-process marker; no broker RPC.
3. `Produce()` calls — records buffer as usual. First produce to a new partition sends `AddPartitionsToTxn` under the hood so the coordinator knows which partitions this transaction touches.
4. `SendOffsetsToTransaction(offsets, consumerGroupMetadata)` — included in `read-process-write`. Writes offsets to `__consumer_offsets` as part of the transaction.
5. `CommitTransaction()` — coordinator writes a `COMMIT` control record to every touched partition + `__consumer_offsets`. Atomic.
6. `AbortTransaction()` — coordinator writes `ABORT` control records.

Either `Commit` or `Abort` is irrevocable. After either, the producer can `BeginTransaction` again.

### `transaction.timeout.ms`
- Default: **60 000 ms** (60 s). Maximum the broker-side coordinator will wait for commit/abort before force-aborting the transaction.
- Bounded by broker `transaction.max.timeout.ms` (default 900 000 ms = 15 min).
- KIP-447 recommends **10 s** for high-throughput Streams-style workloads — short timeouts mean faster recovery from zombies.
- Must be **larger** than the time the producer needs between `BeginTransaction` and `CommitTransaction` in worst case, otherwise you race the coordinator aborting you.

### EOS v1 vs EOS v2 (KIP-447)
Old (EOS v1, pre-Kafka 2.5):

- **One transactional producer per input partition**. A Streams app consuming from 100 input partitions needed 100 producers, each with its own `transactional.id`. Massive overhead.
- Rebalances were expensive: moving a partition meant moving the associated `transactional.id`.

New (EOS v2, KIP-447, 2.5+, now the default in Kafka 3.x+/4.x):

- **One producer per thread / per Streams StreamThread**, regardless of input partition count.
- `SendOffsetsToTransaction` now carries the **consumer group metadata** (member ID + generation), so the broker can validate that the producer is writing offsets for a consumer that still legitimately owns those partitions. If the consumer got kicked out during the transaction, the broker rejects the offset commit → transaction aborts.
- Streams configuration: `processing.guarantee=exactly_once_v2`. The legacy `exactly_once` (v1) was deprecated in KIP-732 and removed in later versions.

This made Kafka Streams EOS practical: a 100-partition app now runs on one producer per thread, not 100.

### Read-process-write pattern

The canonical EOS loop:

```
consumer.subscribe("input")
producer.initTransactions()

while running:
    records = consumer.poll(timeout)
    producer.beginTransaction()
    for r in records:
        result = transform(r)
        producer.send("output", result)
    producer.sendOffsetsToTransaction(
        records.lastOffsetsByPartition(),
        consumer.groupMetadata()
    )
    producer.commitTransaction()
```

Invariants this preserves:

- If the process crashes between `beginTransaction` and `commitTransaction`, all outputs and offset advances are aborted. On restart, `initTransactions` aborts the in-flight transaction; consumer rereads the same input → same outputs → same offsets.
- The consumer **must** use `enable.auto.commit=false`. Offsets are committed *by the producer* as part of the transaction, not by the consumer.
- Downstream consumers must use `isolation.level=read_committed` — otherwise they'd see aborted records and break the guarantee.

Anti-patterns:
- Processing side effects outside the transaction (DB writes, HTTP calls) — those aren't rolled back on abort. Move them inside an idempotent target or use outbox-pattern.
- Calling `consumer.commit()` alongside `producer.sendOffsetsToTransaction` — double commit; one or the other.
- Spawning async work from inside the transaction that finishes after `commitTransaction` — those writes are **not** in the transaction.

### `isolation.level`
- `read_uncommitted` (default): consumer returns all records including those in open/aborted transactions. Never use for EOS pipelines.
- `read_committed`: consumer only sees records from committed transactions.
  - Introduces the **Last Stable Offset (LSO)**: the smallest offset ≥ which may still be part of an open transaction. `read_committed` reads stall at the LSO when a transaction is open.
  - **Lag metrics must compare to LSO**, not log-end-offset, or you'll see false lag during active transactions.

### Kafka Streams EOS
`processing.guarantee=exactly_once_v2` gives EOS for the entire topology: input topic → intermediate state stores (RocksDB) → output topic. Mechanics:

- Each `StreamThread` runs one transactional producer.
- State store changes are backed by a **changelog topic** that participates in the transaction. Crash during processing → state reverts to pre-transaction.
- Transactions are committed every `commit.interval.ms` (100 ms in EOS mode) or when `max.task.idle.ms` elapses, whichever first.
- Requires brokers with `transaction.state.log.replication.factor >= 3` and `transaction.state.log.min.isr >= 2`.

### Idempotent consumers (external-world EOS)
Kafka's EOS stops at Kafka's boundary. If your sink is a database, HTTP service, or anything else, you need **application-level idempotence**:

- **Dedupe table**: store processed `(topic, partition, offset)` or a business-level key. On each record, check/insert atomically with the side effect. SQL: `INSERT ... ON CONFLICT DO NOTHING` with the offset as a unique key.
- **Idempotent target operation**: upsert by primary key, conditional writes (`If-None-Match`), HTTP `PUT` with stable keys.
- **Outbox pattern**: inside a DB transaction, write business state + an event row; a separate relay process reads the outbox and publishes to Kafka. Combined with `read_committed`, this gives end-to-end exactly-once for DB→Kafka.

Dedupe tables grow without bound — prune by retention window (e.g., older than `log.retention.ms`).

## Config reference

### Producer (for EOS)

| Key | Default | Notes |
|---|---|---|
| `transactional.id` | `null` | Must be set and stable per logical producer |
| `enable.idempotence` | `true` | Forced on when `transactional.id` set |
| `acks` | `all` | Forced when transactional |
| `transaction.timeout.ms` | `60000` | Lower to 10–30s for fast recovery |
| `max.in.flight.requests.per.connection` | `5` | Works with transactions since 1.0 |

### Consumer (for EOS reads)

| Key | Default | Notes |
|---|---|---|
| `isolation.level` | `read_uncommitted` | **Set to `read_committed` for EOS** |
| `enable.auto.commit` | `true` | **Must be `false`** — offsets committed via transaction |

### Broker

| Key | Default | Notes |
|---|---|---|
| `transaction.state.log.replication.factor` | `3` | RF of `__transaction_state`; keep at 3 |
| `transaction.state.log.min.isr` | `2` | ISR floor for the transaction log |
| `transaction.max.timeout.ms` | `900000` (15 min) | Upper bound on client `transaction.timeout.ms` |
| `transactional.id.expiration.ms` | `604800000` (7 days) | How long an idle `transactional.id` is remembered |

### Kafka Streams

| Key | Default | Notes |
|---|---|---|
| `processing.guarantee` | `at_least_once` | Set to `exactly_once_v2` for EOS |
| `commit.interval.ms` | `30000` (at-least-once) / `100` (EOS) | Lower in EOS for latency |

## .NET / C# snippet (Confluent.Kafka) — read-process-write

```csharp
using Confluent.Kafka;

var consumerConfig = new ConsumerConfig
{
    BootstrapServers  = "broker1:9092,broker2:9092,broker3:9092",
    GroupId           = "order-enricher",
    EnableAutoCommit  = false,                           // REQUIRED for EOS
    IsolationLevel    = IsolationLevel.ReadCommitted,    // REQUIRED for EOS reads
    AutoOffsetReset   = AutoOffsetReset.Earliest,
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.CooperativeSticky,
};

var producerConfig = new ProducerConfig
{
    BootstrapServers   = "broker1:9092,broker2:9092,broker3:9092",
    // Stable per logical producer — survives restarts. Fencing depends on it.
    TransactionalId    = $"order-enricher-{Environment.GetEnvironmentVariable("POD_NAME") ?? Environment.MachineName}",
    TransactionTimeoutMs = 30_000,
    EnableIdempotence  = true,                          // implied by transactional.id but be explicit
    Acks               = Acks.All,
    CompressionType    = CompressionType.Zstd,
};

using var consumer = new ConsumerBuilder<string, byte[]>(consumerConfig).Build();
using var producer = new ProducerBuilder<string, byte[]>(producerConfig).Build();

// One-time: bumps epoch, aborts any in-flight transaction owned by a previous incarnation.
producer.InitTransactions(TimeSpan.FromSeconds(30));

consumer.Subscribe("orders");

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

const int BatchSize = 500;
var buffer = new List<ConsumeResult<string, byte[]>>(BatchSize);

try
{
    while (!cts.IsCancellationRequested)
    {
        var cr = consumer.Consume(TimeSpan.FromMilliseconds(200));
        if (cr is not null && !cr.IsPartitionEOF) buffer.Add(cr);

        // Commit when we have a batch or the poll timed out with something buffered.
        if (buffer.Count >= BatchSize || (cr is null && buffer.Count > 0))
        {
            FlushTransaction(consumer, producer, buffer);
            buffer.Clear();
        }
    }
}
finally
{
    if (buffer.Count > 0) FlushTransaction(consumer, producer, buffer);
    consumer.Close();
}

static void FlushTransaction(
    IConsumer<string, byte[]> consumer,
    IProducer<string, byte[]> producer,
    List<ConsumeResult<string, byte[]>> batch)
{
    producer.BeginTransaction();
    try
    {
        foreach (var r in batch)
        {
            var enriched = Enrich(r.Message.Value);
            producer.Produce("orders-enriched", new Message<string, byte[]>
            {
                Key   = r.Message.Key,
                Value = enriched
            });
        }

        // Advance consumer offsets for every partition in the batch to (last_offset + 1).
        var offsets = batch
            .GroupBy(r => r.TopicPartition)
            .Select(g => new TopicPartitionOffset(
                g.Key,
                new Offset(g.Max(r => r.Offset.Value) + 1)))
            .ToList();

        // KIP-447: pass consumer group metadata for validation.
        producer.SendOffsetsToTransaction(
            offsets,
            consumer.ConsumerGroupMetadata,
            TimeSpan.FromSeconds(10));

        producer.CommitTransaction(TimeSpan.FromSeconds(30));
    }
    catch (KafkaException ex) when (ex.Error.IsFatal)
    {
        // Fatal: producer must be rebuilt. Typically InvalidProducerEpoch = zombie fencing.
        throw;
    }
    catch (KafkaException)
    {
        // Transient: abort and retry on the next loop.
        producer.AbortTransaction(TimeSpan.FromSeconds(10));
        throw;
    }
}

static byte[] Enrich(byte[] input) => input; // placeholder
```

Notes:

- `TransactionalId` embeds the pod/host name. This is essential: a new incarnation of the same pod must reuse the same id so fencing works. Two pods with the same id → one gets fenced.
- `InitTransactions` **blocks** until the coordinator acknowledges. Typical: 100 ms–2 s. Longer means coordinator is recovering.
- `SendOffsetsToTransaction` takes **last_offset + 1** (the offset to *resume from*), consistent with normal `Commit` semantics.
- `ConsumerGroupMetadata` is the KIP-447 piece — the broker rejects the offset commit if the consumer has lost its assignment.
- On `AbortTransaction`, the consumer **does not** rewind automatically. On the next loop iteration, `Consume()` will continue from where it left off in memory — but no offsets were committed, so on any restart/rebalance, processing starts from the last committed offset, which is correct.
- On fatal `InvalidProducerEpoch` (zombie fencing detected), you must dispose the producer and build a new one. This happens if a new pod with your `transactional.id` came up and called `InitTransactions` while you were alive.

## Senior-level gotchas
- **`transactional.id` must be stable across restarts, unique across live instances.** Random UUIDs at process start = no fencing = duplicate writes during restarts. Hostname/pod name = correct.
- **`transaction.timeout.ms` too short** (< processing time) → coordinator force-aborts you mid-batch. You'll see `ProducerFencedException` or `InvalidProducerEpoch`. Measure worst-case batch time first.
- **Skipping `isolation.level=read_committed`** downstream silently breaks the guarantee — aborted records are visible.
- **Auto-commit on a transactional consumer** double-commits offsets (once by the consumer, once by the transaction) — races and inconsistency. Must be off.
- **EOS does not extend to DB/HTTP sinks.** Anything outside Kafka needs idempotent operations or a dedupe table.
- **Producer per thread, not per request.** `IProducer` is thread-safe but *one* transaction at a time. Spawning a producer per request is a performance disaster (PID registration round-trip every time).
- **`SendOffsetsToTransaction` with stale `ConsumerGroupMetadata`** (after a rebalance during processing) will fail. Catch `KafkaException`, abort, and restart the loop.
- **`__transaction_state` under-replicated** is a cluster-level latent risk. Monitor `UnderReplicatedPartitions` for this topic specifically.
- **EOS v1 is removed** in Kafka 4.x. If you're upgrading from older Streams apps, switch to `exactly_once_v2` first.
- **Transactional id expiration** (`transactional.id.expiration.ms` = 7 days) wipes coordinator state for idle IDs. Rarely a problem in prod but surprising in dev/staging: a dormant service returning after a week re-registers without fencing.
- **Control records count toward offsets.** A consumer's visible offset doesn't equal broker log-end-offset even without gaps — the difference is control markers. Lag math must account for this.
- **Read-process-write latency is dominated by `commit.interval.ms` + `transaction.timeout.ms`**, not by produce throughput. Tuning EOS means tuning these.

## References
- [Confluent blog: Exactly-Once Semantics is Possible — Here's How Kafka Does It](https://www.confluent.io/blog/simplified-robust-exactly-one-semantics-in-kafka-2-5/)
- [KIP-447: Producer scalability for exactly once semantics](https://cwiki.apache.org/confluence/display/KAFKA/KIP-447%3A+Producer+scalability+for+exactly+once+semantics)
- [KIP-98: Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)
- [KIP-732: Deprecate eos-alpha and replace eos-beta with eos-v2](https://cwiki.apache.org/confluence/x/zJONCg)
- [Strimzi: Exactly-once semantics with Kafka transactions](https://strimzi.io/blog/2023/05/03/kafka-transactions/)
- [Confluent developer: Apache Kafka for .NET — Transactional Commits](https://developer.confluent.io/courses/apache-kafka-for-dotnet/transactional-commits-hands-on/)
- [Confluent.Kafka .NET `IProducer` API reference](https://docs.confluent.io/platform/current/clients/confluent-kafka-dotnet/_site/api/Confluent.Kafka.IProducer-2.html)
