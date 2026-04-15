# Consumer Groups & Rebalancing — Deep Dive

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- A **consumer group** is a logical identity (`group.id`) that lets N consumers share the work of M partitions such that each partition is owned by exactly one consumer. Scale-out = more consumers, bounded by partition count.
- The **group coordinator** (a broker elected per group) tracks membership, assignments, and offsets in `__consumer_offsets`. Each new membership state has a monotonic **generation/epoch** — any request carrying a stale generation is rejected.
- Since **Kafka 4.0, KIP-848** brings a **new rebalance protocol** where the broker (not the group leader/client) drives assignment. It eliminates the global sync barrier — no more stop-the-world. Opt-in via `group.protocol=consumer` on the client.
- Before KIP-848, the best-available client-side protocol is **incremental cooperative rebalancing** (KIP-429) with the `CooperativeStickyAssignor` — now the default in modern Kafka/Streams. Eager rebalancing is legacy.
- **Static membership** (KIP-345, `group.instance.id`) lets an instance restart within `session.timeout.ms` without triggering any rebalance — essential for low-downtime deployments. Combine with cooperative-sticky / KIP-848 for best results.
- Stop-the-world rebalances are still a real pitfall in mixed-assignor groups, misconfigured session timeouts, and any code that does heavy work inside the `PartitionsRevoked` callback.

## Deep dive

### Group coordinator & generation/epoch
- Each `group.id` is hashed onto a partition of `__consumer_offsets`; the **leader of that partition** is the **group coordinator** for the group.
- Coordinator state (members, assignments, committed offsets) is written to that same partition and replicated. Failover: new leader elected → new coordinator → replays log → resumes coordination.
- Every rebalance increments a **generation ID** (epoch). Every request from a consumer carries its current generation. A commit or heartbeat with a stale generation is rejected with `ILLEGAL_GENERATION` → consumer must rejoin.
- Under KIP-848, the broker owns the authoritative **member epoch** per consumer; there's no more "group leader" on the client side.

### The classic protocol (pre-KIP-848)
Before 4.x default:

1. **Join group**: all members send `JoinGroup` to the coordinator. Coordinator picks one as the **group leader** (first to join, usually).
2. **Sync group**: leader computes the assignment locally using the configured `PartitionAssignor` and sends it back in `SyncGroup`. Every other member receives their assignment in the response.
3. **Heartbeat** until next rebalance.

Any membership change (new consumer, departure, subscription change, partition count change) kicks off steps 1–2 again.

Two rebalance flavors existed:

- **Eager rebalancing (legacy)**: every consumer **revokes all partitions** on `JoinGroup`, processing stops globally, then reassignment happens. This is the "stop-the-world" most people complain about.
- **Incremental cooperative rebalancing (KIP-429, 2.4+)**: consumers keep their current partitions. Each rebalance round the assignor computes a target state and only **revokes partitions that need to move**. A second rebalance round then reassigns them. Result: partitions you keep never stop processing.

### Client-side assignors

| Assignor | Strategy | Cooperative? | Pros | Cons |
|---|---|---|---|---|
| `RangeAssignor` | Per-topic, assigns contiguous partition ranges per consumer | No (eager) | Simple, co-partitions across topics with same key space | Unbalanced if partition counts don't divide evenly; joins across topics skew |
| `RoundRobinAssignor` | Round-robin across (topic, partition) tuples | No (eager) | Even distribution | Reassigns everything on any change |
| `StickyAssignor` | Minimize reassignment while staying balanced | No (eager) | Less churn than round-robin | Still stop-the-world; the worst of both |
| `CooperativeStickyAssignor` (**default in modern clients**) | Sticky + KIP-429 incremental | **Yes** | Only the deltas pause. Minimum disruption | Two rebalance rounds per change (slightly more chatter) |

The broker doesn't care which assignor you use — it's client-side logic, negotiated via the `JoinGroup` request. **All members of a group must agree on an assignor**. If you roll out a new assignor across a group, mixed old/new membership will fall back to a common assignor (usually range) until rollout completes.

### KIP-848: Next-gen rebalance protocol (Kafka 4.0+)

KIP-848 moves assignment logic to the **broker**. Key changes:

- No more `JoinGroup`/`SyncGroup`. Replaced by a lightweight `ConsumerGroupHeartbeat` RPC that carries deltas (subscriptions, owned partitions) and returns per-member assignment deltas.
- Broker maintains authoritative group state and computes assignments server-side, incrementally.
- Every consumer sees only *its own* assignment target; no group leader election on the client.
- **No global synchronization barrier**. A new consumer joining doesn't pause existing members at all — they get incremental revoke/assign instructions.
- Assignor implementation is pluggable on the broker via `group.consumer.assignors` (server config). Two built-ins: `uniform` (even across members) and `range` (legacy-compatible).
- Rebalance times drop an order of magnitude: a benchmark of 10 consumers adding 900 partitions dropped from ~103 s (classic) to ~5 s (KIP-848).

Enabling:

- **Broker**: set `group.coordinator.rebalance.protocols=classic,consumer` (both, for migration) and `group.version=1` (enabled by default in 4.0). Server-side opt-in.
- **Client**: set `group.protocol=consumer` (default is still `classic` for compatibility). Confluent.Kafka exposes this via the raw config bag.
- Mixed-mode coexistence: a group can have some members on classic, some on KIP-848 during migration (the coordinator mediates). After migration, set the classic protocol off.

Client-side config `partition.assignment.strategy` is **ignored** under KIP-848 (server decides). Static membership still works.

### Static membership (KIP-345)

Set `group.instance.id=<stable unique id>` on each consumer. Effects:

- When a static member disconnects (crash, restart, network blip), the coordinator does **not** immediately rebalance. It waits up to `session.timeout.ms` for the same `group.instance.id` to rejoin.
- If the same ID rejoins in time → no rebalance. Generation unchanged. Assignment preserved.
- If it doesn't rejoin → rebalance happens as normal.

Use cases:
- Kubernetes pods with stable names (StatefulSet). Set `group.instance.id=$(POD_NAME)`.
- VM-backed services with stable hostnames.
- Anywhere a rolling restart would otherwise cause 2N rebalances (one per restart).

Pitfalls:
- IDs **must be globally unique within the group**. Two live members with the same ID → fencing; one will be evicted with `FENCED_INSTANCE_ID`.
- **Leaver must rejoin before `session.timeout.ms` expires**. Default 45 s is usually enough for a K8s pod restart; JVM-heavy apps may need 60–120 s.
- Increasing `session.timeout.ms` trades faster-rebalance-on-genuine-failure for no-rebalance-on-restart. Pick consciously.

### What triggers a rebalance

Triggers (all of these are rebalance events):

1. New consumer joins the group (`Subscribe` + first `Consume`).
2. Consumer leaves gracefully (`Close()`) — cooperative leave-group.
3. Consumer session times out (`session.timeout.ms` with no heartbeat).
4. Consumer exceeds `max.poll.interval.ms` → proactively leaves group.
5. Subscription changes (regex topic subscription matches a new topic, or subscription list changes).
6. A subscribed topic gains/loses partitions (repartitioning).
7. Group coordinator failover (new coordinator revalidates membership).

Under KIP-848, all of these still trigger coordinator work — but the "work" is an incremental delta, not a stop-the-world sync.

### Stop-the-world pitfalls (still relevant)

Even with cooperative-sticky or KIP-848, you can shoot yourself in the foot:

- **Mixing assignors**: deploying `CooperativeSticky` to half the group and `Range` to the other half → group falls back to `Range` (eager) and you lose all the cooperative benefit. Rollouts must be careful: add the new assignor to the list *first*, redeploy all members, then remove the old one.
- **Heavy `PartitionsRevoked` callbacks**: flushing state, closing DB connections, waiting for in-flight HTTP. If this takes > `rebalance.timeout.ms` (default `max.poll.interval.ms`), the coordinator forces you out of the group → "lost" not "revoked" → no chance to commit.
- **`enable.auto.commit=true` + rebalance**: offsets you'd committed moments ago may lag actual processing; new owner reprocesses.
- **Regex subscriptions** (`Subscribe(".*-events")`) cause a rebalance every time the topic list refreshes and a new match appears. On volatile topic names this is a storm source.
- **Small `session.timeout.ms`** (e.g., 10 s) + GC pause on the client = spurious eviction + rebalance.
- **Cross-language clients with different defaults**: Java default assignor changed over versions. Don't assume all members agree — set `partition.assignment.strategy` explicitly.

### Migration strategy: classic → KIP-848

1. Upgrade brokers to 4.x (KRaft required).
2. Ensure `group.coordinator.rebalance.protocols` includes both `classic` and `consumer`.
3. Roll out clients with `group.protocol=consumer` one member at a time. The coordinator handles mixed protocols during rollout.
4. Monitor `kafka.server:type=group-coordinator-metrics,name=num-groups-consumer-protocol` to confirm migration.
5. Once all clients are on the new protocol, optionally drop `classic` from `group.coordinator.rebalance.protocols` at the broker.

## Config reference

| Key | Default | Notes |
|---|---|---|
| `group.id` | (required) | Logical group identity |
| `group.instance.id` | `null` | Static membership — set for stable pods/VMs |
| `group.protocol` | `classic` (4.0) | Set to `consumer` for KIP-848 |
| `partition.assignment.strategy` | `[RangeAssignor, CooperativeStickyAssignor]` (Java client 3.x+) | Ignored under KIP-848 |
| `session.timeout.ms` | `45000` | Group-side liveness window |
| `heartbeat.interval.ms` | `3000` | Client heartbeat cadence |
| `max.poll.interval.ms` | `300000` | Processing deadline; leaving group proactively if exceeded |
| `rebalance.timeout.ms` (Java) | `max.poll.interval.ms` | Max time members can take during revoke/assign phases |
| `group.min.session.timeout.ms` (broker) | `6000` | Broker-side floor |
| `group.max.session.timeout.ms` (broker) | `1800000` (30 min) | Broker-side ceiling |
| `group.coordinator.rebalance.protocols` (broker) | `classic,consumer` (4.0+) | Server-side protocol support |
| `group.consumer.min.session.timeout.ms` (broker, KIP-848) | `45000` | Dedicated KIP-848 floor |
| `group.consumer.heartbeat.interval.ms` (broker, KIP-848) | `5000` | Server-pushed heartbeat interval for new protocol |
| `group.consumer.assignors` (broker) | `uniform,range` | Server-side assignor list (KIP-848) |

## .NET / C# snippet (Confluent.Kafka)

Example showing static membership, cooperative-sticky (classic protocol), graceful rebalance callbacks, and the KIP-848 opt-in.

```csharp
using Confluent.Kafka;

var config = new ConsumerConfig
{
    BootstrapServers = "broker1:9092,broker2:9092,broker3:9092",
    GroupId          = "payment-processor",

    // Static membership: tie to pod name (StatefulSet). No rebalance on restart-within-session.
    GroupInstanceId  = Environment.GetEnvironmentVariable("POD_NAME")
                       ?? Environment.MachineName,

    // Classic protocol with cooperative-sticky (pre-KIP-848 best practice).
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.CooperativeSticky,

    // --- Opt-in to KIP-848 (Kafka 4.0+ brokers only): uncomment below line,
    //     and the PartitionAssignmentStrategy above is ignored.
    // Extra config via the raw bag:
    //   new ConsumerConfig(existing) { { "group.protocol", "consumer" } }

    EnableAutoCommit      = false,
    SessionTimeoutMs      = 60_000,   // tolerate slow pod restart
    HeartbeatIntervalMs   = 5_000,
    MaxPollIntervalMs     = 600_000,  // allow longer processing per poll

    AutoOffsetReset       = AutoOffsetReset.Earliest,
    IsolationLevel        = IsolationLevel.ReadCommitted,
};

using var consumer = new ConsumerBuilder<string, byte[]>(config)
    .SetPartitionsAssignedHandler((c, parts) =>
    {
        // With cooperative-sticky, this is the *delta* assigned to us, not the full set.
        Console.WriteLine($"assigned delta: [{string.Join(", ", parts)}]");
        // Warm state for new partitions only. DO NOT rebuild state for previously-owned partitions.
    })
    .SetPartitionsRevokedHandler((c, parts) =>
    {
        // Cooperative: this is the *delta* being revoked, not necessarily all.
        // CRITICAL: commit + quiesce only these partitions, fast (< rebalance.timeout.ms).
        try { c.Commit(parts.Select(tpo => new TopicPartitionOffset(tpo.TopicPartition, tpo.Offset))); }
        catch (KafkaException ex) { Console.Error.WriteLine($"commit-on-revoke failed: {ex.Message}"); }

        FlushLocalStateFor(parts);
    })
    .SetPartitionsLostHandler((c, parts) =>
    {
        // We were evicted (session timeout). DO NOT commit — another member may already own these.
        Console.Error.WriteLine($"lost: [{string.Join(", ", parts)}] — state will be rebuilt when reassigned");
        AbandonLocalStateFor(parts);
    })
    .Build();

consumer.Subscribe("payments");

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

try
{
    while (!cts.IsCancellationRequested)
    {
        var cr = consumer.Consume(TimeSpan.FromMilliseconds(500));
        if (cr is null || cr.IsPartitionEOF) continue;
        ProcessPayment(cr);
        // commit strategy from 06-consumers.md
    }
}
finally
{
    consumer.Close(); // cooperative leave-group — no rebalance-after-close waiting for session timeout
}

static void ProcessPayment(ConsumeResult<string, byte[]> _) { }
static void FlushLocalStateFor(List<TopicPartitionOffset> _) { }
static void AbandonLocalStateFor(List<TopicPartition> _) { }
```

Notes:
- `SetPartitionsAssignedHandler` under cooperative-sticky gets only the **incremental** partitions added. Don't reset state you already hold.
- `PartitionsRevokedHandler` gets the **delta being revoked**. Fast path: commit those specific offsets and flush only their state.
- `PartitionsLostHandler` means you were evicted; another owner might already be moving ahead — don't commit.
- KIP-848 opt-in: add `{ "group.protocol", "consumer" }` to the config. Requires Kafka 4.0+ brokers.

## Senior-level gotchas
- **Cooperative-sticky rollout across a group must be staged**: add to the assignor list, deploy everyone, then remove the old. Skipping this causes the group to fall back to eager mid-rollout.
- **Static membership + unbounded `session.timeout.ms`** = never-rebalancing zombie members. If a static member genuinely dies and never restarts, its partitions hang until the timeout expires. 45–120 s is the sweet spot.
- **KIP-848 is broker-opt-in** via `group.coordinator.rebalance.protocols`. Client-only opt-in doesn't work if the broker isn't configured for it — you get classic behavior silently.
- **Regex `Subscribe` + frequent topic creation** = rebalance storms. Prefer explicit topic lists where topology is stable.
- **Rebalance callbacks run on the consumer poll thread.** Slow callbacks → callback timeout → eviction → another rebalance → callback storm.
- **One consumer per topic-partition; never more than partitions consumers in a group.** Extra consumers sit idle. Scale by increasing partitions (one-way door — partitions can be added but never removed, and adding partitions disrupts key→partition mapping for keyed records).
- **Generation / epoch mismatch** errors (`ILLEGAL_GENERATION`, `UNKNOWN_MEMBER_ID`) are normal during rebalance — the client rejoins automatically. But if they're *frequent*, you have a rebalance storm; investigate.
- **`group.instance.id` duplication across deployments** → `FENCED_INSTANCE_ID` → production outage. Use a dedup-proof source (pod name, not config file).
- **Partition count is effectively permanent.** Adding partitions reshuffles `hash(key) % numPartitions` routing → old and new records for the same key end up in different partitions → ordering breaks forever. Plan partition counts with ~3–5 year growth headroom.
- **Lost vs revoked** — the single most under-documented distinction. Lost = don't touch state/offsets; revoked = clean up gracefully. Confusing the two causes offset rollback bugs.

## References
- [Apache Kafka 4.1: Consumer Rebalance Protocol](https://kafka.apache.org/41/operations/consumer-rebalance-protocol/)
- [Confluent blog: KIP-848 — A New Consumer Rebalance Protocol](https://www.confluent.io/blog/kip-848-consumer-rebalance-protocol/)
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol)
- [KIP-429: Kafka Consumer Incremental Rebalance Protocol](https://cwiki.apache.org/confluence/x/vAclBg)
- [KIP-345: Introduce static membership protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-345%3A+Introduce+static+membership+protocol+to+reduce+consumer+rebalances)
- [Confluent blog: Incremental Cooperative Rebalancing in Apache Kafka](https://www.confluent.io/blog/incremental-cooperative-rebalancing-in-kafka/)
