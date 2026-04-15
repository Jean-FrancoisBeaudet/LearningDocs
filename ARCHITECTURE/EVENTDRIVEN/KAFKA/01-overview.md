# Kafka Overview: What It Is and Why It Exists

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR
- Kafka is a **distributed, partitioned, replicated commit log** exposed as a pub/sub messaging system — not a queue, not a database, but a log you can tail, replay, and fan out.
- Born at LinkedIn in 2010 (Kreps/Narkhede/Rao), open-sourced 2011, Apache TLP 2012, Confluent founded 2014. The design goal was a **unified substrate for all moving data** in a large org.
- **Kafka 4.0 (March 2025) removed ZooKeeper entirely**. KRaft (KIP-500) is now the only supported metadata plane. ZK-mode clusters must migrate on 3.9 before upgrading.
- Core primitive: an **append-only log per partition**. Ordering is per-partition, consumers track their own offsets, and the broker is a dumb, fast, sequential-IO file server.
- Use Kafka when you need **durable event streams with replay, multiple independent consumers, and >100k msg/s** — not for point-to-point RPC, not for small low-latency RabbitMQ-style work queues.
- Modern defaults in 4.x: **KRaft**, **EOS v2** (idempotent + transactional producers), **cooperative-sticky rebalancing**, **tiered storage (KIP-405) GA**, **queues for Kafka (KIP-932) preview** for share-group semantics.

## Deep dive

### What Kafka actually is
Kafka is often described as "a message queue", but that mental model is wrong and will get you into trouble. Kafka is a **distributed commit log** — a sequence of immutable records, partitioned for horizontal scale, replicated for durability, and retained by time or size rather than consumed-and-deleted.

The consequences of that model are everything that makes Kafka different from RabbitMQ, ActiveMQ, or SQS:

- **Consumers don't destroy data.** A message is not "acked off the queue". Instead, each consumer (or consumer group) maintains a monotonically increasing **offset** pointer into the log. New consumers can start at the beginning and replay history; existing consumers can rewind for reprocessing.
- **Broadcast is free.** Ten independent consumer groups reading the same topic cost the broker essentially the same as one, because the broker is just serving sequential reads out of the page cache.
- **Ordering is per-partition, not per-topic.** This is the single most misunderstood property of Kafka. If you need global ordering you need one partition, and one partition means one consumer and no horizontal scale — so you almost never want that.
- **The broker is dumb.** No per-message ACLs inside a message, no server-side filtering, no selectors, no priority queues. All routing is "topic + partition + offset". This is a feature: it keeps the broker fast and bounds failure modes.

### A short history
- **2010** — LinkedIn starts Kafka to unify a sprawl of fragile ETL pipelines and activity-tracking feeds. The name comes from Franz Kafka, chosen by Jay Kreps because "it's a system optimized for writing".
- **2011** — Open-sourced.
- **2012** — Apache top-level project.
- **2014** — Confluent founded; Kreps's "The Log: What every software engineer should know about real-time data's unifying abstraction" is published.
- **2017** — Exactly-once semantics (KIP-98) and the idempotent producer land in 0.11.
- **2019** — KIP-500 proposed: remove ZooKeeper. KRaft work begins.
- **2022** — Kafka 3.3 marks KRaft as production-ready.
- **2024** — Kafka 3.7/3.8 mature KRaft; tiered storage (KIP-405) graduates early access.
- **March 2025** — **Kafka 4.0** ships. ZooKeeper support **removed**. KRaft is the only mode. KIP-848 next-gen consumer rebalance protocol enabled by default for new groups. Java 17 becomes the minimum for brokers, Java 11 for clients.
- **2025–2026** — Tiered storage GA, Queues for Kafka (KIP-932) in preview, share groups giving Kafka RabbitMQ-style competing-consumer semantics on top of the log.

### The log-based messaging model, concretely
A topic is a logical stream. Each topic is split into N **partitions**. Each partition is a log — a sequence of records `(key, value, headers, timestamp, offset)` — stored on exactly one broker at a time (the **leader**) and replicated to `replication.factor - 1` followers.

```
Topic: orders  (partitions=3, RF=3)

 Partition 0 ─┬─► [0][1][2][3][4][5][6] ◄── append only, leader on broker-1
              └─ replicated to broker-2, broker-3

 Partition 1 ─┬─► [0][1][2][3][4][5][6][7][8]
              └─ leader on broker-2

 Partition 2 ─┬─► [0][1][2][3]
              └─ leader on broker-3
```

A producer picks a partition (by key hash, or explicitly, or round-robin with sticky batching). A consumer group coordinates so that **each partition is read by exactly one consumer in the group** at a time. Add more partitions to get more parallelism; the ceiling on consumer parallelism in a group is the partition count.

### Why this shape wins at scale
Three architectural choices make Kafka fast:

1. **Sequential disk IO + page cache.** The broker writes by appending to a file and reads by `sendfile()`-ing that file to the socket. Random IO never happens on the hot path. Modern NVMe + Linux page cache makes this absurdly fast — Kafka regularly does 1+ GB/s per broker.
2. **Zero-copy reads.** `sendfile(2)` moves bytes from the page cache directly to the NIC without a userspace hop. Combined with the broker never decoding the message body, this means the broker scales almost linearly with NIC bandwidth.
3. **Batching everywhere.** Producers batch by `linger.ms` and `batch.size`; consumers fetch in bulk via `fetch.min.bytes`; replication itself is a specialized consumer. Per-record overhead gets amortized into "per-batch" overhead.

### Core use cases (what Kafka is genuinely good at)

- **Event-driven microservices backbone.** Services publish domain events (`OrderPlaced`, `PaymentCaptured`); other services subscribe independently. Decouples producers from consumers and gives you audit + replay for free.
- **Log aggregation / telemetry pipeline.** Replacement for Scribe/Flume/Fluentd central tier. Fan in from thousands of hosts, fan out to ES/Splunk/S3/Snowflake.
- **Stream processing substrate.** Kafka Streams, Flink, Spark Structured Streaming, ksqlDB — all consume from Kafka, do stateful transformation, and write back.
- **Change data capture (CDC).** Debezium tails transaction logs of Postgres/MySQL/SQL Server and publishes row-change events to Kafka. This is arguably the dominant "get data out of your monolith DB" pattern in 2026.
- **Event sourcing / storage of record.** With tiered storage (KIP-405) you can now keep years of history on S3/ADLS cheaply and still have Kafka as the source of truth.
- **Metrics & IoT ingest.** High-volume, low-fanout, time-ordered telemetry.

### Where Kafka is the wrong tool
- **Task queues with per-message retries, delay, priority, and dead-letter policy.** RabbitMQ, Azure Service Bus, or AWS SQS are far better. Kafka can be forced into this with share groups (KIP-932) but it's still a log under the hood.
- **RPC-style request/response.** Kafka's minimum end-to-end latency is tens of milliseconds at best; your REST/gRPC call is faster and simpler.
- **Tiny throughput, tiny team.** The operational surface (brokers, controllers, topic design, retention, monitoring) is non-trivial. If you do 100 msg/s and have three engineers, pick SQS/Service Bus.
- **Ad-hoc point-to-point messaging between two services with no replay requirement.** Just use HTTP or a managed broker.

### ZooKeeper → KRaft transition
Before 4.0, every Kafka cluster needed a ZooKeeper ensemble to store cluster metadata (topics, partitions, ACLs, configs) and run controller election. This had painful consequences:

- **Two systems to operate, monitor, secure, and upgrade.** ZK had its own JVM, its own config, its own failure modes, its own CVE stream.
- **Metadata scalability ceiling.** Controller failover meant reading all metadata from ZK and broadcasting it to brokers — this took minutes on large clusters (100k+ partitions).
- **Split-brain risk during network partitions** between the ZK ensemble and the brokers.

**KRaft (KIP-500)** replaces ZK with a Raft-based quorum *inside Kafka itself*. A small set of nodes (usually 3 or 5) run as **controllers** and maintain a replicated metadata log (`__cluster_metadata`). Brokers become pure followers of that log — they replay metadata updates like any other Kafka consumer.

Benefits in 4.x:
- **Single binary, single protocol.** One thing to deploy, secure, patch.
- **Faster controller failover** — seconds instead of minutes on large clusters, because the new controller already has the metadata log replicated locally.
- **Millions of partitions per cluster** feasible (old ZK mode topped out around 200k).
- **Simpler mental model**: metadata is just another Kafka log.

In Kafka 4.x you **cannot run ZK mode at all**. Migration from ZK-mode clusters must be done on 3.9 using the dual-write migration procedure, then upgraded to 4.x.

### When to use Kafka vs alternatives (2026 decision matrix)

| Need | Pick |
|---|---|
| Durable, replayable event log, many consumers, >100k msg/s | **Kafka** |
| Task queue with retries, delay, DLQ, priority | RabbitMQ, Azure Service Bus, SQS |
| Fully managed, don't want to run brokers, AWS-native | Kinesis or MSK Serverless |
| Fully managed, GCP-native | Pub/Sub |
| Ultra-low latency (<1 ms), in-memory, ephemeral | Redis Streams, NATS JetStream |
| Kafka protocol but want different storage/cost model | Redpanda, WarpStream, AutoMQ (Kafka-compatible) |
| Embedded event log inside a single app | Chronicle Queue, LMDB, rolling your own |
| Just HTTP calls between two services | Don't use a broker. |

The Kafka-compatible alternatives (Redpanda, WarpStream, AutoMQ, Confluent's Freight tier) are a real trend in 2026 — they speak the Kafka wire protocol but re-architect storage (often S3-backed) for lower cost. If you're greenfield on AWS with bursty traffic, they're worth evaluating.

## Config reference

Cluster-level defaults worth knowing from day one:

| Key | Default (4.x) | Notes |
|---|---|---|
| `process.roles` | (required) | KRaft: `broker`, `controller`, or `broker,controller` (combined mode). |
| `controller.quorum.voters` | (required) | Static quorum, e.g. `1@c1:9093,2@c2:9093,3@c3:9093`. |
| `node.id` | (required) | Unique per node in KRaft mode (replaces `broker.id`). |
| `inter.broker.protocol.version` | current | In 4.x, always a 4.x version; no ZK fallback. |
| `auto.create.topics.enable` | `false` (hardened default) | Leave off in prod. Kafka 4.x ships with safer defaults than 2.x. |
| `num.partitions` | 1 | Default for auto-created topics; irrelevant if you pre-create. |
| `log.retention.hours` | 168 (7 days) | Override per topic. |
| `log.segment.bytes` | 1 GiB | Segment rollover size. |
| `default.replication.factor` | 1 | **Always set to 3 in prod.** |
| `min.insync.replicas` | 1 | **Set to 2** with RF=3 for meaningful durability. |
| `group.coordinator.new.enable` | true (4.x) | Enables KIP-848 consumer group protocol. |

## Senior-level gotchas
- **"Ordering" means per-partition ordering.** If your domain requires strict ordering across all orders for a given customer, the partition key must be `customerId`. Choosing `orderId` splits that customer's events across partitions and loses ordering. This is the #1 rookie mistake.
- **Partition count is sticky.** You can increase partitions but never decrease, and increasing them **rehashes keys to different partitions**, which breaks any consumer that relies on keyed ordering. Pick partition count with growth in mind (see `03-topics-and-partitions.md`).
- **Retention is per-topic, not per-consumer.** If consumer A is down for a week and retention is 3 days, A's data is gone. Retention is a broker policy, not a queue guarantee.
- **`acks=all` without `min.insync.replicas >= 2` is a lie.** With `min.insync.replicas=1`, "all in-sync replicas acked" can mean "the leader acked itself" — no durability improvement over `acks=1`.
- **Auto-create topics is a footgun.** A typo in a producer config creates a new 1-partition, RF=1 topic silently. Disable it cluster-wide and enforce topic creation via IaC.
- **Don't put Kafka behind a load balancer.** Clients need to talk to the **leader of each partition** directly; an L4 LB round-robins and breaks the protocol. Use `advertised.listeners` correctly instead.
- **KRaft combined mode (broker+controller on same node) is fine for dev, discouraged for large prod.** Keep controllers dedicated on 3 or 5 small nodes.
- **Kafka is not a database.** No secondary indexes, no ad-hoc queries, no joins on the server. If you find yourself wanting those, you want Flink/ksqlDB reading from Kafka, or you want a different system.

## References
- [Apache Kafka 4.0 release notes](https://kafka.apache.org/40/getting-started/upgrade/)
- [Confluent blog: Apache Kafka 4.0 Release](https://www.confluent.io/blog/latest-apache-kafka-release/)
- [InfoQ: Kafka 4.0 — KRaft Simplifies Architecture](https://www.infoq.com/news/2025/04/kafka-4-kraft-architecture/)
- [Apache Kafka: Uses](https://kafka.apache.org/uses/)
- [Jay Kreps — The Log: What every software engineer should know](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum)
