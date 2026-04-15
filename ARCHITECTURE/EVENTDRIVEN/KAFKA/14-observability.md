# Kafka Observability — Metrics, Tracing, Tooling

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR

Kafka exposes metrics via **JMX** on brokers, controllers, producers, consumers, Kafka Streams apps, Connect workers, and MirrorMaker. A modern stack does three things:

1. **Scrape JMX** (usually via the [JMX Exporter](https://github.com/prometheus/jmx_exporter) Java agent) → **Prometheus** → **Grafana**, or via the **OpenTelemetry Collector** with the `jmx` / `kafkametrics` receiver.
2. **Track consumer lag** independently — `records-lag-max` on the consumer side is necessary but insufficient (you miss lag for crashed consumers). Use `Burrow` or `kafka-lag-exporter` that compute lag from broker offsets.
3. **Distributed tracing** across producer → topic → consumer via the **OpenTelemetry Kafka instrumentation** (W3C traceparent header propagation).

The metrics you *must* alert on are a small, stable set: `UnderReplicatedPartitions`, `OfflinePartitionsCount`, `ActiveControllerCount`, handler idle %, request latency p99, producer `record-error-rate`, consumer `records-lag-max`. Everything else is diagnostic.

In 4.x (KRaft-only), the old ZooKeeper-oriented metrics are gone. New KRaft metrics cover the controller quorum: `MetadataSnapshotLag`, `current-leader`, `current-state`, etc. KIP-848 (new consumer protocol) adds its own set of consumer-group coordinator metrics.

## Deep dive

### 1. The JMX metrics model

Every Kafka metric has a MBean `ObjectName` of the form:

```
kafka.<component>:type=<type>,name=<metric>[,<tag>=<value>,…]
```

Examples:

- `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`
- `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=orders`
- `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce`
- `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=orders-consumer,topic=orders,partition=3`

Brokers publish metric values as `Value`, `Count`, `OneMinuteRate`, `FifteenMinuteRate`, `Mean`, `50thPercentile`, `95thPercentile`, `99thPercentile`, `99.9thPercentile`. A meter with no percentiles (`Rate`) costs a single counter; a timer with percentiles is expensive — disable the high-cardinality `topic=<t>,partition=<p>` breakdowns if you have tens of thousands of partitions.

Prometheus JMX Exporter sample config (fragment):

```yaml
rules:
  - pattern: 'kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec><>OneMinuteRate'
    name: kafka_server_messagesinpersec_1m
  - pattern: 'kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value'
    name: kafka_replica_under_replicated_partitions
  - pattern: 'kafka.network<type=RequestMetrics, name=TotalTimeMs, request=(.+)><>(\d+)thPercentile'
    name: kafka_request_total_time_ms
    labels:
      request: "$1"
      quantile: "0.$2"
```

### 2. Broker / controller metrics — the critical set

| Metric | Meaning | Alert rule of thumb |
|---|---|---|
| `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions` | Number of partitions whose ISR < replicas. | `> 0` for 5 min = page |
| `kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount` | Partitions below `min.insync.replicas` — producers with `acks=all` are failing. | `> 0` = page |
| `kafka.controller:type=KafkaController,name=ActiveControllerCount` | 1 on the active controller, 0 on standbys. Cluster-wide sum must equal 1. | `sum != 1` = page |
| `kafka.controller:type=KafkaController,name=OfflinePartitionsCount` | Partitions with no leader at all. | `> 0` = page |
| `kafka.controller:type=KafkaController,name=GlobalPartitionCount` | Total partitions in the cluster | Capacity tracking |
| `kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent` | % time request-handler (I/O) threads are idle. | `< 0.2` for 10 min = broker CPU/IO saturated |
| `kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent` | % time network threads idle. | `< 0.3` = add network threads or brokers |
| `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce` | End-to-end produce latency (p50, p99, p999). | p99 > SLO = investigate |
| `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer` | Consumer fetch latency. | p99 > 500ms = investigate |
| `kafka.server:type=ReplicaFetcherManager,name=MaxLag,clientId=Replica` | Follower-to-leader lag in messages. | Sustained growth = follower can't keep up |
| `kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs` | fsync latency. | p99 > 1 s indicates disk problems |
| `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec` / `BytesOutPerSec` | Broker throughput. | Capacity trend, not an alert |
| `kafka.server:type=SessionExpireListener,name=ZooKeeperExpiresPerSec` | Only legacy; in KRaft you track `MetadataLoadErrorCount` instead. | |

KRaft-specific controller metrics (4.x):

| Metric | Meaning |
|---|---|
| `kafka.controller:type=KafkaController,name=MetadataLastAppliedRecordOffset` | Offset of last record applied by the active controller. |
| `kafka.controller:type=KafkaController,name=MetadataLastCommittedRecordOffset` | Offset committed by the quorum. |
| `kafka.server:type=broker-metadata-metrics,name=last-applied-record-lag-ms` | Broker's lag behind the controller metadata stream. Critical — if brokers diverge they make incorrect decisions. |
| `kafka.server:type=RaftManagerMetrics,name=current-state` | `leader` / `candidate` / `follower` / `observer`. |
| `kafka.server:type=RaftManagerMetrics,name=current-leader` | Node ID of the active controller. |
| `kafka.server:type=RaftManagerMetrics,name=commit-latency-ms` | Latency of Raft commit (quorum roundtrip). |

Collection cadence: high-priority (URP, OfflinePartitions, ActiveControllerCount, handler idle%) every **10–30 s**. Everything else every **1 min**.

### 3. Producer metrics

From `kafka.producer:type=producer-metrics,client-id=<id>`:

| Metric | Meaning | What to watch |
|---|---|---|
| `record-send-rate` | Records/sec sent | Throughput trend |
| `record-send-total` | Cumulative count | |
| `record-error-rate` | Failed sends/sec | > 0 sustained = alert |
| `record-retry-rate` | Retries/sec | Spikes indicate broker instability |
| `request-latency-avg` / `request-latency-max` | Time from send to ack | Watch p99 vs SLO |
| `batch-size-avg` | Average batch size bytes | Too small = `linger.ms` too low, hurting throughput |
| `records-per-request-avg` | Records batched per request | Larger is more efficient |
| `compression-rate-avg` | Size after compression / before | Sanity-check compression codec |
| `buffer-available-bytes` | Free bytes in the producer buffer | Near 0 → producer is back-pressuring |
| `buffer-total-bytes` | `buffer.memory` | |
| `bufferpool-wait-ratio` | Fraction of time `send()` blocks on the buffer | > 0.2 = allocate more `buffer.memory` or slow the app |
| `record-queue-time-avg` | Time a record waits in the accumulator before being sent | High = linger/batch pressure |
| `requests-in-flight` | Concurrent in-flight requests | Cap via `max.in.flight.requests.per.connection` |

Per-topic variants exist under `kafka.producer:type=producer-topic-metrics,client-id=<id>,topic=<t>` — helpful but cardinality-heavy.

### 4. Consumer metrics

From `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<id>`:

| Metric | Meaning | What to watch |
|---|---|---|
| `records-consumed-rate` | Records/sec consumed | Throughput |
| `bytes-consumed-rate` | Bytes/sec | |
| `records-lag-max` | Max lag across assigned partitions (records behind log-end) | **Primary SLO metric** |
| `records-lag-avg` | Average lag | |
| `records-lead-min` | Minimum lead (distance from log start) — near 0 = about to hit retention | |
| `fetch-latency-avg` / `fetch-latency-max` | Fetch request RTT | |
| `fetch-rate` | Fetches/sec | |
| `fetch-size-avg` | Bytes per fetch | Compare vs `fetch.max.bytes` |

From `kafka.consumer:type=consumer-coordinator-metrics,client-id=<id>`:

| Metric | Meaning |
|---|---|
| `commit-rate` / `commit-latency-avg` | How often / how fast we commit offsets |
| `rebalance-rate-per-hour` | Frequent rebalances = instability or bad `session.timeout.ms` / `max.poll.interval.ms` |
| `rebalance-latency-avg` | How long group rebalances take. KIP-848 (new consumer group protocol) reduces this dramatically by moving assignment to the coordinator. |
| `failed-rebalance-rate-per-hour` | Should be 0 |
| `heartbeat-rate` | Heartbeats/sec |
| `last-heartbeat-seconds-ago` | Liveness check |

KIP-848 adds `kafka.consumer:type=consumer-group-metrics` on both the broker (coordinator) and the client; new metrics include `assignment-count`, `target-assignment-epoch`, and `heartbeat-response-time-max`. Worth adding to dashboards if you're on the new protocol.

**`records-lag-max` is not enough** — a consumer that crashed reports nothing. Always pair it with broker-side lag computed from `__consumer_offsets` vs `LogEndOffset` (Burrow, kafka-lag-exporter).

### 5. Kafka Streams & Connect metrics

Streams adds `kafka.streams:type=stream-thread-metrics` with `process-latency-avg`, `poll-latency-avg`, `punctuate-rate`, `commit-latency-avg`, plus per-store metrics (`put-latency-avg`, `get-latency-avg`, `restore-records-rate`).

Connect adds `kafka.connect:type=connect-worker-metrics,connector=<c>` with `connector-startup-success-total`, `task-failure-rate`, and per-task `source-record-poll-rate` / `sink-record-send-rate`.

### 6. Tooling

| Tool | What it does | Notes |
|---|---|---|
| **Prometheus + Grafana + JMX Exporter** | De-facto OSS stack. Exporter as Java agent on each JVM; Prometheus scrapes `/metrics`. | Be careful with MBean pattern explosion — filter aggressively. |
| **OpenTelemetry Collector** | Scrapes JMX (`jmxreceiver` + Kafka target) **or** uses the purpose-built `kafkametricsreceiver` (talks directly to the broker admin client). Pushes to OTLP-compatible backend (Tempo, Jaeger, Datadog, New Relic, Honeycomb, etc.). | Preferred for greenfield — vendor-neutral, also handles traces. |
| **Burrow** (LinkedIn) | Consumer lag monitoring service. Infers status (OK / WARN / ERR / STALL / REWIND / STOP) per group from offset trajectories. Stateless, polls the cluster. | Great for "is my group healthy?" alerts without relying on the consumer itself. |
| **kafka-lag-exporter** (Lightbend/maintained community fork) | Prometheus exporter that computes lag in seconds (not just records) by interpolating message timestamps. | Lag-in-time is much more actionable than lag-in-records (a 100k-record lag means different things at 10 msg/s vs 10k msg/s). |
| **Confluent Control Center** | Enterprise UI: cluster health, consumer lag, stream lineage, broker metrics, RBAC audit log. Part of Confluent Platform license. | Heavy; needs its own Kafka cluster to store metric topics. |
| **AKHQ** (OSS) | Web UI: browse topics, search messages, view consumer groups, manage ACLs, view schema registry. Read-focused, lighter than Control Center. | Excellent dev/ops tool; not a metrics platform. |
| **Conduktor Platform / Gateway** | Commercial. UI + governance + proxy with runtime policies (data masking, rate limiting, audit). Includes monitoring dashboards and data-catalog features. | Broader than pure observability; evaluate cost vs AKHQ + Grafana. |
| **Kafbat UI** | Active fork of what used to be `provectus/kafka-ui` (unmaintained as of 2024). OSS Web UI, similar scope to AKHQ. | Preferred OSS UI by many teams in 2025+. |
| **Cruise Control UI** | Rebalancing recommendations + anomaly detection. Observability-adjacent — see ops file. | |

### 7. End-to-end distributed tracing

Kafka breaks the synchronous call graph, so tracing requires explicit header propagation.

- **OpenTelemetry Kafka instrumentation** (Java, .NET, Python, Go, Node) injects W3C `traceparent` and `tracestate` into Kafka record headers on send, and extracts them on receive. A span `<topic> publish` becomes the parent of `<topic> process` in the consumer.
- In Java, the OTel Java agent does this automatically — zero code changes for Kafka clients & Streams.
- In .NET with **Confluent.Kafka**, you either:
  - Use `OpenTelemetry.Instrumentation.ConfluentKafka` (auto), or
  - Manually inject using `Activity.Current?.Id` into a `Header` on produce and `ActivityContext.Parse` on consume.
- Consumer groups with high parallelism generate huge span volumes — sample by trace ID, not by span.

Gotcha: Kafka Streams' internal topics (repartition, changelog) also propagate headers, so one logical transaction can produce dozens of spans. Use a tail-based sampler (Grafana Tempo, Jaeger, Honeycomb) that keeps slow/error traces.

### 8. Dashboards to build

**Cluster health (one dashboard, rolls up all brokers)**
- URP, OfflinePartitions, ActiveControllerCount (big stat panels)
- Bytes in/out per broker (stacked)
- Request handler / network processor idle % per broker
- p99 Produce / FetchConsumer / FetchFollower latency
- ISR shrink/expand rate
- Log flush latency p99
- KRaft: metadata lag per broker, commit latency

**Per-topic deep dive**
- Messages in, bytes in/out per sec
- Partition count, leader distribution across brokers
- Min/avg/max partition size
- Producer `record-error-rate`, `record-retry-rate`
- Consumer lag (records + time) by group

**Per consumer group**
- Lag per partition (heatmap)
- Commit rate, commit latency
- Rebalance rate, rebalance duration
- Assigned partitions per member (imbalance?)

**Producer SLO dashboard**
- `record-send-rate`, `record-error-rate`, `request-latency-avg` p99
- `bufferpool-wait-ratio`, `buffer-available-bytes`
- `batch-size-avg`, `records-per-request-avg`

**Capacity / trend**
- Total partitions, partitions per broker (watch for > 4 k per broker)
- Disk usage per log directory (and per topic if you have budget for cardinality)
- Retention vs disk free — project the day you run out

## Config reference

### Exposing JMX on the broker JVM

```bash
# In kafka-server-start.sh / systemd unit:
export KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
 -Dcom.sun.management.jmxremote.authenticate=true \
 -Dcom.sun.management.jmxremote.ssl=true \
 -Dcom.sun.management.jmxremote.password.file=/etc/kafka/jmx.password \
 -Dcom.sun.management.jmxremote.access.file=/etc/kafka/jmx.access \
 -Djava.rmi.server.hostname=$(hostname -f)"

# Attach the Prometheus JMX exporter as a javaagent:
export KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=7071:/etc/kafka/jmx-exporter.yaml $KAFKA_OPTS"
```

### Broker configs relevant to observability

| Property | Typical value | Notes |
|---|---|---|
| `metric.reporters` | `org.apache.kafka.common.metrics.JmxReporter` | Default; add custom reporters here |
| `metrics.num.samples` | `2` | Sampling buckets for rates |
| `metrics.sample.window.ms` | `30000` | |
| `auto.include.jmx.reporter` | `true` | Default |
| `kafka.metrics.polling.interval.secs` | `10` | |
| `log.cleaner.enable` | `true` | Compaction metrics only meaningful if on |

### Producer / consumer observability knobs

| Property | Default | Notes |
|---|---|---|
| `client.id` | `""` | **Always set it.** Without it, all your metrics collapse under `client-id=consumer-1` etc. |
| `metrics.recording.level` | `INFO` | Set to `DEBUG` to get per-partition producer/consumer metrics (expensive). |
| `statistics.interval.ms` (librdkafka / Confluent.Kafka) | `0` (off) | Set to `5000` to get a JSON stats blob every 5s via the statistics handler. |

## .NET / C# snippet (Confluent.Kafka)

Confluent.Kafka wraps librdkafka. It does **not** expose JMX metrics — you get a JSON statistics callback instead. Forward it to your metrics backend.

```csharp
using Confluent.Kafka;
using System.Text.Json;
using System.Diagnostics.Metrics;

var meter = new Meter("MyApp.Kafka", "1.0.0");
var sendRate = meter.CreateObservableGauge<double>("kafka_producer_record_send_rate", () => _lastSendRate);
// … plus other gauges …

double _lastSendRate, _lastLatencyAvg, _lastBufferAvail;

var config = new ProducerConfig
{
    BootstrapServers = "broker:9092",
    ClientId = "orders-producer-1",           // critical for dashboards
    StatisticsIntervalMs = 5000,              // emit stats every 5s
    Acks = Acks.All,
    EnableIdempotence = true,
    CompressionType = CompressionType.Zstd,
    LingerMs = 20,
    BatchSize = 256 * 1024
};

using var producer = new ProducerBuilder<string, string>(config)
    .SetStatisticsHandler((_, json) =>
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        // librdkafka top-level stats: https://github.com/confluentinc/librdkafka/blob/master/STATISTICS.md
        _lastSendRate     = root.GetProperty("txmsgs").GetDouble();       // cumulative; compute delta
        _lastLatencyAvg   = root.GetProperty("brokers")
                                .EnumerateObject().First().Value
                                .GetProperty("rtt").GetProperty("avg").GetDouble() / 1000.0;
        _lastBufferAvail  = root.GetProperty("msg_cnt").GetDouble();
    })
    .SetErrorHandler((_, e) =>
    {
        // Non-fatal errors surface here — correlate with broker-side URP etc.
        Log.Warning("Kafka producer error: {Reason} (IsFatal={Fatal})", e.Reason, e.IsFatal);
    })
    .Build();
```

Then export `_lastSendRate`, `_lastLatencyAvg`, etc. via the OTel .NET SDK / Prometheus exporter. For distributed tracing, prefer `OpenTelemetry.Instrumentation.ConfluentKafka` — it registers `Activity` sources for `Confluent.Kafka.Producer.Send` and `Confluent.Kafka.Consumer.Consume` and does header propagation automatically.

## Senior-level gotchas

1. **Consumer lag blind spot.** `records-lag-max` from the consumer itself goes dark when the consumer crashes. Always monitor lag from the *broker* side too (Burrow / kafka-lag-exporter).
2. **Lag in records lies.** A 100 k-record lag means 1 s at 100 k msg/s and 2 hours at 14 msg/s. Prefer *lag-in-time* where possible.
3. **Cardinality blowup.** Enabling `metrics.recording.level=DEBUG` or scraping per-partition metrics on a 20 k-partition cluster adds millions of time series. Use allowlists in the JMX exporter.
4. **`ActiveControllerCount` must sum to exactly 1.** Two controllers = split brain (rare but catastrophic); zero = no-one is making metadata decisions. Alert on `sum != 1`, not per-broker.
5. **KRaft metadata lag is the new ZK session expiry.** Brokers with stale metadata make wrong decisions (incorrect leader info, outdated ACLs). Alert on `last-applied-record-lag-ms > 5000`.
6. **`RequestHandlerAvgIdlePercent < 0.2` is not always CPU.** It's I/O + CPU + lock contention; rule out disk saturation (`iostat -x 1`) before adding threads.
7. **`min.insync.replicas` breaches are silent to dashboards that only track URP.** `UnderMinIsrPartitionCount` is the correct signal for producer-facing availability.
8. **Rebalance storms are a symptom, not a cause.** Track `rebalance-rate-per-hour`, then chase: `max.poll.interval.ms` too low, slow processing, JVM GC pauses, or a consumer that keeps OOMing.
9. **Percentile clamping.** Kafka's built-in percentile reservoirs are `ExponentiallyDecayingSample`. For short bursts they can under-report; p99 averaged over a 1-min window is smoother than p99 sampled instantaneously.
10. **Don't trace through `__consumer_offsets` or `__transaction_state`.** Some auto-instrumentations try; these are internal topics and will spam your tracing backend.
11. **Broker-side p99 vs client-side p99**. Broker `Produce TotalTimeMs` excludes network RTT to the client. A client seeing 200 ms while the broker reports 20 ms means network or client-side buffering is the culprit.
12. **Confluent.Kafka stats JSON is cumulative**. To get rates, compute deltas between statistics callbacks — don't ship cumulative counts as gauges.
13. **KIP-848 metrics are different**. If you migrate consumers to the new protocol, your old `rebalance-latency-avg` dashboard panel goes empty — update dashboards before cutover.

## References

- [Apache Kafka Monitoring docs (4.x)](https://kafka.apache.org/41/operations/monitoring/)
- [Confluent — Monitoring Kafka with JMX](https://docs.confluent.io/platform/current/kafka/monitoring.html)
- [Instaclustr — Kafka monitoring: Key metrics and 5 tools to know in 2025](https://www.instaclustr.com/education/apache-kafka/kafka-monitoring-key-metrics-and-5-tools-to-know-in-2025/)
- [OpenTelemetry — Instrumenting Apache Kafka clients](https://opentelemetry.io/blog/2022/instrument-kafka-clients/)
- [Uptrace — Kafka Monitoring with OpenTelemetry Collector](https://uptrace.dev/guides/opentelemetry-kafka)
- [Conduktor — Kafka Cluster Monitoring and Metrics](https://www.conduktor.io/glossary/kafka-cluster-monitoring-and-metrics)
- [SigNoz — Kafka metrics guide](https://signoz.io/guides/kafka-metrics/)
- [Burrow — consumer lag monitoring](https://github.com/linkedin/Burrow)
- [kafka-lag-exporter](https://github.com/seglo/kafka-lag-exporter)
- [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter)
