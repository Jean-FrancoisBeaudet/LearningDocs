# Kafka Operations & Tuning

> Version baseline: Apache Kafka 4.x (KRaft-only). Last researched: 2026-04.

## TL;DR

Kafka's operational profile is unusual: it leans heavily on the **OS page cache** for read/write performance and keeps the JVM heap small. A well-run broker is mostly a big XFS file server with a ~6–12 GB JVM on top.

Ops priorities, in order:

1. **Right-size partitions per broker.** One broker comfortably handles **2 000–4 000** partitions with KRaft; 5 000–8 000 is an upper band with tuning; degradation is sharp beyond ~10 k. KRaft (4.x) raised the practical ceiling by roughly 10× vs ZooKeeper-era numbers.
2. **Small JVM heap, huge page cache.** 6–12 GB heap on 64–256 GB hosts. Never exceed ~31 GB (loses compressed oops). G1GC for < 16 GB heaps, ZGC (generational, JDK 21+) for larger.
3. **Fast local disks.** NVMe or equivalent, JBOD preferred over RAID, XFS with `noatime,nodiratime`.
4. **10/25 GbE minimum networking.** Replication doubles inbound bandwidth for RF=3 (2 follower fetches per produced byte).
5. **Automate rebalance** with **Cruise Control** — hand-editing partition assignments at scale is a bug farm.
6. **Cross-cluster replication** via **MirrorMaker 2** (Connect-based). For DR, active-active, geo-replication, migrations.
7. **KRaft controller quorum**: 3 controllers tolerate 1 failure, 5 tolerate 2. Dynamic quorum (KIP-853, Kafka 3.9+) is preferred for 4.x greenfield clusters — lets you add/remove controllers without downtime.

## Deep dive

### 1. Cluster sizing

#### 1.1 Partitions per broker

Each partition replica costs:
- An open file handle (usually 2–3 — active segment + index + timeindex).
- Memory for the index mmaps.
- Metadata entries in the controller's in-memory state.
- Replication fetcher thread capacity (followers fetch from leaders).

Practical ceilings (2026, KRaft):

| Partitions / broker | Regime |
|---|---|
| < 2 000 | Comfortable, sub-ms metadata ops, no tuning needed. |
| 2 000 – 4 000 | Target band for most production clusters. |
| 4 000 – 8 000 | Tuned clusters (bumped file descriptors, thread pools, careful topic design). |
| > 10 000 | Avoid. Controller failovers become slow, fetcher efficiency tanks. Shard across clusters instead. |

Cluster-wide: modern KRaft clusters have been demonstrated well past **1 million partitions** in benchmarks; real-world production rarely exceeds a few hundred thousand.

Formula (rough): `partitions = max(target_throughput / per_partition_throughput, desired_consumer_parallelism)`. Per-partition throughput is typically **5–50 MB/s** depending on message size and codec.

#### 1.2 Disk sizing

`disk per broker ≈ RF × peak_bytes_in_per_sec × retention_sec / broker_count × 1.3 safety`

Example: 200 MB/s produce, 7 days retention, RF=3, 9 brokers:
`3 × 200 MB/s × 604 800 s / 9 × 1.3 ≈ 52 TB per broker`.

Practical tips:
- Keep per-broker disk usage **< 70 %** — leaves headroom for rebalance and compaction.
- Enforce retention by **time** and **size** both (`retention.ms` + `retention.bytes`) so a runaway topic can't eat the disk.
- Tiered storage (see `16-tiered-storage.md`) lets you keep hot data local and offload cold to S3 — changes the sizing math entirely.

#### 1.3 Network

For RF=3, `egress ≈ 2 × ingress` on the leader broker (two followers pulling), plus consumer fetches. At 200 MB/s produce with 2 × consumer fan-out: `200 × 2 (replication) + 200 × 2 (consumers) = 800 MB/s = 6.4 Gb/s` per broker — 10 GbE is marginal, 25 GbE comfortable.

Enable `socket.send.buffer.bytes=1048576` / `socket.receive.buffer.bytes=1048576` (1 MB) on high-throughput links to let TCP window scaling open properly.

### 2. JVM and GC

**Heap size**:
- Target `6–12 GB` for typical workloads. Netflix and Confluent both recommend staying small.
- Absolute max `~31 GB` (compressed oops limit — bigger heaps actually waste memory on wider pointers).
- Everything above heap should be OS page cache.

**GC choice (2026)**:

| GC | Heap size sweet spot | JDK | Notes |
|---|---|---|---|
| G1GC | 4–16 GB | 11+ | Battle-tested. Pause target 20 ms. |
| ZGC (generational) | 16 GB+ | 21+ | Sub-ms pauses, low overhead. Netflix migrated in 2024. |
| Shenandoah | 8 GB+ | 17+ | Alternative low-pause. Rare in Kafka deployments. |
| ParallelGC | — | — | Don't. High pauses kill replication. |

**G1GC starter flags** (Kafka default since 2.6):

```bash
-Xms6g -Xmx6g
-XX:MetaspaceSize=96m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=20
-XX:InitiatingHeapOccupancyPercent=35
-XX:G1HeapRegionSize=16M
-XX:G1ReservePercent=20
-XX:+ExplicitGCInvokesConcurrent
-XX:+AlwaysPreTouch
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/kafka/heapdump.hprof
```

**Generational ZGC (JDK 21+)**:

```bash
-Xms12g -Xmx12g
-XX:+UseZGC
-XX:+ZGenerational
-XX:+AlwaysPreTouch
```

Generational ZGC (JEP 439, JDK 21) is *much* more efficient than non-generational ZGC — don't compare to old benchmarks.

**Never**:
- Run a 64 GB heap "to be safe" — it hurts, it doesn't help.
- Combine `-Xms` ≠ `-Xmx`; always pin them equal + `AlwaysPreTouch`.
- Leave `-XX:+UseCompressedOops` off on heaps ≤ 31 GB.

### 3. OS and disk tuning

#### 3.1 Filesystem

- **XFS** is the first choice (auto-tunes extent allocation, handles large files well, robust under fsync load).
- **ext4** works but needs care (`data=writeback` is risky).
- Avoid ZFS/btrfs for log dirs — CoW semantics fight the append-only log.

Mount options:

```
/dev/nvme0n1 /var/kafka/data-0 xfs defaults,noatime,nodiratime 0 0
```

- `noatime` / `nodiratime` — don't update access timestamps on reads (huge savings on hot topics).
- `nobarrier` is sometimes suggested but **don't** — modern disks handle barriers cheaply and you want crash safety.

#### 3.2 JBOD vs RAID

- **JBOD**: multiple log dirs (`log.dirs=/data-0,/data-1,/data-2,…`). Kafka places partitions across them. A disk failure takes out only the partitions on that disk; replication heals them. Preferred for cost and IO.
- **RAID 10**: simpler ops (Kafka sees one log dir), hides single-disk failures. You pay 50 % of capacity. Fine for small clusters.
- **RAID 5/6**: **never**. Write amplification and rebuild times kill fsync latency.

Kafka 4.x's KIP-858 (graceful handling of failed log dirs in KRaft) finally matches what ZooKeeper mode did — a single failed JBOD volume fails only its partitions without killing the broker. On older 3.x KRaft clusters, a volume failure could crash the whole broker; verify you're on 3.7+ before committing to JBOD.

#### 3.3 Kernel knobs

```bash
# /etc/sysctl.d/99-kafka.conf
vm.swappiness = 1                 # nearly disable swap; Kafka must not swap
vm.dirty_background_ratio = 5     # start flushing dirty pages at 5 % of RAM
vm.dirty_ratio = 60               # hard ceiling at 60 %
vm.dirty_expire_centisecs = 12000 # 120 s
vm.max_map_count = 262144         # room for many mmapped index files
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.core.netdev_max_backlog = 50000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_window_scaling = 1
fs.file-max = 2097152
```

Per-process file descriptors (`/etc/security/limits.d/kafka.conf`):

```
kafka soft nofile 1048576
kafka hard nofile 1048576
kafka soft nproc  65536
kafka hard nproc  65536
```

Partition count × 3 (segment + index + timeindex) is a lower bound on FDs. 100 k partitions = 300 k FDs, so the 1 M default matters.

THP (transparent huge pages): advice has shifted; on recent kernels `madvise` is fine. `never` is the conservative setting.

### 4. Broker tuning

#### 4.1 Threads

```properties
num.network.threads=8            # accept + read/write sockets; scale with clients
num.io.threads=16                # request handlers; ~ CPU cores
num.replica.fetchers=4           # follower fetchers per broker pair; bump on replication-heavy clusters
queued.max.requests=500
```

Rule of thumb: `num.io.threads ≈ physical cores`, `num.network.threads ≈ cores / 2`. Monitor `*AvgIdlePercent` — idle < 20 % = bump.

#### 4.2 Log / flush

```properties
log.segment.bytes=1073741824              # 1 GiB segments
log.retention.hours=168                   # 7 d
log.retention.bytes=-1                    # disable unless protecting disk
log.roll.hours=168                        # force roll even if segment isn't full
log.flush.interval.messages=9223372036854775807   # let the OS decide
log.flush.interval.ms=                     # unset — rely on page-cache flush
num.recovery.threads.per.data.dir=4       # parallel recovery after unclean shutdown
```

**Do not set `log.flush.interval.ms` to something aggressive**. Kafka's durability model relies on replication, not fsync. Forcing fsync per message collapses throughput by 10×+ with minimal durability gain.

#### 4.3 Replication

```properties
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
replica.lag.time.max.ms=30000
replica.fetch.max.bytes=1048576
replica.socket.timeout.ms=30000
```

`min.insync.replicas=2` with RF=3 and `acks=all` is the standard strong-durability setup: the broker rejects produces if fewer than 2 replicas are in-sync, so you can tolerate 1 broker loss without data loss *and* without silently accepting unreplicated writes.

### 5. KRaft operations

#### 5.1 Quorum sizing

- 3 controllers: tolerate 1 failure (quorum = 2). Default for most clusters.
- 5 controllers: tolerate 2 failures. For very large clusters or strict availability SLAs.
- 7+ controllers: diminishing returns; write latency grows because every metadata commit needs majority ack.

Separate the controller role from the broker role in production (`process.roles=controller` on dedicated nodes). Combined (`process.roles=broker,controller`) is fine for dev but couples controller latency to broker load.

#### 5.2 Static vs dynamic quorum

- **Static** (pre-3.9): `controller.quorum.voters=1@c1:9093,2@c2:9093,3@c3:9093` — every node has the whole list. Adding a controller requires rolling the whole cluster.
- **Dynamic** (KIP-853, 3.9+): `controller.quorum.bootstrap.servers=c1:9093,c2:9093,c3:9093`; actual voter set is stored in the metadata log and managed with `kafka-metadata-quorum.sh add-controller` / `remove-controller`. **Preferred for 4.x greenfield clusters** — lets you resize the quorum without downtime.

Migrating an existing static quorum to dynamic is not supported in 4.x yet — a future KIP will cover it.

#### 5.3 Common ops commands

```bash
# Quorum status
bin/kafka-metadata-quorum.sh --bootstrap-server broker:9092 describe --status
bin/kafka-metadata-quorum.sh --bootstrap-server broker:9092 describe --replication

# Add a dynamic controller
bin/kafka-metadata-quorum.sh --bootstrap-server broker:9092 \
  add-controller --config controller.properties

# Dump the metadata log for forensics
bin/kafka-dump-log.sh --cluster-metadata-decoder \
  --files /var/kafka/meta/__cluster_metadata-0/00000000000000000000.log
```

### 6. Rolling upgrades

General flow (still holds in 4.x):

1. Read the *upgrade notes* for your source/target versions. Message-format bumps and protocol bumps need ordering.
2. Set `inter.broker.protocol.version=<current>` explicitly on every broker (so they won't auto-negotiate a newer IBP before all brokers are on the new binary).
3. Rolling restart one broker at a time onto the new binary:
   - Wait for `UnderReplicatedPartitions = 0` before moving to the next.
   - In KRaft, also wait for `last-applied-record-lag-ms ≈ 0` on the restarted broker.
   - For controllers, wait for the quorum to re-form and elect a leader.
4. Once all brokers are on the new binary **and** healthy, bump `inter.broker.protocol.version` and do another rolling restart.
5. Final step: drop the explicit `inter.broker.protocol.version` line (optional, lets future upgrades auto-negotiate).

In 4.x the IBP concept is replaced by **`metadata.version`** (a finalized feature flag). Upgrade by running `kafka-features.sh --bootstrap-server … upgrade --metadata 4.0`. This is a metadata-log record, not a config — no broker restart required to activate.

Upgrade from ZK to KRaft: **ZK is removed in 4.0**, so you must migrate to KRaft on a 3.x release (3.5+ has the ZK-to-KRaft migration path) before stepping up to 4.x.

### 7. MirrorMaker 2 (MM2)

MM2 is a set of Kafka Connect source/sink connectors that replicate topics, consumer group offsets, ACLs, and configs between clusters.

Key connectors:
- **MirrorSourceConnector** — replicates records, creates `<source>.topic` on the target by default (configurable with `replication.policy.class`).
- **MirrorCheckpointConnector** — replicates consumer offsets so a consumer group can fail over.
- **MirrorHeartbeatConnector** — emits heartbeats into a `heartbeats` topic, used for lag / RPO measurement.

Topologies:

| Topology | Description | Use case |
|---|---|---|
| Active-Passive | One-way replication A → B. B is cold standby. | DR |
| Active-Active | Two-way replication A ↔ B with distinct prefixes (`A.topic`, `B.topic`). | Geo-locality |
| Hub-and-spoke | Multiple edges feed one central cluster. | IoT / edge aggregation |
| Aggregation / Fan-in | Regional clusters → global analytics cluster. | Analytics |
| Migration | A → B replication while cutting clients over. | Cluster consolidation / cloud migration |

Common pitfalls:
- **Identity replication**: If you want `topic` on the target (not `A.topic`), set `replication.policy.class=org.apache.kafka.connect.mirror.IdentityReplicationPolicy`. Required for migrations. *Never* use Identity on active-active — cycles cause loops.
- **Offset translation**: Consumer offsets don't map 1:1 across clusters (different log offsets). MM2's `RemoteClusterUtils` / Checkpoint connector translates. Validate before failover.
- **ACLs and configs**: Replicated by default only if `sync.topic.acls.enabled=true` and `sync.topic.configs.enabled=true`. Many teams forget and discover it at DR drill time.
- **Throughput**: MM2 is a Connect job; scale with `tasks.max` and dedicated Connect workers. Don't co-locate with production workloads.

Alternative: **Confluent Cluster Linking** (Confluent Platform / Cloud). Offset-preserving, no Connect workers, byte-for-byte mirror. Vendor-locked but operationally simpler than MM2.

### 8. Cruise Control

Open-source (LinkedIn). Solves "how should I rebalance partitions?" by modeling broker load, disk, CPU, network as goals and producing partition reassignments.

Core capabilities:
- **Workload monitor**: scrapes broker metrics (JMX / Prometheus) into a load model.
- **Analyzer**: optimizes against goals — `RackAwareGoal`, `ReplicaDistributionGoal`, `DiskUsageDistributionGoal`, `NetworkInboundUsageDistributionGoal`, etc.
- **Executor**: applies partition reassignments with configurable concurrency and throttling.
- **Anomaly detector**: disk failures, broker failures, metric anomalies → self-healing (move partitions off a failing broker).

KRaft support: Cruise Control 2.5.x+ integrates with the KRaft metadata quorum directly (no ZK required).

Day-2 uses:
- **Add a broker**: spin up, let CC rebalance partitions to it.
- **Decommission a broker**: `remove_broker` endpoint migrates all partitions off.
- **Re-balance after partition count increase**: new partitions initially go to one broker; CC spreads them.
- **Anomaly self-healing**: if a broker dies, CC can automatically generate a reassignment that rehomes its partitions across survivors (turn on with care — it can mask underlying problems).

### 9. Quota management (day 2)

See `13-security.md` for quota *definition*. Day-2 ops:

- Watch `kafka.server:type=ClientQuotaManager,name=throttle-time` per user/client. Non-zero for long periods means the quota is actually biting.
- Default quotas: set via `kafka-configs --alter --entity-type users --entity-default` so new users don't start unlimited.
- Controller-mutation quotas (topic/partition/ACL create rate) — essential on multi-tenant clusters to stop a runaway automation from DoS-ing the controller.

## Config reference

### Critical broker configs

| Property | Recommended | Notes |
|---|---|---|
| `num.network.threads` | `8` | Scale with client count |
| `num.io.threads` | `16` | Scale with cores |
| `num.replica.fetchers` | `4` | Bump to 8+ on large clusters |
| `queued.max.requests` | `500` | |
| `socket.send.buffer.bytes` | `1048576` | 1 MB on 10 GbE+ |
| `socket.receive.buffer.bytes` | `1048576` | |
| `socket.request.max.bytes` | `104857600` | 100 MB cap per request |
| `log.segment.bytes` | `1073741824` | 1 GiB |
| `log.retention.hours` | `168` | 7 days default |
| `log.cleaner.threads` | `4` | Compaction |
| `num.recovery.threads.per.data.dir` | `4` | Faster start after crash |
| `default.replication.factor` | `3` | |
| `min.insync.replicas` | `2` | With RF=3, `acks=all` |
| `unclean.leader.election.enable` | `false` | `true` prioritizes availability over durability |
| `replica.lag.time.max.ms` | `30000` | Followers > this fall out of ISR |
| `auto.create.topics.enable` | `false` | **Always disable in prod** |
| `delete.topic.enable` | `true` | Required for automation |
| `controller.quota.window.num` | `11` | |
| `process.roles` | `broker` or `controller` | Split in prod |
| `controller.quorum.bootstrap.servers` | `c1:9093,c2:9093,c3:9093` | Dynamic quorum (KIP-853) |
| `metadata.log.dir` | `/var/kafka/meta` | Separate disk from log dirs if possible |

### OS knobs (Linux)

| Knob | Value | Why |
|---|---|---|
| `vm.swappiness` | `1` | Kafka must not swap |
| `vm.dirty_background_ratio` | `5` | Start background flush early |
| `vm.dirty_ratio` | `60` | Hard limit before sync flush |
| `vm.max_map_count` | `262144` | Many mmapped index files |
| `fs.file-max` | `2097152` | Global FD ceiling |
| `ulimit -n` | `1048576` | Per-process FDs |
| XFS mount | `noatime,nodiratime` | Skip atime updates |
| `net.core.somaxconn` | `4096` | Accept queue |
| `net.core.rmem_max` / `wmem_max` | `16777216` | TCP buffers |

## .NET / C# snippet

Ops tuning is mostly broker-side, but one client-side configuration matters for operational resilience: handling broker throttling gracefully in a .NET producer.

```csharp
var producerConfig = new ProducerConfig
{
    BootstrapServers = "broker-1:9092,broker-2:9092,broker-3:9092",
    ClientId = "orders-producer",
    Acks = Acks.All,
    EnableIdempotence = true,
    CompressionType = CompressionType.Zstd,
    LingerMs = 20,
    BatchSize = 256 * 1024,
    MessageSendMaxRetries = int.MaxValue,   // retry until delivery.timeout.ms
    RetryBackoffMs = 100,
    // Make sure this is longer than any expected broker-side throttle delay
    // (if your quota throttles are ~5 s, leave plenty of room):
    MessageTimeoutMs = 120_000,             // librdkafka: delivery.timeout.ms
    SocketKeepaliveEnable = true,
    StatisticsIntervalMs = 5000
};
```

## Senior-level gotchas

1. **Don't give Kafka a 64 GB heap.** More heap ≠ more throughput. You starve the page cache, which is where Kafka's speed comes from.
2. **Fsync tuning is a trap.** Kafka's durability is replication, not fsync. Forcing `log.flush.interval.ms=1000` is the #1 misapplied tuning in the wild — it destroys throughput and buys you nothing that RF=3 + `min.insync.replicas=2` + `acks=all` doesn't already give.
3. **JBOD on KRaft**: require Kafka ≥ 3.7 (KIP-858) for graceful per-volume failures. Earlier KRaft versions crashed the whole broker on one failed disk.
4. **`unclean.leader.election.enable=true` silently loses data.** Only flip it if you've decided availability > durability *per topic* — use topic-level overrides, not cluster-wide.
5. **Rebalance storms during rolling restarts.** Unless `replica.lag.time.max.ms` is long enough to survive a rolling restart, followers fall out of ISR during the restart and URP spikes. 30 s default is usually fine; 60 s on slow networks.
6. **KRaft metadata lag is the new ZK session expiry.** A broker with 10+ s `last-applied-record-lag-ms` might accept stale leader info. Alert and debug (usually slow disk on `metadata.log.dir`).
7. **Controller = different workload.** Co-located controllers on busy brokers lead to metadata starvation under load. Split the roles in prod.
8. **MirrorMaker 2 offset translation is eventually consistent.** Don't assume a DR failover is zero-data-loss on offsets — verify with the Checkpoint connector's `__consumer_offsets` snapshot.
9. **Cruise Control goal ordering matters.** `RackAwareGoal` first, then distribution goals. Get it wrong and it cheerfully moves replicas off-rack to satisfy a disk-balance goal.
10. **THP `always` is fine on modern kernels** — old advice to set `never` is mostly outdated. But measure; some workloads still regress.
11. **Don't use `log.retention.bytes` as primary retention**. It's a safety net, not a policy. Use `retention.ms` as the primary and `retention.bytes` only to prevent disk-full.
12. **Upgrade ordering**: bump binaries first, then `metadata.version` (4.x) or `inter.broker.protocol.version` (3.x). Reversing this breaks the cluster.
13. **ZooKeeper is gone in 4.0.** If you're still on ZK, your migration path is `3.x ZK → 3.x KRaft-migration → 3.x KRaft → 4.x`. There is no direct ZK → 4.0 jump.
14. **Cruise Control needs a fresh metric window after adding brokers.** Running a rebalance immediately after adding a node produces bad plans because the new broker has no metric history — wait ~30 min or the `metric.sampling.interval.ms` × `num.metric.samples`.

## References

- [Apache Kafka — Monitoring & Operations (4.x)](https://kafka.apache.org/41/operations/)
- [Apache Kafka — KRaft Operations](https://kafka.apache.org/42/operations/kraft/)
- [Conduktor — JVM Tuning for Kafka: G1GC vs ZGC](https://www.conduktor.io/blog/kafka-jvm-tuning-g1gc-vs-zgc-production)
- [Confluent — Running Kafka in Production](https://docs.confluent.io/platform/current/kafka/deployment.html)
- [Instaclustr — Kafka performance: 7 critical best practices 2026](https://www.instaclustr.com/education/apache-kafka/kafka-performance-7-critical-best-practices-in-2026/)
- [Red Hat — Dynamic Kafka controller quorum (KIP-853)](https://developers.redhat.com/articles/2024/11/27/dynamic-kafka-controller-quorum)
- [Conduktor — MirrorMaker 2 for Cross-Cluster Replication](https://www.conduktor.io/glossary/kafka-mirrormaker-2-for-cross-cluster-replication)
- [Cruise Control (LinkedIn)](https://github.com/linkedin/cruise-control)
- [KIP-858: JBOD in KRaft](https://cwiki.apache.org/confluence/display/KAFKA/KIP-858%3A+Handle+JBOD+broker+disk+failure+in+KRaft)
- [KIP-853: KRaft Controller Membership Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes)
