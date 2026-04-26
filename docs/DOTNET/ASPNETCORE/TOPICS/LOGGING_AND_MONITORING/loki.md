# Grafana Loki

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [microsoft-logging](./microsoft-logging.md), [serilog](./serilog.md), [opentelemetry](./opentelemetry.md), [seq](./seq.md), [elastic-search](./elastic-search.md)._

`Grafana Loki` is a horizontally-scalable log store designed around a Prometheus-shaped model: **indexed labels, unindexed log lines.** That's the whole product in one sentence — and it's also the source of every footgun. Loki indexes only the **labels** attached to a stream of log lines (`{app="orders-api", env="prod", level="error"}`). Inside that stream, lines are stored as compressed chunks; you grep them at query time. Result: logs are *cheap to store* (TB-scale on object storage) and *fast to filter by label*, but **slow to search by content** if your label cardinality is wrong.

## The mental model

```
{app="orders-api", env="prod"}            ◄── stream key (the labels)
  ├── 2026-04-25T12:00:01Z  GET /api/orders/42 200  ◄── log line (text or JSON, NOT indexed)
  ├── 2026-04-25T12:00:02Z  POST /api/orders 201
  └── ...
```

Two queries:

```text
{app="orders-api", env="prod"}                          -- pull the entire stream by label
{app="orders-api"} |= "OrderId=42"                      -- pull stream, then grep for "OrderId=42"
{app="orders-api"} | json | OrderId="42"                -- parse each line as JSON, filter by field
```

Label match → fast (index lookup). `|=` / JSON parse → linear scan over chunks. The trick is keeping the per-stream chunk size small enough that the scan is bounded.

## The cardinality footgun

Each unique combination of label values creates a new stream. Loki's docs draw a hard line: **don't put high-cardinality data in labels.** Concretely:

| Use as label | Don't use as label |
|---|---|
| `app` (a few values) | `user_id` (millions) |
| `env` (dev/staging/prod) | `request_id` (per-request) |
| `level` (5 levels) | `order_id` (high cardinality) |
| `tenant` (if bounded) | `trace_id` |
| `pod_name` (bounded by replicas) | full URL with query string |

Bad labels = millions of streams = slow queries, memory pressure on the ingester, possibly Loki crashes. Good labels stay in single-digit-thousand stream count per service per day.

The high-cardinality data — `OrderId`, `TraceId`, `UserId` — goes **inside the log line** (as JSON properties). LogQL parses them at query time:

```text
{app="orders-api", level="error"}
  | json
  | OrderId="42"
```

Slower than label match, but bounded — only the `error`-level chunk is scanned, not the whole 500 GB ingest.

## Wiring — Serilog sink

```csharp
builder.Host.UseSerilog((ctx, sp, cfg) => cfg
    .Enrich.FromLogContext()
    .Enrich.WithProperty("app", "orders-api")
    .Enrich.WithProperty("env", ctx.HostingEnvironment.EnvironmentName)
    .WriteTo.GrafanaLoki(
        uri: "http://loki:3100",
        labels: [
            new("app", "orders-api"),
            new("env", ctx.HostingEnvironment.EnvironmentName)
        ],
        propertiesAsLabels: ["level"],         // explicitly opt-in low-card properties
        textFormatter: new JsonFormatter()));   // line content as JSON for `| json` parsing
```

Package: `Serilog.Sinks.Grafana.Loki`. The critical line is `propertiesAsLabels: ["level"]` — by default, the sink can promote properties to labels, which is exactly the cardinality landmine. Be explicit; allow only the bounded ones.

## Wiring — OTel logs to Loki

```csharp
builder.Logging.AddOpenTelemetry(o => o.AddOtlpExporter());
// → Collector → Loki via the loki receiver/exporter
```

The OTel collector's `lokiexporter` (or Loki's native OTLP endpoint) accepts log records and uses Resource attributes as labels. Same cardinality rule applies — keep `service.name`, `service.version`, `deployment.environment` as labels; everything else as line content. See [opentelemetry](./opentelemetry.md).

## LogQL basics

```text
{app="orders-api"}                                    -- stream selector (required)
{app="orders-api"} |= "error"                         -- line contains
{app="orders-api"} != "healthz"                       -- line does NOT contain
{app="orders-api"} |~ "(?i)timeout"                   -- regex match
{app="orders-api"} | json                             -- parse JSON, exposes fields
{app="orders-api"} | json | level="Error"             -- filter by JSON field
{app="orders-api"} | json | line_format "{{.Message}}" -- reshape output

-- Metrics from logs (range vectors, like Prometheus):
rate({app="orders-api"} |= "error" [5m])              -- error log rate
sum by (app) (count_over_time({env="prod"} [1h]))     -- log volume per app
histogram_quantile(0.99,
  sum(rate({app="orders-api"} | json | unwrap Elapsed [5m])) by (le))
```

The `unwrap` clause turns a numeric field into a histogram input — that's how you derive p99 latency from log lines without a separate metric.

## Ingestion — direct push vs agent

| Path | When |
|---|---|
| **Direct push** from app (Serilog/OTel sink) | Small fleets; fewer moving parts. Sink batches and retries; on Loki down, in-memory buffer or disk. |
| **Agent (Promtail / Grafana Alloy / OTel Collector)** | Production. Apps log to stdout; agent tails the container/node logs and pushes to Loki. App is decoupled from log infra outage. |

Containers in Kubernetes: write JSON to stdout (`AddJsonConsole`), let Promtail/Alloy as a DaemonSet ship lines to Loki. App stays simple, agent handles batching and back-pressure.

## Storage and scaling

Loki uses **object storage** (S3, GCS, Azure Blob) for chunks plus a small index store. That's why retention is so cheap — pay storage prices, not Elasticsearch shard prices. Compaction merges chunks; retention drops them. Sizes worth knowing:

- **Chunks** target ~1.5 MB compressed per stream — small streams = many tiny chunks = waste; huge streams = slow scans. Cardinality balances this for you.
- **Retention** is set globally and per-tenant. 30 / 90 / 365-day retention on cheap object storage is typical.
- **Boltdb-shipper** has been replaced by **TSDB** as the index store for new clusters — TSDB scales label cardinality better.

## When to use Loki

- Cost-sensitive log retention at scale — TB/month doesn't fit ES budget.
- Already on Grafana stack — Tempo (traces), Prometheus/Mimir (metrics), Loki (logs) integrate in one UI.
- Bounded label cardinality is a constraint you can live with.

## When NOT to use Loki

- Devs primarily want full-text structured search by arbitrary field — [Seq](./seq.md) or [Elasticsearch](./elastic-search.md) are dramatically better at that. Loki's `| json | Field="x"` works but is slower because there's no field index.
- High label cardinality is unavoidable (multi-tenant SaaS with millions of tenants).
- Small fleet — Seq's setup cost is lower and the UX is better for "find this one error."

## Senior-level gotchas

- **Cardinality is THE rule.** Repeat: `OrderId`, `UserId`, `TraceId` → log line content, never labels. A junior who promotes `request_id` to a label can OOM your ingester within hours. The Loki team treats >100k active streams per tenant as a smell — verify with `loki_ingester_memory_streams`.
- **Auto-promoting properties to labels is the most common configuration bug.** `propertiesAsLabels` in `Serilog.Sinks.Grafana.Loki` defaults to *all* event properties in some versions — check explicitly and pin to the bounded set.
- **Loki indexes only labels — `|=` is grep.** Searching for `OrderId=42` across a wide time range without label filters scans every chunk Loki has. Always pin a stream selector first; `{app="..."}` is mandatory in practice.
- **`| json` parses the whole line every time.** For frequent queries on the same field, store it as a *structured metadata* attribute (Loki 2.9+) — that's a middle-ground index between labels (cardinality-limited) and unindexed line content.
- **Stream "churn" is invisible until it hurts.** If a label takes new values across deploys (e.g. `version="2026.04.25-abc123"` per build), every deploy creates fresh streams that fragment storage. Monitor `loki_ingester_streams_created_total`.
- **The `level` label is convenient but not free.** Each level becomes a separate stream. Multiplied by `app` × `env` × `pod_name` it adds up. Acceptable cost for the query ergonomics it buys; just know the math.
- **Logs and traces correlate via `TraceId` *in the line*, not via a label** (since trace ids are obviously high-cardinality). Grafana's "trace ↔ logs" linking uses derived field config to extract `TraceId` from the log line text and link to Tempo. Set up the derived field; otherwise correlation isn't automatic.
- **Direct push from app code couples log delivery to app health.** If Loki is slow, your app blocks (or drops) on the sink. Promtail/Alloy as a sidecar/DaemonSet decouples — app writes to stdout and is done; the agent handles retries.
- **LogQL has no joins.** You can't `JOIN logs ON OrderId = OrderId` between two services in one query. Either include enough context per log line, or use Tempo/distributed tracing for cross-service correlation.
- **Loki retention is global per-tenant.** "Keep audit logs 7 years, app logs 14 days" doesn't work in one tenant — split into multi-tenant config or pre-shunt audit logs to a different store.
- **The Grafana Cloud Loki SaaS bills by ingest GB and active series.** "We can just store all our logs there" is fine until you uncap label cardinality and the bill 50× overnight. Set per-tenant stream limits server-side.
- **Loki is open-source AGPLv3.** For most users that's irrelevant; for vendors embedding Loki it has implications. Grafana Cloud or Grafana Enterprise Logs sidesteps the licence concern.
