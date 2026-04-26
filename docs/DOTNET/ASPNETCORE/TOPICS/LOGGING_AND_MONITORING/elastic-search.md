# Elasticsearch / ELK

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [microsoft-logging](./microsoft-logging.md), [serilog](./serilog.md), [opentelemetry](./opentelemetry.md), [seq](./seq.md), [loki](./loki.md)._

`Elasticsearch` is a distributed inverted-index database — the storage half of the **ELK** / **Elastic Stack** (E = Elasticsearch, L = Logstash or now Elastic Agent, K = Kibana for query/dashboards). For .NET logs the appeal is full-text search **plus** typed-field aggregations at terabyte/petabyte scale, with mature multi-tenant clustering. The cost: it's a real distributed database — you operate shards, mappings, ILM policies, and pay for the JVM heap. The right shape on the .NET side is **emit ECS-formatted JSON** (Elastic Common Schema — Elastic's standard field naming) and let either Filebeat/Elastic Agent or a direct sink ship it.

## What ECS gives you

ECS is a field-naming convention: `@timestamp`, `log.level`, `service.name`, `http.request.method`, `error.stack_trace`, etc. Two reasons it matters:

1. **Out-of-the-box Kibana dashboards work.** They query `service.name`, not `MyAppNameField`.
2. **Mappings are predictable.** `http.response.status_code` is always a `long`; `@timestamp` is always `date`. No mapping explosions from inconsistent field types.

Use Elastic's first-party packages so this happens for free.

## Wiring — Serilog with ECS

```csharp
using Elastic.CommonSchema.Serilog;
using Serilog.Sinks.Elasticsearch;

builder.Host.UseSerilog((ctx, sp, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithEcsHttpContext(sp.GetRequiredService<IHttpContextAccessor>())
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("https://es:9200"))
    {
        AutoRegisterTemplate = true,
        AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv8,
        TypeName = null,                 // Elasticsearch 8+: typeless
        IndexFormat = "logs-orders-{0:yyyy.MM}",
        CustomFormatter = new EcsTextFormatter(),   // ECS-shaped JSON
        ModifyConnectionSettings = c => c.BasicAuthentication("user", "pass")
    }));
```

Packages: `Elastic.CommonSchema.Serilog`, `Serilog.Sinks.Elasticsearch` (or the newer `Elastic.Serilog.Sinks` direct from Elastic). The newer sink is the preferred path going forward — same idea, owned by Elastic.

## Wiring — OpenTelemetry → Elastic

```csharp
builder.Services.AddOpenTelemetry()
    .UseElasticOpenTelemetry();    // Elastic.OpenTelemetry
```

Elastic operates an OTLP endpoint on Elastic Cloud, so the OTel path skips Filebeat entirely — app sends OTLP → Elastic. For self-managed deploys, run an OTel collector with the `elasticsearch` exporter. See [opentelemetry](./opentelemetry.md).

## Index lifecycle (ILM) — non-optional

Logs in Elasticsearch require a lifecycle policy. Without one, you fill disks and the cluster locks. Standard shape:

| Phase | Duration | Action |
|---|---|---|
| **Hot** | 0–7 days | Active writes + queries; SSD; many replicas |
| **Warm** | 7–30 days | Read-only; force-merged; fewer replicas |
| **Cold** | 30–90 days | Searchable but slow tier (HDD or frozen tier on object storage) |
| **Delete** | > 90 days | Drop index |

Wired as an ILM policy + index template + rollover alias. Use `logs-*` data streams (introduced in ES 7.9+) — they handle rollover automatically and pair with ILM policies. Index per month (`logs-orders-2026.04`) is fine for low volumes; rollover at 50 GB / 30 days for high volumes.

## Mapping explosions — the #1 cluster killer

Elasticsearch defaults to **dynamic mapping** — first time it sees a new field, it infers a type. Two failure modes:

- **Mapping limit** (default 1000 fields per index). Apps that log per-request unique fields (e.g. promoting every querystring key) hit this and writes start failing.
- **Type conflicts.** Service A logs `userId` as a number; service B logs it as a string. The index that maps it first wins; the other service's events get rejected with `mapper_parsing_exception`.

Defenses:

1. **Strict templates.** Define the schema (ECS) and refuse unknown fields: `"dynamic": "strict"` or `"runtime"` mappings (computed at query time, no index cost).
2. **Pre-aggregate / namespace user fields.** Anything custom goes under `labels.*` (free-form keyword) or `numeric_labels.*`.
3. **One service = one index pattern.** Don't co-mingle services into a shared `logs-*` write index.

## Querying — Kibana + KQL/Lucene

Kibana Discover is the log-search UI:

```text
service.name : "orders-api" and log.level : "error"
http.response.status_code >= 500 and url.path : "/api/orders/*"
order.id : 42
trace.id : "4bf92f3577b34da6a3ce929d0e0e4736"
```

Two query languages: KQL (recommended, simpler) and Lucene (more features). Aggregations and dashboards then build on the same indexed fields. ECS field names are the contract — once your apps emit ECS, every dashboard built against `service.name` / `http.*` works.

## Sizing rules of thumb

- **Heap**: 50% of RAM, max 31 GB (compressed pointers cliff). One Elasticsearch node = one JVM = ≤ 31 GB heap regardless of box size; bigger boxes run multiple nodes.
- **Shards**: target 10–50 GB per shard. Too small → overhead per shard; too large → slow recovery.
- **Index per service per month** for moderate volumes; **rollover by size** for high.
- **Hot tier on SSD**, warm/cold on HDD or searchable snapshots. Tier transitions are where ILM earns its keep.

## OpenSearch — the AWS fork

Elasticsearch's licence changed in 2021 (Elastic Licence + SSPL); AWS forked it as **OpenSearch** (Apache 2.0). The .NET clients (`Elastic.Clients.Elasticsearch`, `Elastic.Serilog.Sinks`, `Elastic.OpenTelemetry`) speak Elasticsearch's API. OpenSearch has its own client (`OpenSearch.Client`) and the wire APIs have diverged enough that you can't always use the Elastic packages against OpenSearch unchanged. Pick one and standardise — for AWS-managed (`Amazon OpenSearch Service`) use OpenSearch packages; for Elastic Cloud / self-managed Elastic, use Elastic's.

## When to use Elasticsearch / OpenSearch

- Full-text + structured search at large scale.
- Existing ELK / Elastic Stack investment, Kibana dashboards, SIEM, etc.
- Hot/warm/cold tiering with retention measured in months/years.
- Mixed log + APM + security analytics on one platform.

## When NOT to

- Small fleet, devs primary audience — [Seq](./seq.md) is faster to set up and easier to operate.
- Cheap log retention is the dominant requirement — [Loki](./loki.md) on S3 is dramatically cheaper.
- You don't have ops capacity to run a stateful distributed JVM cluster — go SaaS (Elastic Cloud, AWS OpenSearch, Logz.io, …) or use a simpler product.

## Senior-level gotchas

- **ECS or you're going to redo this.** Custom field names work for the first six months and then break the day someone wants a Kibana dashboard. Standardise on ECS at the sink (`EcsTextFormatter`, `Elastic.OpenTelemetry`). The migration cost from non-ECS to ECS later is non-trivial.
- **Dynamic mapping is a foot-cannon.** Devs log per-request unique fields ("`promoCode_XYZ123`": value) thinking they're just adding properties — Elasticsearch indexes each as a new field. Hits the 1000-field mapping limit and writes fail cluster-wide. Use `dynamic: strict` plus a free-form `labels.*` for ad-hoc properties.
- **`{@Object}` destructuring of EF entities → mapping explosion.** Tracked entities have hundreds of properties (and proxies). Project to a DTO before logging or configure Serilog `ByTransforming<TEntity>(...)` — see [serilog](./serilog.md).
- **Type conflicts across services kill ingest silently.** Service A logs `Tenant` as a string; service B as a number. The second service's events get `mapper_parsing_exception` — but unless you're watching the dead-letter queue or the sink's self-log, you'll never know. Centralise the schema (ECS) and CI-check for foreign field names.
- **No ILM policy = guaranteed disk-full incident.** Set ILM up before sending the first log. Monitor `cluster.routing.allocation.disk.watermark` — when watermark trips, the cluster goes read-only and writes 5xx until you free space. By then it's an outage.
- **Shards are not free.** The "thousands of indices, dozens of shards each" pattern is how clusters become unmanageable. Use data streams, target 10–50 GB shards, and don't pre-create monthly indices for years.
- **Bulk failures are partial.** A 1000-doc bulk request can succeed for 999 and fail for 1; the sink must read the response and retry only failed items. `Serilog.Sinks.Elasticsearch` does this; ad-hoc HTTP loops often don't.
- **Cluster auth defaults changed in 8.x.** Security is on by default with TLS and built-in users. New apps that copy old config (anonymous access on 9200) will not connect. `https://` and basic-auth or API key, not `http://`.
- **Filebeat / Elastic Agent vs direct sink.** Direct sink (Serilog → ES) is simpler but couples app to ES availability. Filebeat as a sidecar/DaemonSet tailing stdout is the production shape — survives ES outages with on-disk queues. App stays on `AddJsonConsole`.
- **Searchable snapshots and frozen tier** are how you afford long retention. Frozen indices live on object storage and are queried lazily; near-line hot tier becomes affordable. Setting them up is real work — but it's the difference between affording 1 year of retention and not.
- **Elastic APM ≠ OpenTelemetry, but Elastic accepts OTLP.** New apps should emit OTel and let Elastic ingest as OTLP — rather than the legacy Elastic APM .NET agent. Future-proofs the wiring; same backend.
- **Licence: Elastic vs OpenSearch matters for self-hosting at scale.** Cloud-managed (Elastic Cloud, AWS OpenSearch) is unaffected — you're paying for the service. Embedding ES in a product you ship triggers SSPL terms; that's when you fork to OpenSearch or pay for an Elastic commercial licence.
