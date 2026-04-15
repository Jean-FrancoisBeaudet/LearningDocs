# Apache Kafka — Deep-Dive Notes

> Version baseline: **Apache Kafka 4.x (KRaft mode)**. Last researched: 2026-04.
> Client code: **.NET / C# with `Confluent.Kafka`** (librdkafka-based). Exception: `10-kafka-streams.md` also covers Streamiz and ksqlDB because Kafka Streams proper is JVM-only.

A senior-level, research-backed tour of Apache Kafka. Each note is self-contained: TL;DR → deep dive → config reference → C# snippet (where applicable) → senior-level gotchas → external references.

---

## Suggested reading order

If you're starting from scratch, read in order. If you already know the basics, jump to the section you need — every file cross-references the ones it depends on.

### Foundations
1. [01 — Overview](01-overview.md) — What Kafka is, history, log-based messaging, ZooKeeper → KRaft, when to use Kafka vs alternatives.
2. [02 — Architecture](02-architecture.md) — Brokers, KRaft controller quorum, `__cluster_metadata`, request pipeline, wire protocol.
3. [03 — Topics and Partitions](03-topics-and-partitions.md) — Log segments, index files, retention vs compaction, partition count planning, choosing a partition key.
4. [04 — Replication and ISR](04-replication-and-isr.md) — Replication factor, ISR, HW vs LEO, `min.insync.replicas`, unclean leader election, KIP-966 Eligible Leader Replicas.

### Clients
5. [05 — Producers](05-producers.md) — `acks`, idempotence, batching, compression, partitioners, back-pressure, `max.in.flight.requests.per.connection`.
6. [06 — Consumers](06-consumers.md) — Poll loop, `__consumer_offsets`, commit strategies, `max.poll.*`, fetch sizing, poison-pill handling, pause/resume/seek.
7. [07 — Consumer Groups and Rebalancing](07-consumer-groups-and-rebalancing.md) — Group coordinator, eager vs cooperative-sticky, static membership (KIP-345), incremental cooperative rebalance (KIP-429), next-gen rebalance protocol (KIP-848).
8. [08 — Delivery Semantics](08-delivery-semantics.md) — At-most / at-least / exactly-once, idempotent producer internals, transactions, EOS v2 (KIP-447), read-process-write pattern, `isolation.level`.

### Data & ecosystem
9. [09 — Log Compaction and Retention](09-log-compaction-and-retention.md) — `retention.ms`/`retention.bytes`, compaction algorithm, tombstones, `cleanup.policy`, CDC use cases, compaction pitfalls.
10. [10 — Kafka Streams](10-kafka-streams.md) — KStream/KTable/GlobalKTable, RocksDB state stores, windowing, joins, interactive queries, EOS in Streams. **JVM-only** — also covers .NET alternatives (Streamiz, ksqlDB, plain-consumer).
11. [11 — Kafka Connect](11-kafka-connect.md) — Distributed vs standalone, source/sink, SMTs, converters, DLQ, internal topics, EOS (KIP-618), Debezium/JDBC/S3/Elasticsearch examples.
12. [12 — Schema Registry](12-schema-registry.md) — Avro / Protobuf / JSON Schema, compatibility modes (BACKWARD / FORWARD / FULL / NONE + _TRANSITIVE), subject naming strategies, wire format, schema evolution.

### Operations
13. [13 — Security](13-security.md) — SASL (PLAIN/SCRAM/GSSAPI/OAUTHBEARER), SSL/mTLS, ACLs, OAuth 2.0 (KIP-768), delegation tokens, quotas, encryption at rest.
14. [14 — Observability](14-observability.md) — JMX metrics, essential broker/KRaft/producer/consumer metrics, Burrow, kafka-lag-exporter, OpenTelemetry tracing, dashboards to build.
15. [15 — Operations and Tuning](15-operations-and-tuning.md) — Cluster sizing, JVM/GC, page cache, JBOD vs RAID, OS sysctls, MirrorMaker 2, Cruise Control, rolling upgrades, KRaft quorum ops.
16. [16 — Tiered Storage](16-tiered-storage.md) — KIP-405, `RemoteLogManager`/`RemoteStorageManager`, S3/GCS/Azure backends, cold-read trade-offs, Confluent Infinite Storage / MSK / Aiven.
17. [17 — Ecosystem and Alternatives](17-ecosystem-and-alternatives.md) — Managed offerings (Confluent Cloud, AWS MSK, Event Hubs, Aiven, Upstash), Redpanda, WarpStream, client libraries, ksqlDB/Flink, UI tools.

---

## Cross-cutting themes

- **Durability vs availability**: [04](04-replication-and-isr.md) → [05](05-producers.md) → [08](08-delivery-semantics.md).
- **Exactly-once**: [05](05-producers.md) idempotence → [08](08-delivery-semantics.md) transactions → [10](10-kafka-streams.md) Streams EOS → [11](11-kafka-connect.md) Connect EOS.
- **Partition count as a design knob**: [03](03-topics-and-partitions.md) → [07](07-consumer-groups-and-rebalancing.md) → [15](15-operations-and-tuning.md).
- **Schema & data contracts**: [12](12-schema-registry.md) → [11](11-kafka-connect.md) → [10](10-kafka-streams.md).

## What's missing here by design

- Introductory "hello world" tutorials — plenty of those online, and [05](05-producers.md) / [06](06-consumers.md) give runnable C# in the right context.
- ZooKeeper mode — removed in Kafka 4.0. Mentioned only where historical context helps.
- Java client snippets — this repo standardised on .NET / `Confluent.Kafka`.

## How these notes were built

Each file was researched via WebSearch against 2024–2026 sources (kafka.apache.org, docs.confluent.io, KIP cwiki pages, Debezium, Confluent/AWS/Aiven/Redpanda/WarpStream engineering blogs) before writing, then structured using the same template (TL;DR / Deep dive / Config reference / C# snippet / Senior-level gotchas / References). Targets 400–800 lines per file at senior depth — trade-offs, failure modes, config defaults, and KIP citations rather than happy-path tutorials.
