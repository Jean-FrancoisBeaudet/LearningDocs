# Schema Registry

> Version baseline: Apache Kafka 4.x (KRaft mode), Confluent Schema Registry
> current (7.x-line), Apicurio Registry 3.x. Last researched: 2026-04.

## TL;DR

Kafka moves **bytes**. Schema Registry lets producers and consumers agree
on the **structure** of those bytes without shipping the schema inside
every record. Every record's payload is prefixed with a tiny header
(`magic byte + schema ID`); the ID is a pointer into a central REST service
that stores the actual schema and enforces evolution rules.

Three supported schema languages:

- **Avro** — most common in Kafka, rich evolution rules, compact binary.
- **Protobuf** — Google's format; good for polyglot orgs already using it.
- **JSON Schema** — validation over JSON; human-readable but bulky.

Two major implementations:

- **Confluent Schema Registry** — de-facto standard; closed-source-ish
  (Confluent Community License). Powers most production Kafka setups.
- **Apicurio Registry** — Red Hat / open source (Apache 2.0). Supports the
  same wire format in "Confluent compatibility" mode and adds features like
  multi-tenant REST, content-based dedupe, and broader format support
  (OpenAPI, AsyncAPI, GraphQL).

The three mechanics you must understand at senior level:

1. **Wire format**: `[0x00][4-byte schema ID, big-endian][payload]`.
2. **Compatibility modes**: BACKWARD (default), FORWARD, FULL, NONE — each
   with a TRANSITIVE variant that checks against **all** prior versions
   instead of just the latest.
3. **Subject naming strategies**: TopicNameStrategy (default, one schema
   per topic), RecordNameStrategy (schema identity = record type),
   TopicRecordNameStrategy (both).

---

## Deep dive

### 1. Wire format

Every message payload that goes through a Confluent/Apicurio(-compat)
serializer is prefixed with a 5-byte header:

```
Byte 0       : 0x00                      (magic byte)
Bytes 1-4    : schema ID (int32 BE)      (assigned by the registry)
Bytes 5..N   : serialized payload
               - Avro:    Avro binary (no embedded schema)
               - Protobuf: message index array (varint) + protobuf bytes
               - JSON Schema: UTF-8 JSON
```

- The magic byte (`0x00`) exists so deserializers can detect "this was
  produced through Schema Registry" vs raw Avro/JSON/Protobuf. It also
  leaves room for future wire formats (nobody has actually bumped it yet).
- The schema ID is **globally unique** within one Schema Registry, not
  per-subject. Two subjects referencing the same Avro schema share an ID.
- On deserialization, the client: reads magic byte, reads ID, asks the
  registry `GET /schemas/ids/{id}` (cached locally), then decodes.

For **Protobuf**, after the 5-byte header there is an additional **message
index** (a varint array) that tells the deserializer which top-level
message in the `.proto` file to use — Protobuf files can contain multiple
messages, unlike Avro where one schema = one record.

Apicurio has two modes:

- **Apicurio native** — schema ID is sent via Kafka **headers**, payload
  has no magic byte. Not interoperable with Confluent deserializers.
- **Confluent-compatible** (`apicurio.registry.as-confluent=true`) — same
  5-byte header, fully interoperable.

### 2. Subjects and schema IDs

- **Schema** — an Avro/Protobuf/JSON Schema document.
- **Subject** — a versioned *history* of schemas. Compatibility rules are
  enforced per subject. Each version gets a **schema ID** (global) and a
  **version number** (per-subject).

The registry stores its state in a compacted Kafka topic (`_schemas` by
default), partitioned by subject. Multiple registry instances form an
active/active cluster; only the leader accepts writes.

REST API cheat sheet:

```
POST   /subjects/{subject}/versions         # register (returns id)
GET    /subjects/{subject}/versions/latest  # latest schema
GET    /schemas/ids/{id}                    # fetch by global id
POST   /compatibility/subjects/{subject}/versions/latest   # dry-run
PUT    /config/{subject}                    # change compatibility mode
GET    /subjects                            # list
DELETE /subjects/{subject}                  # soft-delete (version history retained)
DELETE /subjects/{subject}?permanent=true   # hard-delete (post-soft-delete)
GET    /mode                                # READWRITE | READONLY | IMPORT
```

### 3. Compatibility modes

When a producer registers a new schema for an existing subject, the
registry checks it against the compatibility rule:

| Mode | Check | Evolution allowed | Update order |
|---|---|---|---|
| `BACKWARD` (default) | New can read data produced by **latest** old | Add optional fields, delete fields (with defaults) | **Consumers first** |
| `BACKWARD_TRANSITIVE` | New can read data produced by **all** old versions | Same as BACKWARD but across full history | Consumers first |
| `FORWARD` | **Latest** old can read data produced by new | Add fields (any), delete optional fields | **Producers first** |
| `FORWARD_TRANSITIVE` | **All** old can read data produced by new | Same as FORWARD but across full history | Producers first |
| `FULL` | Both BACKWARD and FORWARD against **latest** | Only additions/deletions of fields with defaults | Either order |
| `FULL_TRANSITIVE` | Both BACKWARD and FORWARD against **all** versions | Strictest; all history must read all history | Either order |
| `NONE` | No checks — anything goes | Anything | Chaos |

Senior rule of thumb:

- Default to **BACKWARD** — consumers upgrade first, new fields are
  optional with defaults, old fields can be deleted.
- Use **FULL_TRANSITIVE** for long-lived critical topics (e.g., customer
  events kept for years) where you cannot control in what order
  consumers/producers upgrade and you may replay old data.
- Use **FORWARD** for event-sourcing topics where the producer (the source
  of truth) can evolve first and consumers catch up.
- Never use **NONE** in production unless you run a manual review gate
  elsewhere (GitOps PR review).

Change the mode per-subject, not globally, unless your team is tiny:

```bash
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility":"FULL_TRANSITIVE"}' \
  http://sr:8081/config/orders-value
```

### 4. Subject naming strategies

The client-side serializer picks the subject name. Three built-in
strategies (set via `key.subject.name.strategy` / `value.subject.name.strategy`):

| Strategy | Subject name formula | Use case |
|---|---|---|
| **TopicNameStrategy** (default) | `<topic>-key` / `<topic>-value` | One record type per topic. Simplest. |
| **RecordNameStrategy** | `<fully-qualified-record-name>` | Same record type published to many topics; or event-sourcing with many types per topic where the *type* is the schema identity. |
| **TopicRecordNameStrategy** | `<topic>-<FQN>` | Many record types per topic, but compatibility scoped per-topic. |

Why this matters: if you want to put multiple event types into one topic
(e.g., `customer-events` carrying `CustomerCreated`, `CustomerUpdated`,
`CustomerDeleted`), you **must** use `RecordNameStrategy` or
`TopicRecordNameStrategy`. The default rejects the second schema with a
compatibility error because it would compare `CustomerUpdated` against
`CustomerCreated` under `BACKWARD`.

Avro union types + one of the record-name strategies is the canonical
pattern for multi-type topics.

### 5. Avro evolution rules

Avro has specific rules that map onto the compatibility modes:

- **Adding a field with a default**: BACKWARD + FORWARD = FULL-safe.
- **Adding a field without a default**: FORWARD only (old readers can't
  read new data, but new readers work).
- **Removing a field with a default**: BACKWARD safe (new readers can read
  old data by using the default).
- **Removing a field without a default**: neither safe.
- **Renaming a field**: NOT safe under any mode. Use `aliases` on the new
  name: `{"name":"email","aliases":["email_address"]}`.
- **Changing field type**: only "promotions" are allowed (int -> long,
  float -> double, string <-> bytes). Most type changes require a new
  field + a migration period.
- **Enum symbol additions**: require a `default` on the enum (Avro 1.9+)
  so old readers have a fallback — otherwise they throw on an unknown
  symbol.
- **Union changes**: adding a branch to a union requires the branch to
  have a default-able position (typically add at the end + use default
  null).

### 6. Protobuf evolution rules

- Adding an **optional** field (which is all fields in proto3 by default)
  is always safe. Readers see the default value (0, "", null).
- Removing a field is safe **as long as the field number is reserved**
  so it's never re-used:
  ```proto
  message Order {
    reserved 4, 7 to 9;
    reserved "old_field_name";
    // ...
  }
  ```
- Renaming a field is safe on the wire (field numbers, not names, are
  encoded). It is NOT safe for JSON encoding.
- Changing a field's type is generally unsafe except wire-compatible
  pairs (int32 <-> int64 <-> uint32 <-> uint64 <-> bool).
- Enum additions: safe if readers use `UNKNOWN = 0` as default and handle
  unknown values defensively.
- Protobuf does **not** have "required" since proto3; schema registry
  cannot enforce "must have field X" — validate in application code.

### 7. JSON Schema evolution rules

- Adding a field: safe if `required` is not updated to include it.
- Making a field optional (removing from `required`): FORWARD-safe.
- Tightening a constraint (`minLength`, `pattern`): breaks BACKWARD.
- Loosening a constraint: breaks FORWARD.
- JSON Schema is the hardest to evolve safely because it has a huge
  surface area (enums, patterns, `oneOf`, `$ref`). Most orgs use it only
  when Avro/Protobuf are off the table.

### 8. Schema references

All three formats support **references**: one schema importing another.
Example Avro `OrderLine` referenced from `Order`:

```json
{
  "schema": "{ ... \"fields\":[{\"name\":\"lines\",\"type\":{\"type\":\"array\",\"items\":\"com.acme.OrderLine\"}}] ... }",
  "references": [
    { "name": "com.acme.OrderLine", "subject": "order-line", "version": 3 }
  ]
}
```

The registry resolves references at deserialization time. References are
the official alternative to copy-pasting common types.

### 9. Confluent vs Apicurio

| Aspect | Confluent | Apicurio |
|---|---|---|
| License | Confluent Community License | Apache 2.0 |
| Wire format | Magic byte + ID (payload) | Native (headers) **or** Confluent-compat (payload) |
| Formats | Avro, Protobuf, JSON Schema | Above + OpenAPI, AsyncAPI, GraphQL, WSDL, XSD |
| Multi-tenancy | Via RBAC in Confluent Platform | First-class (tenants per REST path) |
| Backend | Kafka compacted topic `_schemas` | Kafka, Postgres, MongoDB, in-memory |
| Compatibility modes | 7 (BACKWARD, FORWARD, FULL, each with TRANSITIVE, NONE) | Same 7 + compatibility on **validity** |
| Validation | Schema only | Content validity + custom rules |
| Java client | `kafka-avro-serializer` etc. | Apicurio serdes (or Confluent-compat mode with confluent serdes) |
| Registry-side operations | Subject aliases (CP 7.5+), data contracts (CP 7.4+) | Content groups, artifact rules, search |

Pick Confluent if you're already on Confluent Platform / Cloud or most of
your connectors/clients expect it. Pick Apicurio for open-source purity,
non-Kafka schema reuse (API schemas in the same registry), or
multi-tenancy.

### 10. Data contracts (CP 7.4+) and schema rules

Confluent's "Data Contracts" add to subjects:

- **Metadata** — owners, tags, description.
- **Rules** — migration rules (transform old records into new format on
  read), domain rules (SQL-like constraints), PII tags.
- Rules run **inside the deserializer** — the app gets validated + migrated
  records automatically.

Apicurio equivalent: artifact rules (validity, compatibility, integrity).

---

## Config reference

### Serializer (Confluent.SchemaRegistry.Serdes.*)

| Config | Default | Meaning |
|---|---|---|
| `schema.registry.url` | _required_ | REST endpoint(s), CSV |
| `basic.auth.credentials.source` | `URL` | or `USER_INFO`, `SASL_INHERIT` |
| `basic.auth.user.info` | _none_ | `user:password` |
| `auto.register.schemas` | `true` | Register on first produce; **set `false` in prod** |
| `use.latest.version` | `false` | Force the latest registered version instead of serializer's own |
| `normalize.schemas` | `false` | Canonicalize (sort fields) before compare — avoids ID churn |
| `key.subject.name.strategy` | `TopicNameStrategy` | Subject naming |
| `value.subject.name.strategy` | `TopicNameStrategy` | Subject naming |
| `schemas.cache.capacity` | `1000` | In-memory schema-ID cache per client |

### Registry (server)

| Config | Meaning |
|---|---|
| `kafkastore.bootstrap.servers` | Kafka (storage) |
| `kafkastore.topic` | `_schemas` |
| `kafkastore.topic.replication.factor` | 3 in prod |
| `schema.compatibility.level` | Cluster default (per-subject overrides) |
| `mode.mutability` | Allow `PUT /mode/{subject}` |
| `leader.eligibility` | `true` for writable nodes |

---

## .NET / C# snippet (Confluent.SchemaRegistry + Avro)

Full producer + consumer with Avro, Schema Registry, explicit compatibility
checking, and `auto.register.schemas=false` (production hygiene).

```csharp
// dotnet add package Confluent.Kafka
// dotnet add package Confluent.SchemaRegistry
// dotnet add package Confluent.SchemaRegistry.Serdes.Avro
// dotnet add package Avro

using Avro;
using Avro.Specific;
using Confluent.Kafka;
using Confluent.SchemaRegistry;
using Confluent.SchemaRegistry.Serdes;

// 1. Schema Registry client (shared; thread-safe; cache schemas locally).
var srConfig = new SchemaRegistryConfig
{
    Url = "http://sr:8081",
    // BasicAuthCredentialsSource = AuthCredentialsSource.UserInfo,
    // BasicAuthUserInfo          = "api-key:api-secret",
};
using var sr = new CachedSchemaRegistryClient(srConfig);

// 2. Producer — Avro value, string key.
var producerConfig = new ProducerConfig
{
    BootstrapServers  = "broker:9092",
    EnableIdempotence = true,
    Acks              = Acks.All,
    CompressionType   = CompressionType.Zstd,
};

var avroSerializerConfig = new AvroSerializerConfig
{
    // In prod, register schemas explicitly via CI/CD, not at runtime.
    AutoRegisterSchemas = false,
    // If your local POCO schema diverges slightly from the registered one,
    // this asks the registry for the latest instead of registering yours.
    UseLatestVersion    = true,
    SubjectNameStrategy = SubjectNameStrategy.Topic,   // or Record / TopicRecord
};

using var producer = new ProducerBuilder<string, OrderCreated>(producerConfig)
    .SetValueSerializer(new AvroSerializer<OrderCreated>(sr, avroSerializerConfig))
    .Build();

var orderEvent = new OrderCreated
{
    order_id    = Guid.NewGuid().ToString(),
    customer_id = "C-42",
    total_cents = 19999,
    event_ts    = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
};

await producer.ProduceAsync("orders",
    new Message<string, OrderCreated> { Key = orderEvent.order_id, Value = orderEvent });

producer.Flush(TimeSpan.FromSeconds(5));

// 3. Consumer — same Avro schema.
var consumerConfig = new ConsumerConfig
{
    BootstrapServers = "broker:9092",
    GroupId          = "orders-indexer",
    AutoOffsetReset  = AutoOffsetReset.Earliest,
    IsolationLevel   = IsolationLevel.ReadCommitted,
    EnableAutoCommit = false,
};

using var consumer = new ConsumerBuilder<string, OrderCreated>(consumerConfig)
    .SetValueDeserializer(new AvroDeserializer<OrderCreated>(sr).AsSyncOverAsync())
    .Build();

consumer.Subscribe("orders");

while (!ct.IsCancellationRequested)
{
    var cr = consumer.Consume(TimeSpan.FromSeconds(1));
    if (cr is null) continue;
    Console.WriteLine($"{cr.Message.Key}: {cr.Message.Value.total_cents} @ {cr.Message.Value.event_ts}");
    consumer.Commit(cr);
}

// 4. Pre-registering schemas and checking compatibility from CI.
var schemaText = await File.ReadAllTextAsync("schemas/order-created.avsc");
var schema     = new Schema(schemaText, SchemaType.Avro);

// Dry-run: will the new schema be accepted?
bool compatible = await sr.IsCompatibleAsync("orders-value", schema);
if (!compatible)
    throw new InvalidOperationException("Schema change breaks compatibility on orders-value");

int newId = await sr.RegisterSchemaAsync("orders-value", schema);
Console.WriteLine($"Registered as schema id {newId}");
```

### Protobuf variant

```csharp
// dotnet add package Confluent.SchemaRegistry.Serdes.Protobuf

using Confluent.SchemaRegistry.Serdes;

using var producer = new ProducerBuilder<string, OrderCreatedProto>(producerConfig)
    .SetValueSerializer(new ProtobufSerializer<OrderCreatedProto>(sr,
        new ProtobufSerializerConfig { AutoRegisterSchemas = false }))
    .Build();
```

### JSON Schema variant

```csharp
// dotnet add package Confluent.SchemaRegistry.Serdes.Json

using Confluent.SchemaRegistry.Serdes;

using var producer = new ProducerBuilder<string, OrderCreated>(producerConfig)
    .SetValueSerializer(new JsonSerializer<OrderCreated>(sr,
        new JsonSerializerConfig { AutoRegisterSchemas = false }))
    .Build();
```

---

## Senior-level gotchas

1. **`auto.register.schemas=true` in prod is how you lose a Friday night.**
   A developer renames a field locally, app auto-registers an incompatible
   schema, the registry rejects it and **the producer crashes**. Even if it
   *succeeds*, you now have an unreviewed schema in prod. Lock it down:
   `auto.register.schemas=false` + CI-driven registration via
   `POST /subjects/{subject}/versions`.

2. **Compatibility mode is per-**subject**, not per-topic.** `orders-key`
   and `orders-value` can have different modes. Changing the cluster
   default only affects subjects with no explicit setting.

3. **`BACKWARD` is not transitive by default.** With a chain of versions
   V1 -> V2 -> V3, BACKWARD only checks V3 vs V2, not V3 vs V1. If you
   replay historical topics (Kafka's superpower), a V1-serialized message
   may crash a V3 consumer. For long-retention topics use
   `BACKWARD_TRANSITIVE` or `FULL_TRANSITIVE`.

4. **Field defaults are not decoration — they are the evolution contract.**
   Adding a field without a default breaks BACKWARD. Always ship defaults
   (`null` is fine) for new fields.

5. **Enum additions break old consumers without defaults.** Avro 1.9 added
   enum `default` precisely for this. On Protobuf, always reserve
   `UNKNOWN = 0` and handle it everywhere.

6. **Renaming is not possible.** Use Avro `aliases` (during a migration
   window, then drop the old name), or add the new field + deprecate the
   old one + remove in a later release.

7. **`normalize.schemas=true` prevents ID inflation.** Without it,
   whitespace-only changes or reordered fields register new schema IDs
   forever. With it, semantically equal schemas get the same ID.

8. **Deleting a subject is usually a mistake.** Soft-delete hides it; hard-
   delete removes the history. If a consumer later sees a message whose
   schema ID points to a deleted schema, it cannot deserialize. Only
   hard-delete after you are sure nobody will ever replay.

9. **Schema ID lookups are cached, not free.** The first encounter of a
   new ID in a hot path makes an HTTP call. Size `schemas.cache.capacity`
   for your cardinality, and for RecordNameStrategy topics with hundreds
   of record types, monitor registry request rate.

10. **TopicNameStrategy + multi-event topic = pain.** The moment you try
    to publish a second event type to the same topic under the default
    strategy, the registry rejects it as incompatible. Decide up-front:
    one type per topic (TopicNameStrategy) OR many types per topic
    (RecordNameStrategy / TopicRecordNameStrategy) — migrating later is
    brutal.

11. **Keys often don't need Schema Registry.** Most Kafka keys are
    primitive strings (IDs). Using `StringSerializer` for the key and
    Avro only for the value is simpler and saves the `-key` subject.
    Only use a keyed schema when the key has actual structure.

12. **Apicurio "native" vs "Confluent-compat" is a foot-gun.** If your
    producer uses Confluent serializers and your consumer uses Apicurio
    native serdes (or vice versa), deserialization silently fails — the
    magic byte isn't where the other side expects. Pick one mode and
    enforce it at the org level.

13. **Schema Registry is a single point of failure for new deserializers.**
    A consumer that has the ID cached keeps working if the registry is
    down, but any new ID (new producer schema version) will fail until
    the registry returns. Run it HA, and cache aggressively client-side.

14. **`IMPORT` mode is the only safe way to bulk-load pre-existing
    schemas** (e.g., cross-cluster migration). It accepts schemas with
    pre-assigned IDs/versions. `READWRITE` (the normal mode) does not.

15. **Data Contracts / Apicurio rules run client-side in the
    deserializer.** They can reject or migrate records the registry
    never saw — great for enforcement, but the *client* must be on a
    new-enough version of the serde library. An old consumer silently
    ignores the rules.

---

## References

- [Confluent — Schema Registry concepts](https://docs.confluent.io/platform/current/schema-registry/fundamentals/index.html)
- [Confluent — Schema Evolution and Compatibility](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html)
- [Confluent — Schema Subjects](https://developer.confluent.io/courses/schema-registry/schema-subjects/)
- [Confluent — Schema Compatibility course](https://developer.confluent.io/courses/schema-registry/schema-compatibility/)
- [Confluent — Manage Schemas](https://docs.confluent.io/cloud/current/sr/schemas-manage.html)
- [Confluent — Schema Registry Best Practices](https://www.confluent.io/blog/best-practices-for-confluent-schema-registry/)
- [Confluent — Serializers / Deserializers](https://docs.confluent.io/platform/current/schema-registry/fundamentals/serdes-develop/index.html)
- [Demystifying the Confluent Schema Registry wire format](https://dev.to/stevenjdh/demystifying-confluents-schema-registry-wire-format-5465)
- [Apicurio Registry — Kafka client serdes](https://www.apicur.io/registry/docs/apicurio-registry/1.3.3.Final/getting-started/assembly-using-kafka-client-serdes.html)
- [AutoMQ 2025 comparison: Confluent / AWS Glue / Apicurio / Redpanda](https://www.automq.com/blog/kafka-schema-registry-confluent-aws-glue-redpanda-apicurio-2025)
- [Debezium — Avro Serialization](https://debezium.io/documentation/reference/stable/configuration/avro.html)
