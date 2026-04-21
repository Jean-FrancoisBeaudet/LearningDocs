# Kafka Connect

> Version baseline: Apache Kafka 4.x (KRaft mode), Kafka Connect 4.x,
> Debezium 3.3.x. Last researched: 2026-04.

## TL;DR

**Kafka Connect** is the integration framework for Kafka — a separate JVM
runtime that runs pre-built **connectors** to move data between Kafka and
external systems. It does three things your own consumer/producer would
have to reinvent:

1. **Offset management** — source connectors checkpoint "how far I've read
   from MySQL", sink connectors checkpoint "how far I've written to S3".
2. **Scale-out & failover** — run in **distributed mode**, tasks are
   rebalanced across worker nodes when one dies.
3. **Schema + transform pipeline** — pluggable **Converters** (Avro /
   Protobuf / JSON Schema / plain JSON / string / byte[]) that integrate
   with Schema Registry, and **Single Message Transforms (SMTs)** for
   lightweight per-record mutation (rename field, mask PII, route to
   different topic).

Two run modes:

- **Standalone** — single process, offsets stored in a local file. Dev-box
  only. No HA.
- **Distributed** — cluster of workers with the same `group.id`. State
  lives in three internal compacted Kafka topics: `connect-offsets`,
  `connect-configs`, `connect-status`. Workers rebalance via the same
  group-coordinator machinery as consumers.

Two connector kinds:

- **Source** — external system -> Kafka (e.g., Debezium MySQL CDC).
- **Sink**   — Kafka -> external system (e.g., S3, Elasticsearch, JDBC).

The REST API (`:8083`) is how you manage everything: POST a JSON config to
`/connectors`, GET `/connectors/{name}/status`, POST `/restart`, etc.

---

## Deep dive

### 1. Distributed mode architecture

```
       +---------+     +---------+     +---------+
       | Worker1 |     | Worker2 |     | Worker3 |    <- same group.id
       +---------+     +---------+     +---------+
            \              |                /
             \             |               /
              v            v              v
     [connect-offsets] [connect-configs] [connect-status]   <- compacted topics
                            |
                            v
                     Kafka cluster
```

- All workers that share `group.id` form one Connect cluster.
- Connector configurations are POSTed to **any** worker; the worker writes
  the config to `connect-configs`, and the group-coordinator-elected
  **leader worker** distributes tasks across the group.
- Each connector has one **Connector** instance (coordination / config)
  and N **Tasks** (the actual data movement). `tasks.max` is per-connector.
- On worker failure, the group rebalances and surviving workers pick up
  orphaned tasks. Incremental Cooperative Rebalancing (KIP-415, the
  default since 2.3) avoids stop-the-world pauses.

### 2. Internal topics

All three are **compacted**. Pre-create them with `replication.factor=3`,
`cleanup.policy=compact`, and sensible partition counts:

| Topic | Purpose | Partitions (typical) |
|---|---|---|
| `connect-offsets` | Source-connector offsets (e.g., binlog position, table+PK cursor) | 25 |
| `connect-configs` | Connector + task configurations | **1** (must be 1 so all workers see same order) |
| `connect-status` | Per-task status (RUNNING, FAILED, PAUSED) + error messages | 5 |

Names are worker-level configs
(`offset.storage.topic`, `config.storage.topic`, `status.storage.topic`).
Sink connectors use the **consumer group** offset mechanism (they're just
Kafka consumers under the hood), not `connect-offsets`.

### 3. Source vs Sink connectors

#### Source

- Polls the external system (or subscribes to its change feed — e.g.,
  Debezium tails the MySQL binlog).
- Emits `SourceRecord`s with a source partition + source offset (opaque
  JSON — connector-defined).
- Connect periodically commits the offset to `connect-offsets` after
  downstream Kafka acks. On restart, the connector resumes from the
  committed offset.

#### Sink

- Subscribes to one or more topics (`topics=...` or `topics.regex=...`).
- Receives `SinkRecord`s, writes them to the external system.
- Commits consumer offsets only **after** the external write succeeds.
  This gives at-least-once semantics out of the box; exactly-once depends
  on the connector using upserts (idempotent writes) or deduplicating
  downstream.

### 4. Converters

Converter = "bytes <-> in-memory record". Configured per **worker** (defaults)
and optionally overridden per **connector**.

```json
"key.converter":   "io.confluent.connect.avro.AvroConverter",
"value.converter": "io.confluent.connect.avro.AvroConverter",
"value.converter.schema.registry.url": "http://sr:8081"
```

Common converters:

| Converter | Notes |
|---|---|
| `AvroConverter` | Confluent, uses Schema Registry. Wire format: magic byte + 4-byte schema id + Avro bytes. |
| `ProtobufConverter` | Same wire format, Protobuf payload. |
| `JsonSchemaConverter` | Same wire format, JSON Schema payload. |
| `JsonConverter` | Plain JSON. Set `schemas.enable=false` for schemaless, `true` for embedded `{ "schema": {...}, "payload": {...} }`. |
| `StringConverter` | Raw UTF-8 string. |
| `ByteArrayConverter` | Opaque bytes — useful as a bypass when the converter would otherwise corrupt binary data. |

**Key insight**: the key and value converters are independent. A common
pattern is `StringConverter` for keys (primary key as text) and
`AvroConverter` for values.

### 5. Single Message Transforms (SMTs)

SMTs are **stateless, per-record** functions chained after the converter
(sources) or before the sink write (sinks). They are intentionally limited
— no joins, no windowing, no state. For real stream processing use Kafka
Streams / Flink / ksqlDB.

Built-in SMTs include `InsertField`, `ReplaceField`, `MaskField`,
`ValueToKey`, `ExtractField`, `HoistField`, `TimestampConverter`,
`RegexRouter`, `TimestampRouter`, `Flatten`, `Cast`, `Filter`,
`HeaderFrom`, `DropHeaders`, `SetSchemaMetadata`.

Debezium's `io.debezium.transforms.ExtractNewRecordState` ("Unwrap") is
arguably the most important SMT in the real world — it turns Debezium's
nested `{before, after, op, source, ...}` envelope into a flat record that
downstream sinks (Elasticsearch, JDBC) can consume directly.

```json
"transforms": "unwrap,route",
"transforms.unwrap.type":  "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.drop.tombstones":  "false",
"transforms.unwrap.delete.handling.mode": "rewrite",
"transforms.route.type":   "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.route.regex":  "(.*)",
"transforms.route.replacement": "cdc.$1"
```

### 6. Dead Letter Queue (DLQ) and error handling

By default, a single bad record kills the task. Two configs change that:

```json
"errors.tolerance": "all",
"errors.deadletterqueue.topic.name": "connect-dlq-myconnector",
"errors.deadletterqueue.topic.replication.factor": "3",
"errors.deadletterqueue.context.headers.enable": "true",

"errors.log.enable": "true",
"errors.log.include.messages": "true",

"errors.retry.timeout":     "600000",
"errors.retry.delay.max.ms":"60000"
```

Behavior:

- `errors.tolerance=none` (default) -> any error fails the task.
- `errors.tolerance=all` -> converter + SMT errors skipped (logged, and
  written to DLQ if configured). **Does NOT skip errors inside the
  connector itself** (e.g., JDBC sink can't insert row) unless the
  connector explicitly opts in.
- DLQ is populated with the **raw bytes** of the failed record (because
  converter may have been what failed). `context.headers.enable=true`
  attaches headers describing which stage failed and the exception.
- Retry configs apply to **connector** operations, not converter/SMT.

DLQ is an at-least-once channel; expect duplicates on retry. Reprocessing
is a manual pipeline: read from DLQ, fix, replay to the original topic.

### 7. Common connectors

| Connector | Type | Notes |
|---|---|---|
| **Debezium MySQL / Postgres / SQL Server / Oracle / MongoDB** | Source | CDC via binlog / WAL / CDC tables. Emits `before`/`after`/`op`. Debezium 3.3 adds EOS for all core connectors and Kafka 4.1 support. |
| **Confluent JDBC source** | Source | Polling-based (`mode=incrementing` or `timestamp+incrementing`). Not CDC — misses deletes. Use Debezium instead for real CDC. |
| **Confluent JDBC sink** | Sink | Writes to any JDBC target. Supports upsert via `insert.mode=upsert` + `pk.mode=record_key`. |
| **Debezium JDBC sink** | Sink | Newer (Debezium 2.5+). Auto-retries transient SQL exceptions, native Debezium envelope support. |
| **Confluent S3 sink** | Sink | Writes to S3 in Avro/Parquet/JSON. Partitioners: default (Kafka partition), TimeBasedPartitioner (event time), FieldPartitioner. |
| **Confluent Elasticsearch sink** | Sink | Indexes to ES. Combine with Debezium + Unwrap SMT for CDC -> search index. |
| **Confluent HTTP sink** | Sink | Generic HTTP POST. Useful for pushing events to webhooks. |
| **MirrorMaker 2** | Source + Sink | Cluster-to-cluster replication; runs on Connect runtime. |

### 8. Exactly-once for Connect (KIP-618)

Since Kafka 3.3, **source** connectors can opt into exactly-once by setting
`exactly.once.source.support=enabled` on the worker + `exactly.once=required`
on the connector. The Connect framework wraps producer writes + offset
commits in a Kafka transaction. Requires:

- `isolation.level=read_committed` downstream.
- Connector implementation declaring EOS support (most Debezium 3.3
  connectors do; some third-party do not).
- `transaction.state.log.replication.factor` >= 3 on the broker.

Sink EOS relies on the external system's idempotency (upserts) — Connect
cannot transactionally commit to Postgres + Kafka simultaneously.

### 9. REST API (cheat sheet)

```
GET    /connectors                          # list
POST   /connectors                          # create  (body: full config)
GET    /connectors/{name}                   # read config
PUT    /connectors/{name}/config            # idempotent create/update
GET    /connectors/{name}/status            # live status (RUNNING|FAILED|PAUSED)
GET    /connectors/{name}/tasks             # task configs
POST   /connectors/{name}/restart           # restart connector (and optionally tasks: ?includeTasks=true&onlyFailed=true)
POST   /connectors/{name}/pause
POST   /connectors/{name}/resume
DELETE /connectors/{name}
GET    /connector-plugins                   # installed plugins
PUT    /connector-plugins/{class}/config/validate   # dry-run a config
```

### 10. Running Connect in production

- Run **3+ workers** behind a load balancer (REST is stateless).
- `plugin.path` isolates connector JARs — mandatory in production to avoid
  classpath hell.
- Monitor JMX: `connector-metrics`, `source-task-metrics`,
  `sink-task-metrics`, `connect-worker-rebalance-metrics`.
- Back up `connect-configs` (or GitOps the configs) — losing it loses
  every connector definition.

---

## Config reference

### Worker config (distributed)

| Config | Example | Meaning |
|---|---|---|
| `bootstrap.servers` | `broker:9092` | Kafka |
| `group.id` | `connect-cluster-prod` | Connect-cluster identity |
| `key.converter` / `value.converter` | `...AvroConverter` | Default converters |
| `offset.storage.topic` | `connect-offsets` | Compacted, 25 partitions |
| `config.storage.topic` | `connect-configs` | Compacted, **1** partition |
| `status.storage.topic` | `connect-status` | Compacted, 5 partitions |
| `offset.flush.interval.ms` | `10000` | Source offset commit cadence |
| `plugin.path` | `/opt/kafka/plugins` | Connector JAR dirs |
| `rest.port` | `8083` | REST |
| `exactly.once.source.support` | `enabled` | Enable EOS machinery |

### Per-connector common

| Config | Meaning |
|---|---|
| `name` | Unique connector name |
| `connector.class` | FQCN |
| `tasks.max` | Parallelism upper bound (connector may create fewer) |
| `topics` / `topics.regex` | Sink subscription |
| `key.converter` / `value.converter` | Override worker defaults |
| `transforms` | Comma-list of SMT names |
| `errors.tolerance` | `none` \| `all` |
| `errors.deadletterqueue.topic.name` | DLQ (sinks only) |
| `errors.retry.timeout` | How long to retry connector operations |

---

## Connector config examples

### Debezium MySQL source -> CDC with Unwrap SMT

```json
{
  "name": "mysql-orders-cdc",
  "config": {
    "connector.class":            "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max":                  "1",
    "database.hostname":          "mysql",
    "database.port":              "3306",
    "database.user":              "debezium",
    "database.password":          "${file:/etc/secrets:mysql_pw}",
    "database.server.id":         "184054",
    "topic.prefix":               "cdc",
    "database.include.list":      "shop",
    "table.include.list":         "shop.orders,shop.customers",
    "schema.history.internal.kafka.bootstrap.servers": "broker:9092",
    "schema.history.internal.kafka.topic":             "schema-history.shop",

    "key.converter":              "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url":   "http://sr:8081",
    "value.converter":            "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://sr:8081",

    "transforms":                 "unwrap",
    "transforms.unwrap.type":     "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones":       "false",
    "transforms.unwrap.delete.handling.mode":  "rewrite",

    "errors.tolerance":                     "all",
    "errors.deadletterqueue.topic.name":    "cdc-dlq",
    "errors.deadletterqueue.context.headers.enable": "true",

    "exactly.once.support":       "required"
  }
}
```

### S3 sink with time-based partitioning

```json
{
  "name": "s3-orders-archive",
  "config": {
    "connector.class":              "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max":                    "4",
    "topics":                       "cdc.shop.orders",
    "s3.bucket.name":               "my-lake",
    "s3.region":                    "us-east-1",
    "flush.size":                   "10000",
    "rotate.interval.ms":           "900000",
    "format.class":                 "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "storage.class":                "io.confluent.connect.s3.storage.S3Storage",
    "partitioner.class":            "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "partition.duration.ms":        "3600000",
    "path.format":                  "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
    "locale":                       "en-US",
    "timezone":                     "UTC",
    "timestamp.extractor":          "RecordField",
    "timestamp.field":              "event_ts",

    "key.converter":                "org.apache.kafka.connect.storage.StringConverter",
    "value.converter":              "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://sr:8081",

    "errors.tolerance":             "all",
    "errors.deadletterqueue.topic.name": "s3-orders-archive-dlq"
  }
}
```

### Elasticsearch sink (CDC into a search index)

```json
{
  "name": "es-orders-index",
  "config": {
    "connector.class":               "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max":                     "2",
    "topics":                        "cdc.shop.orders",
    "connection.url":                "https://es:9200",
    "connection.username":           "elastic",
    "connection.password":           "${file:/etc/secrets:es_pw}",
    "key.ignore":                    "false",
    "schema.ignore":                 "true",
    "behavior.on.null.values":       "delete",
    "write.method":                  "upsert",

    "transforms":                    "unwrap,key",
    "transforms.unwrap.type":        "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.delete.handling.mode": "none",
    "transforms.key.type":           "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.key.field":          "id",

    "errors.tolerance":              "all",
    "errors.deadletterqueue.topic.name": "es-orders-dlq",
    "errors.retry.timeout":          "600000"
  }
}
```

### JDBC sink (upsert)

```json
{
  "name": "jdbc-sink-postgres",
  "config": {
    "connector.class":     "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max":           "4",
    "topics":              "orders-enriched",
    "connection.url":      "jdbc:postgresql://pg:5432/analytics",
    "connection.user":     "kafka_sink",
    "connection.password": "${file:/etc/secrets:pg_pw}",
    "insert.mode":         "upsert",
    "pk.mode":             "record_key",
    "pk.fields":           "order_id",
    "auto.create":         "true",
    "auto.evolve":         "true",
    "delete.enabled":      "true",

    "errors.tolerance":                     "all",
    "errors.deadletterqueue.topic.name":    "jdbc-dlq",
    "errors.retry.timeout":                 "600000"
  }
}
```

---

## .NET / C# note

Connector configs are **JSON POSTed to the Connect REST API** — language-
agnostic. From .NET you just need an HTTP client. There is no
`Confluent.Connect` SDK because there doesn't need to be one. Tiny helper:

```csharp
using var http = new HttpClient { BaseAddress = new Uri("http://connect:8083/") };

// Idempotent create/update via PUT /connectors/{name}/config
var connectorName = "mysql-orders-cdc";
var connectorJson = await File.ReadAllTextAsync("mysql-orders-cdc.config.json");

var resp = await http.PutAsync(
    $"connectors/{connectorName}/config",
    new StringContent(connectorJson, System.Text.Encoding.UTF8, "application/json"));
resp.EnsureSuccessStatusCode();

// Check status
var status = await http.GetStringAsync($"connectors/{connectorName}/status");
Console.WriteLine(status);
```

For a richer experience: **`KafkaFlow.Connect`** / **`Aiven.Kafka.Connect.Client`**
style wrappers exist, but rolling your own 50-line REST client is usually
cleaner.

---

## Senior-level gotchas

1. **`connect-configs` MUST be 1 partition.** If you accidentally create it
   with more, Connect will not work correctly — configs must be totally
   ordered. Pre-create all three internal topics explicitly.

2. **`plugin.path` is not a classpath.** Don't put connector JARs on
   `CLASSPATH` — Connect uses isolated classloaders per plugin directory.
   Mixing them causes NoSuchMethodError when two connectors ship
   conflicting dependency versions (Jackson, Guava, etc.).

3. **Standalone mode is never production.** No HA, offsets in a local file.
   Use it for dev only.

4. **Sink consumer groups are named `connect-{connector-name}`.** This is
   what you reset / monitor lag on. Two sinks with the same name share
   offsets — usually a bug.

5. **Debezium source offsets are in `connect-offsets`, not consumer
   offsets.** If you need to "rewind" a Debezium connector, you must
   surgically edit `connect-offsets` (KIP-875 added a REST API for this
   in Kafka 3.6+: `PATCH /connectors/{name}/offsets` and
   `DELETE /connectors/{name}/offsets`). Never hand-edit the compacted
   topic directly.

6. **SMTs are stateless.** They cannot look up another topic. For joins /
   enrichment you need Kafka Streams, ksqlDB, or a custom Processor API
   app between source and sink.

7. **`errors.tolerance=all` masks real problems.** It silently skips
   converter/SMT errors. Without a DLQ + alerting, bad records disappear
   silently and you discover the data loss months later during an audit.
   Always pair with `errors.deadletterqueue.topic.name` and alert on DLQ
   non-zero volume.

8. **JDBC source is not CDC.** It polls `SELECT * WHERE updated_at > ?`
   and therefore cannot see deletes, cannot see rows whose PK was reused,
   and creates read load on the source DB. Debezium reads the log instead
   — dramatically better. Only use JDBC source for simple append-only
   tables.

9. **`tasks.max` is an upper bound, not a target.** The connector decides
   how many tasks it actually creates. Debezium MySQL always creates 1
   task per connector (binlog is single-threaded). Setting `tasks.max=8`
   on it buys you nothing.

10. **Schema evolution bites at the converter.** If `schemas.enable=true`
    on `JsonConverter`, producers must send the `{schema, payload}`
    envelope or the sink task fails. Most teams should use
    `AvroConverter` + Schema Registry for anything non-trivial.

11. **EOS requires connector opt-in.** `exactly.once.source.support=enabled`
    on the worker is necessary but not sufficient — the connector class
    must declare `SourceConnector.exactlyOnceSupport()=SUPPORTED`.
    Check connector docs before relying on it.

12. **Incremental cooperative rebalancing reduces but does not eliminate
    pauses.** A worker losing its lease still causes a short stall; scale
    horizontally so no single worker carries too many tasks.

13. **Secret management.** `${file:...}` and `${vault:...}` config
    providers are how you keep passwords out of the REST API response —
    never embed plaintext secrets; the REST API happily returns them.

14. **Schema history topic (Debezium) is NOT compacted.** It's an
    append-only log of DDL changes, and Debezium reads it from the
    beginning on startup. It must have `cleanup.policy=delete` with
    effectively infinite retention, and you must not reset it without
    re-snapshotting.

---

## References

- [Apache Kafka — Connect User Guide](https://kafka.apache.org/41/kafka-connect/user-guide/)
- [Confluent — Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html)
- [Confluent — SMTs](https://docs.confluent.io/cloud/current/connectors/single-message-transforms.html)
- [Confluent blog — Error Handling and DLQs](https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/)
- [KIP-618 — EOS for Source Connectors](https://cwiki.apache.org/confluence/display/KAFKA/KIP-618%3A+Exactly-Once+Support+for+Source+Connectors)
- [KIP-875 — Connect offsets REST API](https://cwiki.apache.org/confluence/display/KAFKA/KIP-875%3A+First-class+offsets+support+in+Kafka+Connect)
- [KIP-802 — SMT/Converter validation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-802:+Validation+Support+for+Kafka+Connect+SMT+and+Converter+Options)
- [Debezium Architecture](https://debezium.io/documentation/reference/stable/architecture.html)
- [Debezium 3.3 Final release notes](https://debezium.io/blog/2025/10/01/debezium-3-3-final-released/)
- [Debezium JDBC sink connector](https://debezium.io/documentation/reference/stable/connectors/jdbc.html)
- [Debezium ExtractNewRecordState SMT](https://debezium.io/documentation/reference/stable/transformations/event-flattening.html)
- [Confluent S3 sink connector](https://docs.confluent.io/kafka-connectors/s3-sink/current/overview.html)
- [Confluent Elasticsearch sink connector](https://docs.confluent.io/kafka-connectors/elasticsearch/current/overview.html)
