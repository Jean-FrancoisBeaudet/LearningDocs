# Replication, ISR, Leader Election, and Durability

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- Every partition has **one leader** and `replication.factor - 1` followers. All produce/fetch go to the leader; followers replicate by issuing `Fetch` requests — they are specialized consumers of the leader's log.
- The **In-Sync Replica (ISR)** set is the subset of replicas caught up with the leader within `replica.lag.time.max.ms` (default 30s). Only ISR members are eligible to become leader under clean election.
- A write is **committed** when all current ISR members have it. The **High Watermark (HW)** is the highest committed offset; consumers can only read ≤ HW. **Log End Offset (LEO)** is the highest offset the leader has locally.
- Durability contract = `acks=all` + `replication.factor ≥ 3` + `min.insync.replicas ≥ 2` + `unclean.leader.election.enable=false` + `enable.idempotence=true`. Drop any one of these and you have a lie.
- Leader election in KRaft is fast and clean: the active controller picks the first replica in the preferred-replica list that is in ISR and unfenced. No ZK watches, no full metadata reload.
- **KIP-966 (Eligible Leader Replicas)** is the 2024–2026 refinement: closes the "Last Replica Standing" data-loss gap by tracking replicas that had HW-level data even after they fell out of ISR, enabling safe recovery when the whole ISR crashes.

## Deep dive

### Replicas, leaders, and followers

`replication.factor=3` means every partition has 3 copies distributed across 3 different brokers. One copy is the **leader**; the other two are **followers**. The assignment is decided at topic creation time (or reassignment) and stored in `__cluster_metadata`.

All client IO goes to the leader:
- Produce appends to the leader's log.
- Fetch (for consumers) reads from the leader's log.
- Followers **fetch** from the leader using the same `Fetch` RPC consumers use, but with replica-side bookkeeping.

A follower continuously issues `Fetch(replicaId=<self>, maxWaitMs, minBytes)` to its leader for each partition it replicates. On each successful fetch it:
1. Appends the returned records to its local log, advancing its **LEO** (Log End Offset).
2. Tells the leader (via the next Fetch) the offset it has reached.

The leader, on receiving each follower's fetch position, updates its own view of the follower's LEO and the min-LEO across ISR. That min becomes the partition's **High Watermark**.

### ISR: who counts as "in sync"

A follower is in ISR if **both**:
- Its LEO is ≥ the leader's LEO at some point within the last `replica.lag.time.max.ms` window, **or**
- It has fetched within that window and caught up.

Legacy Kafka also had `replica.lag.max.messages` (byte-based lag). That was removed because it's impossible to tune across topics with different throughput — a single global threshold was always wrong for somebody. Time-based lag (KIP-16) is what's used today.

Leader responsibilities around ISR:
- **Shrink ISR** when a follower hasn't caught up within `replica.lag.time.max.ms`. Persists the new ISR to `__cluster_metadata` (via the controller).
- **Expand ISR** when a lagging follower catches up again. Also controller-persisted.

ISR membership is the source of truth for leader election eligibility. That's why it's persisted in metadata, not just in broker memory.

### High Watermark vs Log End Offset

This pair is the conceptual core of Kafka replication. Get it right and everything else makes sense.

```
Leader log:   [0][1][2][3][4][5][6][7][8]
                                 ↑     ↑
                                HW    LEO
                                 (6)  (9)

Follower F1:  [0][1][2][3][4][5][6][7]       LEO=8, caught up-ish
Follower F2:  [0][1][2][3][4][5][6]          LEO=7, lagging

ISR = {leader, F1, F2}
min LEO across ISR = 7  →  HW advances to 7 after next fetch round
```

- **LEO (Log End Offset)**: next offset to be written = current log size. Each replica has its own LEO.
- **HW (High Watermark)**: the partition's committed offset, defined as `min(LEO of all ISR members)`. Monotonically non-decreasing.

Consumer visibility rule: **consumers can only read offsets < HW**. Records past HW exist on the leader's disk but are not yet "committed" — they might be lost if the leader crashes before followers replicate them.

For `acks=all` producers, the producer's ACK is sent only when **HW ≥ the produced offset**. This is what `acks=all` actually means: "acknowledge when this record is committed across the full current ISR", not "when every configured replica has it".

### The durability chain: what `acks=all` actually guarantees

`acks=all` guarantees durability **only if `min.insync.replicas` is set appropriately and ISR is healthy**. The interaction:

- **`acks=all`** + **`min.insync.replicas=2`** + **RF=3** → the leader will refuse produces with `NotEnoughReplicasException` if ISR drops below 2. A committed record exists on at least 2 brokers. Losing any one broker loses zero data.
- **`acks=all`** + **`min.insync.replicas=1`** + **RF=3** → if ISR shrinks to just the leader (both followers lag out), produces continue and records are "committed" to a single broker. If that broker's disk dies, data is gone. `acks=all` became `acks=1` silently.
- **`acks=1`** → leader acks on local append regardless of ISR. Any leader crash before replication = data loss window of in-flight batches.
- **`acks=0`** → fire-and-forget. Don't use this for anything you care about.

**The combination you want in production**:
```
broker:  replica.lag.time.max.ms = 30000
         unclean.leader.election.enable = false
topic:   replication.factor = 3
         min.insync.replicas = 2
producer: acks = all
          enable.idempotence = true
          retries = Integer.MAX_VALUE (default when idempotence on)
          max.in.flight.requests.per.connection <= 5
```

### Unclean leader election: the durability/availability lever

`unclean.leader.election.enable` (broker-level, overridable at topic level):

- **`false` (recommended, and the default for new topics since 2.4)**: if all ISR members are down, the partition goes **offline**. Producers get `NotLeaderForPartitionException`; consumers stall. Data is preserved at the cost of availability — you wait for an ISR broker to come back.
- **`true`**: any out-of-ISR replica can be elected leader. Its LEO might be behind the last committed HW, so **any records that were committed but not yet replicated to this replica are lost**. Availability is preserved at the cost of durability.

This is a genuine business decision, not a technical one. For financial data, event sourcing, or anything where records are legally-retained: `false`. For metrics, logs, or telemetry where a 5-minute data gap is acceptable to keep ingesting: maybe `true`.

### KRaft leader election, concretely

When a partition needs a new leader (broker unfenced, failed, preferred-replica rebalance), the **active controller**:

1. Reads the partition's current replica assignment, ISR, and leader epoch from its in-memory metadata state.
2. Walks the **preferred replica list** (the topic's assignment order) and picks the first replica that is:
   - In the current ISR,
   - On a broker that is currently unfenced and alive (i.e. heartbeating).
3. Increments the **leader epoch** (monotonic per-partition counter).
4. Writes a `PartitionChangeRecord` to `__cluster_metadata` with the new leader and epoch.
5. The record propagates to all brokers via the metadata log fetch. The old leader (if still alive) stops accepting writes; the new leader takes over.

If no replica satisfies (2) and `unclean.leader.election.enable=false`, the partition goes offline. If it's true, step 2 relaxes to "any alive replica" and the highest-LEO option is picked.

Compared to ZK-mode:
- ZK-mode: controller watched ZK znodes, got notifications, then broadcast `LeaderAndIsrRequest` to brokers. Election time was dominated by ZK watch latency and per-broker RPCs — seconds to minutes on large clusters.
- KRaft: the metadata log is already replicated to every broker. Leader change is one record append. Sub-second on any cluster size.

### Leader epoch and the end of "divergence"

The **leader epoch** is a monotonically increasing integer per partition, incremented on every leadership change. Every record batch carries the epoch of the leader that wrote it, and every broker keeps a `leader-epoch-checkpoint` file mapping epoch → first offset at that epoch.

This fixes a subtle old bug: if a follower truncated its log based solely on HW after a leader change, it could truncate correct data and keep wrong data. KIP-101 replaced HW-based truncation with epoch-based truncation. On becoming a follower of a new leader, the replica issues an `OffsetForLeaderEpoch` request, the leader answers "at epoch E, the log diverged at offset X", and the follower truncates to X.

You will almost never touch leader epoch directly, but if you see `kafka.server:type=ReplicaManager,name=IsrShrinksPerSec` combined with unexpected data loss reports, leader-epoch confusion is the usual culprit — usually traceable to bad manual intervention.

### KIP-966: Eligible Leader Replicas (ELR)

A long-standing durability gap: imagine RF=3, `min.insync.replicas=2`, all three replicas die in succession. When they come back, which one becomes leader? Old algorithm: "any alive replica in ISR at shutdown time". But ISR can shrink to 1 before a broker fails, and the last-standing ISR member might crash before writes land safely. Detecting "who has all the HW-committed data" was impossible.

**KIP-966** introduces **Eligible Leader Replicas (ELR)**: a controller-tracked set of replicas that, at the moment they fell out of ISR, had data up to at least the then-current HW. Rules:

- When ISR shrinks, the removed replica enters ELR if it was caught up to HW.
- When ISR is empty, the controller elects from ELR (not from arbitrary alive replicas) — guaranteeing the new leader has all previously-committed data.
- ELR members promoted to leader rejoin ISR; others re-enter ISR via the normal catch-up path.
- If both ISR and ELR are empty, the operator's choice (config-driven) dictates whether to stall for durability or elect anyway with possible loss.

Practical impact in 4.x: Kafka is now genuinely correct under cascaded failure without operator guesswork. You should still run with RF=3 and `min.insync.replicas=2`, but the "Last Replica Standing" horror story is closed.

### Failure modes and what actually happens

**Follower slow disk** → falls out of ISR after 30s → ISR=2 (assuming RF=3), which still satisfies `min.insync.replicas=2` → produces continue, HW stays correct → when follower catches up, rejoins ISR.

**Leader broker crashes** → controller detects via missed heartbeat (`broker.session.timeout.ms`, 9s) → fences broker → selects new leader from ISR → new leader's LEO becomes the new HW baseline → followers truncate if they had speculatively replicated past-HW data → resume. Produce latency spike during the 1–10s window.

**Whole rack loses power, 2 of 3 replicas down** → ISR=1 (just the surviving replica) → if `min.insync.replicas=2`, produces fail with `NotEnoughReplicasException`. Consumer reads work (HW is still valid). When a failed broker comes back, it catches up, ISR=2 again, produces resume.

**All ISR members down** → partition offline with `unclean=false`. Wait for recovery. With ELR (KIP-966), the first ELR member that comes back is elected safely.

**Network partition between controllers and broker** → broker fenced by controller → partitions hosted on that broker have leaders reassigned → when partition heals, broker rejoins, catches up. No split-brain because only one active controller exists (Raft) and only one leader per partition exists (epoch enforcement).

### ASCII diagram: the commit path

```
 Producer ──Produce(acks=all)──▶ Leader (broker-1)
                                    │
                                    │ 1. append to local log, LEO++
                                    │ 2. park request in ProducePurgatory
                                    │
                       ┌────────────┼────────────┐
                       │                         │
              Follower F1 Fetch            Follower F2 Fetch
              (pulls from leader)          (pulls from leader)
                       │                         │
              F1 appends, replies           F2 appends, replies
              with its LEO                  with its LEO
                       │                         │
                       └────────────┬────────────┘
                                    │
                   Leader updates min(LEO across ISR) = new HW
                                    │
                   ProducePurgatory checks: produced offset ≤ HW?
                                    │
                                    ▼
                   Yes → send ACK to producer
```

## Config reference

### Broker / cluster

| Key | Default | Notes |
|---|---|---|
| `default.replication.factor` | 1 | **Set to 3 in any real cluster.** |
| `min.insync.replicas` | 1 | Cluster default; **override topic-level to 2 with RF=3.** |
| `unclean.leader.election.enable` | false | Keep false for durability. |
| `replica.lag.time.max.ms` | 30000 | ISR eviction window. |
| `replica.fetch.min.bytes` | 1 | Follower fetch batching. |
| `replica.fetch.wait.max.ms` | 500 | Follower fetch max wait. |
| `replica.fetch.max.bytes` | 1048576 | Per-partition follower fetch cap. |
| `num.replica.fetchers` | 1 | Threads per broker for replication. Bump on brokers with many follower partitions. |
| `replica.high.watermark.checkpoint.interval.ms` | 5000 | HW persistence frequency. |
| `broker.session.timeout.ms` | 9000 | Time before controller fences an unresponsive broker. |
| `controller.quorum.election.timeout.ms` | 1000 | Raft election timeout on controllers. |

### Topic-level

| Key | Default | Notes |
|---|---|---|
| `replication.factor` | (broker default) | Immutable after topic create (requires reassignment). |
| `min.insync.replicas` | (broker default) | Override per topic. |
| `unclean.leader.election.enable` | (broker default) | Override per topic. |

### Producer (Confluent.Kafka / librdkafka)

| Key | Default | Notes |
|---|---|---|
| `acks` | `all` (when idempotence on) | Commit semantics. |
| `enable.idempotence` | false (must enable) | Exactly-once per partition per session. |
| `max.in.flight.requests.per.connection` | 5 | Must be ≤ 5 with idempotence. |
| `retries` | INT32_MAX (with idempotence) | Keep default. |
| `delivery.timeout.ms` | 120000 | Upper bound on producer.send retries. |
| `request.timeout.ms` | 30000 | Per-request timeout. |

## Config example: a durability-first topic

No C# snippet here — this section is broker/topic-side configuration. An example safe topic spec:

```bash
kafka-topics.sh --bootstrap-server b1:9092 \
  --create --topic payments.events \
  --partitions 24 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config unclean.leader.election.enable=false \
  --config retention.ms=1209600000 \
  --config compression.type=zstd
```

Paired producer config (.NET) for EOS v2:

```csharp
var cfg = new ProducerConfig
{
    BootstrapServers  = "b1:9092,b2:9092,b3:9092",
    EnableIdempotence = true,
    Acks              = Acks.All,
    // Transactional writes → exactly-once across multiple topics/partitions
    TransactionalId   = "payments-processor-1",
    CompressionType   = CompressionType.Zstd,
};
```

## Senior-level gotchas

- **`min.insync.replicas` must be strictly less than RF.** Setting `min.insync.replicas = RF` means losing any single replica halts writes — no maintenance headroom. RF=3 + min.ISR=2 is the canonical balance.
- **A "healthy" cluster can still lose data if `acks < all`.** Auditing producer configs across hundreds of microservices is the most common source of silent data-loss risk. Enforce via config guardrails (producer config validators, Confluent's broker-side `acks` enforcement, or a linter in CI).
- **Adding replicas after the fact is expensive.** Reassignment copies the full log to the new replica. Throttle it with `leader.replication.throttled.rate` / `follower.replication.throttled.rate` or you'll saturate network and starve clients.
- **"ISR=1" alerts are the canary.** An alert on `kafka.cluster:type=Partition,name=UnderMinIsr` catches the real danger — not "ISR shrank" but "ISR shrank below min". Always page on this.
- **Leader skew happens.** After rolling restarts, leadership can concentrate on a few brokers. Kafka auto-rebalances preferred leaders (`auto.leader.rebalance.enable=true`, default) every `leader.imbalance.check.interval.seconds` (300s), but only if imbalance exceeds `leader.imbalance.per.broker.percentage` (10%). On sensitive clusters, monitor leader distribution directly.
- **`unclean.leader.election.enable=true` buried in a default broker config is a silent liability.** Defaults changed to `false` in 2.4, but older clusters carrying forward their `server.properties` sometimes still have it true. Audit.
- **Rack awareness matters.** `broker.rack` + topic assignment with `--rack-aware` places replicas in different failure domains (racks/AZs). Without it, RF=3 can put all three replicas in the same AZ and a single AZ outage kills the partition.
- **Idempotent producer resets on broker restart with expired transactional.id.transaction.timeout.** If your producer sits idle for longer than `transaction.max.timeout.ms` (broker, default 15 min), the PID is fenced. You'll see `InvalidProducerEpoch` on the next send. Producers should handle this by recreating the producer, not by retrying.
- **Replication throughput is bounded by `num.replica.fetchers` × per-fetcher throughput.** On a broker replicating many high-rate partitions as follower, the default of 1 fetcher thread is a bottleneck; bump to 4–8.
- **HW propagation is one round-trip slower than LEO propagation.** The follower learns the new HW on its *next* fetch after it advanced its own LEO. This is why consumer-observable commit latency is always ≥ 2× the replication round-trip time under `acks=all`.

## References
- [Confluent — Kafka Replication](https://docs.confluent.io/kafka/design/replication.html)
- [KIP-966: Eligible Leader Replicas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966:+Eligible+Leader+Replicas)
- [Jack Vanlightly — KIP-966: Fixing the Last Replica Standing issue](https://jack-vanlightly.com/blog/2023/8/17/kafka-kip-966-fixing-the-last-replica-standing-issue)
- [KIP-101: Alter Replication Protocol to use Leader Epoch for Truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation)
- [Apache Kafka — Broker Configs (4.1)](https://kafka.apache.org/41/configuration/broker-configs/)
- [Lydtech — Kafka Replication & Min In-Sync Replicas](https://www.lydtechconsulting.com/blog/kafka-replication)
- [Bytefreak — High Watermark in Apache Kafka Explained](https://bytefreak.dev/high-watermark-in-apache-kafka)
