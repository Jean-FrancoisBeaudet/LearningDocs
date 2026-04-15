# Kafka Architecture: Brokers, KRaft, and the Request Pipeline

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- A Kafka 4.x cluster is **brokers + a KRaft controller quorum**. No ZooKeeper. Controllers maintain `__cluster_metadata`, a Raft-replicated single-partition log; brokers are stateless followers of that log for metadata purposes.
- The controller quorum is **2n+1 nodes** (typically 3 or 5). One is the **active controller** (Raft leader); the rest are hot-standbys that have the metadata log already replicated, making failover a sub-second affair.
- Clients speak the **Kafka wire protocol** (binary, TCP, length-prefixed). They first call `Metadata` to learn who leads each partition, then go direct to that leader for `Produce`/`Fetch`.
- Each broker has a **reactor-style request pipeline**: Acceptor → Processor (network) threads → request queue → I/O (request-handler) threads → **Purgatory** (for delayed ops) → response queue → Processor → socket.
- **Purgatory** is the key piece for understanding Kafka latency. Produce `acks=all` and Fetch with `fetch.min.bytes > 1` both park in purgatory until a condition (replication / bytes / timeout) is satisfied. It is backed by a hierarchical timing wheel — O(1) insert, O(1) expire.
- KIP-853 (dynamic quorum) in 3.9 and KIP-848 (next-gen consumer protocol) in 4.0 are the two big 2024–2026 architectural shifts worth knowing.

## Deep dive

### Cluster topology in KRaft 4.x

```
                 ┌────────────────────────────────────┐
                 │          Controller Quorum         │
                 │   (Raft, 2n+1 nodes, n>=1)         │
                 │                                    │
                 │   [c1*]──[c2]──[c3]                │
                 │     ^  active controller           │
                 │     │                              │
                 │     │ replicates __cluster_metadata│
                 └─────┼──────────────────────────────┘
                       │ push metadata deltas
                       ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ broker-1 │  │ broker-2 │  │ broker-3 │
        │ (leader  │  │ (leader  │  │ (leader  │
        │  for P0) │  │  for P1) │  │  for P2) │
        └──────────┘  └──────────┘  └──────────┘
             ▲             ▲             ▲
             │             │             │
             └──── clients (producers/consumers/admin) ────┘
```

Three node roles exist:
- **Controller-only** (`process.roles=controller`) — recommended for production. 3 or 5 dedicated small machines.
- **Broker-only** (`process.roles=broker`) — hosts topic data, handles producers/consumers.
- **Combined** (`process.roles=broker,controller`) — dev/test/single-box. Supported but discouraged for real workloads because of resource contention and blast-radius concerns.

### The controller quorum and `__cluster_metadata`

The KRaft controllers implement Raft (a variant of it — KIP-595). Their state machine is the cluster metadata: topics, partitions, replica assignments, ISR sets, configs, ACLs, quotas, broker registrations, producer IDs.

All of that state is stored as records in a single-partition internal topic **`__cluster_metadata`**, replicated across the quorum. The **active controller** is the Raft leader; followers replicate the log and can take over if the leader fails.

Important properties:

- **Event-sourced.** The controller's in-memory state is rebuilt by replaying the metadata log. Snapshots (`log.snapshot.*`) bound recovery time.
- **Brokers subscribe to the metadata log.** Each broker maintains its own replica of `__cluster_metadata` via the Raft fetch protocol. When the active controller appends a record (say, "partition P3 leader moved from broker-2 to broker-3"), every broker sees it within milliseconds and updates local state.
- **NO_OP records** are written periodically by the active controller. Raft is pull-based in KRaft, so followers need a heartbeat signal; `NoOpRecord` serves that purpose and confirms leader liveness.
- **Metadata versioning** via `metadata.version` (aka feature flags) gates KIP rollouts. You bump it with `kafka-features.sh upgrade` after all nodes are on the new binary.
- **Dynamic quorum (KIP-853, 3.9+)** lets you add/remove controller voters at runtime without cluster downtime — the old world required editing static config and rolling.

Compare this to the old ZK model where metadata was a tree of znodes, controller failover meant the new controller doing a full ZK read of every topic/partition (minutes on large clusters), and the broker fanout of metadata was a bespoke push protocol. KRaft is **strictly simpler and strictly faster**.

### Broker internals: the request pipeline

A Kafka broker is, at its core, a reactor network server plus a log storage engine. The request path looks like this:

```
client socket
    │
    ▼
┌──────────────┐       ┌──────────────────┐
│  Acceptor    │──────▶│  Processor #N    │  (network threads,
│  (1 thread)  │       │  (num.network.-  │   do SSL/SASL, framing,
└──────────────┘       │   threads)       │   read/write sockets,
                       └──────┬───────────┘   non-blocking)
                              │ enqueue Request
                              ▼
                       ┌──────────────────┐
                       │ Request Queue    │  (bounded, queued.max.requests)
                       └──────┬───────────┘
                              │ dequeue
                              ▼
                       ┌──────────────────┐
                       │ KafkaApis /      │  (I/O threads,
                       │ RequestHandler   │   num.io.threads)
                       │ pool             │   run the actual logic
                       └──────┬───────────┘
                              │
                  ┌───────────┼───────────┐
                  │           │           │
         completes now    Purgatory   LogSegment IO
         (e.g. acks=1)    (waiting)   (append/fetch)
                  │           │           │
                  └───────────┼───────────┘
                              ▼
                       ┌──────────────────┐
                       │ Response Queue   │
                       └──────┬───────────┘
                              ▼
                       ┌──────────────────┐
                       │ Processor        │
                       └──────┬───────────┘
                              ▼
                          client socket
```

#### Acceptor & Processors (network threads)
- **One Acceptor** per listener (`listeners=PLAINTEXT://...,SASL_SSL://...` → one per listener). It does `accept()` and hands the socket to a Processor.
- **`num.network.threads`** (default **3**) Processors per listener. Each Processor owns a set of sockets, runs a Java NIO `Selector`, handles the TLS/SASL handshake, reads length-prefixed Kafka protocol frames, and enqueues a `Request` object onto the shared request queue.
- Processors never do disk IO. They only push bytes.

#### I/O threads (request-handler pool)
- **`num.io.threads`** (default **8**) threads pull from the request queue and run `KafkaApis` — the switch statement that dispatches `ProduceRequest`, `FetchRequest`, `ListOffsets`, `Metadata`, etc.
- For a Produce request, the I/O thread appends to the partition's active log segment via `Log.append()`, which writes to the page cache (fsync only if `flush.messages`/`flush.ms` say so — by default Kafka relies on replication for durability, not fsync).
- For a Fetch request, the I/O thread runs `Log.read()` which usually `sendfile()`s directly from page cache → socket. Zero-copy.

#### Purgatory: the delayed-operations data structure

Two scenarios make a request "not complete now":

1. **Produce with `acks=all`**: the leader has appended locally but must wait for all in-sync followers to fetch that offset before replying.
2. **Fetch with `fetch.min.bytes > 1` or `fetch.max.wait.ms > 0`**: not enough new data yet; wait until either bytes accumulate or the timeout fires.

If the I/O thread blocked on these, the handler pool would be saturated by idle waits. Instead, Kafka parks the `Request` object in **Purgatory** — a map-like structure keyed by "watch key" (e.g. `TopicPartition`) backed by a **hierarchical timing wheel** for timeout handling.

Key properties of the timing wheel:
- **O(1) insert and O(1) tick-expiry** — no log-factor like a priority queue.
- Requests are re-examined on two triggers: (a) something relevant happens (follower fetch advanced the HW, or new data appended), or (b) the timeout bucket rolls.
- When the watch condition is met, the request is pulled out, response is built, enqueued on the response queue.

The standard purgatories are:
- **ProducePurgatory** — waits for replication (HW advance).
- **FetchPurgatory** — waits for bytes/time.
- **DelayedJoinPurgatory** — consumer group rebalance timing.
- **DelayedHeartbeatPurgatory** — session timeouts.
- **DelayedCreateTopics / DelayedDeleteTopics** — metadata ops.

Understanding purgatory is how you reason about Kafka tail latency. If your P99 produce latency spikes, the answer is almost always "replication got slow and requests sat in ProducePurgatory longer" — not "disks got slow".

#### KafkaRequestHandler threading and backpressure
- The request queue is **bounded** (`queued.max.requests`, default 500). When full, Processors stop reading from sockets → TCP backpressure pushes back on the client. This is an intentional overload-shedding mechanism.
- The response queue is per-Processor (not shared), so a slow client only backs up its own Processor, not the whole broker.
- `request.timeout.ms` on the client bounds how long it will wait; in-flight requests are tracked by a correlation ID.

### The client-broker protocol

Kafka's wire protocol is binary, length-prefixed, versioned per-API. It is **not HTTP**. Each API (Produce, Fetch, Metadata, JoinGroup, SyncGroup, Heartbeat, OffsetCommit, OffsetFetch, etc.) has its own numeric key and independent version evolution.

Bootstrap sequence for a producer:
1. Connect to a `bootstrap.servers` host; send `ApiVersions` to negotiate supported versions.
2. Send `Metadata(topics=[...])` to learn the cluster: broker list, partition leaders, controller id.
3. For each partition you want to produce to, open a connection to that partition's **leader broker** and send `Produce`.
4. If `NOT_LEADER_FOR_PARTITION` comes back, refresh metadata (a leader election happened) and retry.

This is why you **cannot put an L4 TCP load balancer in front of Kafka**. A Produce for partition 7 *must* go to the current leader of partition 7 — there is no routing logic on any other broker. `advertised.listeners` is the mechanism by which a broker tells clients "here is my real address", and it must resolve directly from the client.

For TLS/SASL, each listener declares a security protocol (`PLAINTEXT`, `SSL`, `SASL_PLAINTEXT`, `SASL_SSL`) and the Processor implements the handshake. Modern deployments use `SASL_SSL` with OAUTHBEARER (KIP-768) or SCRAM-SHA-512.

### KIP-848: the new consumer group protocol

The old consumer rebalance (JoinGroup/SyncGroup "stop the world") had two big problems: every rebalance paused every consumer, and the protocol was leader-on-client (the group leader computed the assignment). KIP-848, enabled by default for new groups in 4.0, moves the assignment logic to the **group coordinator on the broker**, uses **incremental rebalances**, and replaces heartbeats with a streaming protocol. Practical effect: rebalances are faster, partial, and far less disruptive. Old-protocol groups continue to work during migration.

### Storage layout on a broker

Each broker has one or more `log.dirs` directories. Inside:

```
/var/lib/kafka/data/
  ├── meta.properties          # cluster.id, node.id, directory.id
  ├── __cluster_metadata-0/    # KRaft metadata log (on controllers)
  ├── orders-0/                # topic "orders" partition 0
  │     ├── 00000000000000000000.log
  │     ├── 00000000000000000000.index
  │     ├── 00000000000000000000.timeindex
  │     ├── 00000000000000000000.snapshot
  │     └── leader-epoch-checkpoint
  └── orders-1/
        └── ...
```

Details in `03-topics-and-partitions.md`. The important architectural point is that **each partition is an independent log directory**, and each broker holds a subset of all partitions (both as leader and as follower replica).

## Config reference

| Key | Default | Notes |
|---|---|---|
| `process.roles` | (none) | `controller`, `broker`, or `broker,controller`. |
| `controller.quorum.voters` | (none, required) | Static list, `id@host:port`. Prefer `controller.quorum.bootstrap.servers` with KIP-853 dynamic quorum. |
| `node.id` | (required) | Unique across cluster. |
| `num.network.threads` | 3 | Processors per listener. Bump to 6–8 on high-connection brokers. |
| `num.io.threads` | 8 | Request-handler pool size. Rule of thumb: 2× cores for mixed workloads. |
| `queued.max.requests` | 500 | Bounded request queue; backpressure knob. |
| `socket.send.buffer.bytes` | 102400 | TCP SO_SNDBUF. |
| `socket.receive.buffer.bytes` | 102400 | TCP SO_RCVBUF. |
| `socket.request.max.bytes` | 104857600 (100 MB) | Max request size accepted. Must be ≥ any producer's `max.request.size`. |
| `replica.fetch.max.bytes` | 1048576 | Per-partition max bytes fetched by followers. |
| `replica.socket.timeout.ms` | 30000 | Follower socket timeout. |
| `metadata.log.dir` | first of `log.dirs` | Controller/KRaft metadata log location. Put on a fast disk. |
| `metadata.log.max.record.bytes.between.snapshots` | 20 MB | Snapshot trigger. |
| `broker.heartbeat.interval.ms` | 2000 | Broker → controller heartbeat. |
| `broker.session.timeout.ms` | 9000 | If exceeded, controller marks broker fenced. |
| `group.coordinator.new.enable` | true (4.x) | KIP-848 next-gen consumer group protocol. |

## Senior-level gotchas

- **Don't co-locate controllers with heavy brokers.** The controller's log is small but latency-sensitive. Noisy-neighbor IO on a combined-mode broker can stall leader election during a failure.
- **Always size `num.io.threads` against disk, not CPU.** If your disk is the bottleneck (common on HDD-backed hosts), more I/O threads just deepens queues.
- **Purgatory size is a leading indicator of trouble.** Monitor `kafka.server:type=DelayedOperationPurgatory,name=PurgatorySize,delayedOperation=Produce`. Growing Produce purgatory = replication is lagging = ISR will shrink soon.
- **`advertised.listeners` vs `listeners` confusion is the #1 Kafka networking bug.** `listeners` is what the broker binds to locally; `advertised.listeners` is what it tells clients to reach it on. In k8s / NAT / cloud setups these differ. Get this wrong and clients can reach brokers for bootstrap but not for partition leaders.
- **ApiVersions negotiation matters for mixed-version fleets.** During a rolling upgrade, clients will see different broker versions and must negotiate. Don't pin client API versions manually.
- **Metadata refresh storms** happen if thousands of clients all hit `metadata.max.age.ms` at the same boot time. Stagger clients or use jittered refresh.
- **`__cluster_metadata` is not a topic you can consume with a normal client.** It's special. Don't try to `kafka-console-consumer.sh` it; use `kafka-metadata-shell.sh`.
- **KRaft controllers share fate with their disk.** Use fast, reliable local NVMe for the metadata log dir. If the metadata log is corrupted on a majority of voters, recovery is ugly. Back up `meta.properties` and snapshot dirs periodically.
- **A broker being "alive" means "it heartbeats to the controller".** A broker with a dead disk can still heartbeat; partitions hosted on that disk will fail but the broker isn't fenced. Use JBOD-aware monitoring.

## References
- [Apache Kafka — KRaft operations](https://kafka.apache.org/41/operations/kraft/)
- [Confluent Developer — Inside the Apache Kafka Broker](https://developer.confluent.io/courses/architecture/broker/)
- [Confluent blog — Kafka Producer Internals: The Producer Request Lifecycle](https://www.confluent.io/blog/kafka-producer-internals-handling-producer-request/)
- [Red Hat — A deep dive into Apache Kafka's KRaft protocol](https://developers.redhat.com/articles/2025/09/17/deep-dive-apache-kafkas-kraft-protocol)
- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum)
- [KIP-853: KRaft Controller Membership Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes)
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol)
