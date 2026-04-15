# Kafka Tiered Storage — KIP-405 and the Remote Log Tier

> Version baseline: Apache Kafka 4.x (tiered storage GA since 3.6, mature in 4.x). Last researched: 2026-04.

## TL;DR

Kafka tiered storage (KIP-405) decouples storage from compute by splitting every topic's log into two tiers:

- **Local tier** — the classic on-broker log, handling writes and hot reads. Kept small (hours to a few days).
- **Remote tier** — closed log segments offloaded to an object store (S3, GCS, Azure Blob, HDFS). Retention here can be months, years, or effectively unlimited.

Producers and consumers don't change. Reads that hit the remote tier transparently go through the broker, which fetches the segment from object storage, caches it briefly, and serves it. The broker never *writes* to the remote tier in response to a producer — only *rolled, closed* segments are eligible for upload.

**Status in 4.x (2026):**
- GA since 3.6 (Oct 2023). Production-grade in 4.x.
- Pluggable `RemoteStorageManager` (RSM) — the Apache repo ships the SPI and a local-filesystem reference. S3 / GCS / Azure Blob implementations come from Aiven (open-source), AWS (MSK), and Confluent.
- KIP-1176 (in flight) extends tiering to cover even the *active* segment — further shrinks the local footprint.
- KIP-1150 (Aiven "Inkless" / Diskless Topics) is a more radical approach — brokers with zero local disk, writing directly to object storage. Related but distinct from KIP-405.

**Why it matters:**
- Retention no longer dictates broker disk size. You can keep **90 days** of a 1 GB/s topic without buying a PB of NVMe.
- Broker recovery after failure is faster — less local data to re-replicate.
- Cost: object storage is 10–20× cheaper per GB than NVMe; you trade cold-read latency for cost.

## Deep dive

### 1. Architecture

```
                 ┌─────────────────────────────────────┐
 Producer ──►    │  Broker                              │
                 │  ┌─────────────┐   ┌───────────────┐ │
                 │  │ Local log   │   │ RemoteLog     │ │
                 │  │ (hot tier)  │──►│ Manager (RLM) │ │
                 │  └─────────────┘   └──────┬────────┘ │
                 │                            │          │
                 │               ┌────────────▼────────┐ │
                 │               │ RemoteStorageMgr    │ │
                 │               │ (pluggable, per     │ │
                 │               │  backend)           │ │
                 │               └────────────┬────────┘ │
                 │                            │          │
                 │               ┌────────────▼────────┐ │
                 │               │ RemoteLogMetadata   │ │
                 │               │ Manager (RLMM)      │ │
                 │               └────────────┬────────┘ │
                 └──────────────────┬────────┼──────────┘
                                    │        │
                     ┌──────────────▼─┐   ┌──▼──────────────┐
                     │ Object Storage │   │ __remote_log_   │
                     │ (S3/GCS/Blob)  │   │ metadata topic  │
                     └────────────────┘   └─────────────────┘
```

Key components:

- **RemoteLogManager (RLM)** — per-broker orchestrator. Scans closed local segments, hands them to the RSM for upload, and prunes local copies. Handles remote fetches on consumer read.
- **RemoteStorageManager (RSM)** — **pluggable**. The SPI (`org.apache.kafka.server.log.remote.storage.RemoteStorageManager`) defines `copyLogSegmentData`, `fetchLogSegment`, `fetchIndex`, `deleteLogSegmentData`. Backend-specific implementation.
- **RemoteLogMetadataManager (RLMM)** — stores metadata about which segments exist remotely, their start/end offsets, leader epochs, etc. The default implementation (`TopicBasedRemoteLogMetadataManager`) uses an internal compacted Kafka topic `__remote_log_metadata` — consistency of the metadata store is thus as strong as Kafka itself.

### 2. Write path

1. Producer sends records to the leader's local log (standard path).
2. Replication to followers (standard path).
3. When a segment rolls (time or size), it becomes eligible for remote upload.
4. The leader's RLM picks up the rolled segment, asks the RSM to upload the `.log`, `.index`, `.timeindex`, `.snapshot`, and leader-epoch checkpoint files to the remote backend.
5. On success, the RLMM writes a metadata event (segment uploaded, offset range, leader epoch info) to `__remote_log_metadata`.
6. Once the segment is safely in the remote tier **and** beyond the local retention threshold (`local.retention.ms` / `local.retention.bytes`), the RLM deletes the local copy.

Only the *leader* uploads. Followers mirror the leader's local log as usual; on leader change, the new leader resumes uploads based on what's already registered in the metadata topic.

### 3. Read path

For reads within the **local** range (recent offsets), behavior is unchanged — direct page-cache reads.

For reads in the **remote** range:
1. Consumer fetch arrives at the leader.
2. RLM locates the segment(s) in the RLMM.
3. RSM fetches the segment data (or a range of it) from object storage.
4. Broker streams bytes to the consumer.
5. Brokers typically keep a small **remote segment cache** (`remote.log.reader.max.pending.tasks`, local tmpfs or small SSD) to amortize re-reads of the same segment.

Latency characteristic:
- Hot local read: **< 10 ms**.
- Cold remote read (S3, first byte): **50–300 ms**; throughput steady-state can still be 100+ MB/s per connection.
- Reading an entire historical backlog can saturate object-store egress — watch for throttling.

### 4. Backends (RSM implementations)

| Backend | Provider | License | Notes |
|---|---|---|---|
| **Aiven tiered-storage-for-apache-kafka** | Aiven (open source) | Apache 2.0 | Supports AWS S3, GCS, Azure Blob. De-facto OSS standard; used by Strimzi, Aiven Cloud, many on-prem deployments. Includes optional client-side encryption and chunking. |
| **AWS MSK tiered storage** | AWS | Proprietary (MSK-integrated) | S3-backed, transparent. Enabled per-topic via MSK API. Cost: tiered storage tier pricing. No access to raw S3 buckets. |
| **Confluent Tiered Storage / Infinite Storage** | Confluent Platform / Cloud | Proprietary (CCL) | S3, GCS, Azure Blob, HDFS. Differs from KIP-405 — Confluent had its own implementation since 2020, predating KIP-405. In Confluent Cloud it's "Infinite Storage" (just set retention higher). |
| **HDFS** | Reference / community | Apache 2.0 | Less common; mainly for Hadoop shops. |
| **LocalTieredStorage** | Apache Kafka reference | Apache 2.0 | File-system-based. Testing / demos only. |

### 5. Trade-offs

#### Benefits

- **Unbounded retention** — years of data at object-store economics.
- **Smaller brokers** — 1–2 TB of local NVMe per broker can cover months of retention if cold data lives on S3.
- **Faster operational ops** — partition reassignment, broker replacement, and cluster expansion only shuffle hot data; remote segments don't move (they're shared).
- **Simpler storage management** — no more "did we buy enough disk?" conversations; retention changes no longer need broker fleet changes.
- **Disaster recovery** — some backends (Aiven, Confluent) can point a fresh cluster at the same remote bucket and resume consumption.

#### Drawbacks and trade-offs

- **Cold-read latency**: object-store first-byte latency is 50–300 ms. A consumer resetting to `earliest` on a long-retention topic will feel this.
- **Object-store egress costs**: re-reading everything off S3 is not free. Forensic replays of years of data can be expensive.
- **Operational complexity**: new failure modes (bucket IAM, network partition to object store, metadata topic corruption).
- **Metadata overhead**: `__remote_log_metadata` is a compacted internal topic; on very high partition counts it itself can become a hotspot. Monitor its lag and size.
- **Compaction incompatibility**: early implementations did not support compacted topics in the remote tier — confirm on your version. KRaft-era tiered storage (4.x) does support compaction, but with caveats (tombstones must traverse the remote tier).
- **Transactional topics**: idempotent/transactional producers work, but `__transaction_state` is never tiered.
- **Deletion semantics**: topic deletes must propagate to the remote store. A failed RSM delete can leave "ghost segments" paying for themselves forever; monitor your bucket.
- **Not a backup**: tiered storage is part of the cluster's live state. If someone truncates a topic, the remote segments get deleted too. Use actual backups for DR against accidental deletion.

### 6. When to use

**Good fits:**
- Compliance / audit topics with multi-year retention.
- Replay use cases: rebuilding a read model from a `log-compacted` source, backfilling analytics, CDC replays.
- Event-sourcing stores where long history matters.
- Cost-optimization on large clusters with uneven access patterns (hot head, cold tail).

**Poor fits:**
- Ultra-low-latency consumers that occasionally reset to `earliest` — cold reads will hurt.
- Short-retention topics (hours to a few days). Overhead of tiering > benefit.
- Low-volume topics where `retention.bytes` is tiny anyway.
- Regulated environments where data residency forbids object-store usage.

### 7. Related: KIP-1150 / Diskless Topics

A separate (but philosophically related) line of work:

- **KIP-1150 / Aiven Inkless / "Diskless Topics"** — brokers hold *zero* local disk for these topics. Every produce writes directly to object storage. Designed for low-throughput, write-heavy, cheap-storage workloads.
- **WarpStream** (see `17-ecosystem-and-alternatives.md`) is a commercial product that builds an entire Kafka-API surface on this philosophy.

KIP-405 tiered storage keeps the traditional broker log but offloads old segments. KIP-1150 eliminates local disk entirely for those topics. Same destination (cheap object storage), different points on the latency/cost curve.

### 8. Managed offerings

#### Confluent Cloud — Infinite Storage
- Just set your retention to whatever you need. Storage is billed per-GB-month; compute billed separately.
- Predates KIP-405 — Confluent's implementation started in 2020 (KCS-based design) and the APIs are compatible but not identical.
- Works with *both* Dedicated and Enterprise clusters.

#### AWS MSK Tiered Storage
- Per-topic opt-in via `tier` config.
- Backed by S3 (managed bucket, no user access).
- Local retention still bound by EBS (up to 16 TB/broker; 30-broker cluster limit).
- Pricing: storage tier charged separately from the broker EBS.

#### Aiven for Apache Kafka (cloud)
- Ships the Aiven open-source RSM. You can use the same RSM on-prem for cross-compatibility.
- Supports all three big clouds for the remote tier, with client-side AES-256 encryption and chunking.
- "Aiven Inkless" (2025+) extends this with Diskless Topics.

#### Strimzi (Kubernetes)
- Strimzi 0.40+ has first-class support for configuring tiered storage via the `Kafka` CR (`spec.kafka.tieredStorage`).
- Typically pairs with the Aiven RSM.

## Config reference

### Cluster / broker — enabling tiered storage

| Property | Example | Notes |
|---|---|---|
| `remote.log.storage.system.enable` | `true` | Master switch; must be `true` to use tiering. |
| `remote.log.storage.manager.class.name` | `io.aiven.kafka.tieredstorage.RemoteStorageManager` | Your RSM plugin. |
| `remote.log.storage.manager.class.path` | `/opt/kafka/plugins/tiered-storage/*` | Classpath isolation. |
| `remote.log.storage.manager.impl.prefix` | `rsm.config.` | Prefix for RSM-specific configs. |
| `remote.log.metadata.manager.class.name` | `org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager` | Default; uses `__remote_log_metadata`. |
| `remote.log.metadata.manager.impl.prefix` | `rlmm.config.` | |
| `remote.log.metadata.manager.listener.name` | `PLAINTEXT` | Which listener the RLMM's Kafka client uses. |
| `remote.log.manager.task.interval.ms` | `30000` | How often RLM scans for segments to upload. |
| `remote.log.manager.thread.pool.size` | `10` | Upload/fetch parallelism. |
| `remote.log.reader.threads` | `10` | Remote-fetch serving threads. |
| `remote.log.reader.max.pending.tasks` | `100` | Backpressure for concurrent cold reads. |

### Aiven RSM plugin configs (example — S3)

```properties
rsm.config.storage.backend.class=io.aiven.kafka.tieredstorage.storage.s3.S3Storage
rsm.config.storage.s3.bucket.name=my-kafka-tiered
rsm.config.storage.s3.region=us-east-1
rsm.config.storage.aws.access.key.id=…
rsm.config.storage.aws.secret.access.key=…
# Chunking + encryption:
rsm.config.chunk.size=4194304                   # 4 MiB chunks
rsm.config.compression.enabled=true
rsm.config.encryption.enabled=true
rsm.config.encryption.key.pair.id=my-key
rsm.config.encryption.key.pairs=my-key
rsm.config.encryption.key.pairs.my-key.public.key.file=/etc/kafka/ts/pub.pem
rsm.config.encryption.key.pairs.my-key.private.key.file=/etc/kafka/ts/priv.pem
```

### Topic-level configs

| Property | Example | Notes |
|---|---|---|
| `remote.storage.enable` | `true` | Per-topic opt-in. |
| `local.retention.ms` | `86400000` (1 day) | How much to keep locally (must be ≤ `retention.ms`). |
| `local.retention.bytes` | `53687091200` (50 GB) | Local size cap. |
| `retention.ms` | `-1` (infinite) or e.g. `31536000000` (1 y) | Total retention including remote. |
| `retention.bytes` | `-1` | Total size cap (rarely used with tiering). |
| `segment.bytes` | `536870912` (512 MiB) | Smaller segments upload faster; too small = too many objects. |
| `segment.ms` | `86400000` (1 day) | Force roll even if segment not full. |

### Enabling per topic

```bash
bin/kafka-topics.sh --bootstrap-server broker:9092 --create \
  --topic orders --partitions 24 --replication-factor 3 \
  --config remote.storage.enable=true \
  --config local.retention.ms=86400000 \
  --config retention.ms=31536000000 \
  --config segment.bytes=536870912
```

## Comparison — managed tiered-storage offerings

| Feature | Apache Kafka + Aiven RSM (OSS) | Confluent Cloud (Infinite) | AWS MSK Tiered | Aiven for Kafka |
|---|---|---|---|---|
| KIP-405 based | Yes | No (pre-KIP-405 impl) | Yes | Yes |
| Backends | S3, GCS, Azure Blob | S3, GCS, Azure Blob | S3 only (managed) | S3, GCS, Azure Blob |
| Client-side encryption | Optional (RSM) | Configurable | Managed KMS | Yes (plugin) |
| Max retention | Unlimited | Unlimited | Unlimited | Unlimited |
| Per-topic opt-in | Yes | Yes (implicit via retention) | Yes | Yes |
| Supports compacted topics | Yes (4.x) | Yes | Yes | Yes |
| Access to raw objects | Yes (your bucket) | No | No | Yes |
| Open source | Plugin is OSS | No | No | Plugin is OSS |
| Pricing model | Your S3 bill + compute | Per-GB-month storage + compute | MSK storage tier + brokers | Per-GB-month + compute |

## Senior-level gotchas

1. **Tiered storage is not a backup.** Deleting a topic deletes its remote segments. Accidents still need real backups.
2. **`local.retention.ms` ≤ `retention.ms`**, or Kafka refuses the config. Subtle: if you change `retention.ms` without adjusting local, brokers may keep more local data than you expected.
3. **Segment upload lag during bursts.** If produce rate > RSM upload throughput, the local log grows past `local.retention.*` until uploads catch up. Monitor `kafka.server:type=BrokerTopicMetrics,name=RemoteCopyLagBytes`.
4. **Cold read thundering herd.** A consumer group resetting to earliest on a long-retention topic can bombard the object store. Apply client-side rate limits; some backends also need explicit throttling configs.
5. **Leader churn replays uploads.** If leadership thrashes, the new leader re-discovers what's remote via the metadata topic — usually fine, but metadata-topic lag can cause temporary duplicate uploads. Monitor `__remote_log_metadata` health.
6. **Bucket IAM is a foot-gun.** The broker needs `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` (for prefix queries). Forgetting `DeleteObject` means deletes silently don't happen — you pay forever.
7. **Cross-region remote tier is usually wrong.** Remote reads pull across regions → latency + egress cost. Keep the bucket same-region as the cluster. Use cross-region **replication** (MirrorMaker) for DR, not cross-region tiering.
8. **Kafka 4.x tiered storage still doesn't handle encrypted-at-rest client payloads differently.** It uploads whatever bytes are in the segment. If you need per-record crypto, do it at the producer.
9. **`__remote_log_metadata` is sacred.** Losing it means you've lost the index of where segments live in your bucket. Treat it like `__consumer_offsets` — RF ≥ 3, `min.insync.replicas=2`, back it up.
10. **Confluent's pre-KIP-405 implementation and Apache's KIP-405 are not bytewise-compatible.** Migration between them requires consumer replay, not bucket copy.
11. **KIP-1176 (active-segment tiering)** will, when merged, further shrink the local footprint but introduces a read path for the freshest data that goes through object storage. Monitor that KIP if you're planning a 5-year data architecture.
12. **Don't enable tiered storage on tiny topics.** The per-object overhead (PUT/LIST costs, metadata entries) can dominate for topics with < 1 GB/day.
13. **Partition count inflation.** A 50 k-partition cluster produces 50 k "remote segment streams" — the RSM and metadata topic must scale. Test before committing.
14. **Compression codec choice matters more with tiering.** ZSTD cuts S3 storage bill proportionally — a 3× compression ratio literally reduces your long-retention cost by 3×.

## References

- [KIP-405: Kafka Tiered Storage](https://cwiki.apache.org/confluence/display/KAFKA/KIP-405%3A+Kafka+Tiered+Storage)
- [Apache Kafka — Tiered Storage docs](https://kafka.apache.org/41/operations/tiered-storage/)
- [Kafka Tiered Storage GA Release Notes (3.6)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Tiered+Storage+GA+Release+Notes)
- [Aiven — Tiered Storage for Apache Kafka (GitHub)](https://github.com/Aiven-Open/tiered-storage-for-apache-kafka)
- [Aiven — Kafka Tiered Storage in Depth](https://aiven.io/blog/apache-kafka-tiered-storage-in-depth-how-writes-and-metadata-flow)
- [Red Hat — Kafka tiered storage deep dive](https://developers.redhat.com/articles/2024/03/13/kafka-tiered-storage-deep-dive)
- [Confluent — Tiered Storage in Confluent Platform](https://docs.confluent.io/platform/current/clusters/tiered-storage.html)
- [KIP-1176: Tiered Storage for Active Log Segment](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1176%3A+Tiered+Storage+for+Active+Log+Segment)
- [KIP-1150: Diskless Topics (Aiven Inkless)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics)
