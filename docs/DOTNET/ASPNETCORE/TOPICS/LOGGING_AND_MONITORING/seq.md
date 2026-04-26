# Seq

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [serilog](./serilog.md), [microsoft-logging](./microsoft-logging.md), [opentelemetry](./opentelemetry.md), [loki](./loki.md), [elastic-search](./elastic-search.md)._

`Seq` (by Datalust) is a **structured-log server** built around CLEF (Compact Log Event Format) — JSON events with named, typed properties. It's not a generic search engine; it's a logging product that understands message templates, scopes, exceptions, and trace correlation natively. The pitch: ELK-class structured-search ergonomics for a .NET shop without running an Elasticsearch cluster. Single-binary or container, sub-second queries on tens of millions of events on commodity hardware. Free for individual / small-scale use; paid licence past that.

## Why Seq fits .NET unusually well

- **Native CLEF understanding.** Other backends collapse `logger.LogInformation("Order {OrderId}", 42)` to one of (a) the rendered message string or (b) generic JSON. Seq stores the **template separately** — `Order {OrderId}` — and indexes `OrderId` as a typed property. You can group by template ("show me every event of this kind") *or* filter by property ("`OrderId = 42`").
- **Filter syntax is C#-shaped.** Queries are expressions: `OrderId = 42 and Level = 'Error'`, `RequestPath like '/api/%' and Elapsed > 500`. No DSL to learn.
- **Built-in trace correlation.** Logs with `TraceId`/`SpanId` (from OTel or `Activity.Current`) form into trace timelines automatically.

## Wiring — Serilog (the common path)

```csharp
builder.Host.UseSerilog((ctx, sp, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Seq(
        serverUrl: ctx.Configuration["Seq:ServerUrl"]!,
        apiKey:    ctx.Configuration["Seq:ApiKey"]));
```

Package: `Serilog.Sinks.Seq`. Production-shaped — sinks are batched (default 1000 events / 2 seconds), durable buffer on disk if Seq is unreachable (`bufferBaseFilename`), API-key auth.

## Wiring — direct from MEL (no Serilog)

```csharp
builder.Logging.AddSeq(builder.Configuration.GetSection("Seq"));
```

Package: `Seq.Extensions.Logging`. Configures Seq directly as a MEL provider — useful if you don't want to take a Serilog dependency. Serilog is more flexible (multi-sink, enrichers), but if Seq is your only sink, this is one less moving part.

## Configuration block

```jsonc
{
  "Seq": {
    "ServerUrl": "http://seq:5341",
    "ApiKey": "...",                 // create per-app in Seq UI; gives Seq an "App" identity per ingest
    "MinimumLevel": {
      "Default": "Information",
      "Override": { "Microsoft.AspNetCore": "Warning" }
    }
  }
}
```

API keys serve two purposes: (1) authenticate, (2) **tag every event from this app**. Different services use different keys; Seq tags events with the key's metadata so you can filter `Application = 'OrdersApi'` in queries.

## Querying

Seq's filter syntax is plain C#-style expressions over event properties. Common shapes:

```text
@Level >= 'Warning' and Application = 'OrdersApi'
OrderId = 42                                                    -- exact match on property
RequestPath like '/api/orders/%' and Elapsed > 500
@Exception is not null                                           -- has stack trace
@MessageTemplate = 'Placing order {OrderId} for {Customer}'      -- group by template
has(TraceId) and TraceId = '4bf92f3577b34da6a3ce929d0e0e4736'   -- trace correlation
```

`@`-prefixed properties are built-ins (`@Level`, `@Timestamp`, `@MessageTemplate`, `@Exception`, `@Message` rendered, `@EventType` template hash). Bare names are user-supplied (whatever `LogContext.PushProperty` or message template added).

## Signals — saved, named filters

A "signal" in Seq is a saved filter you can pin, share, or use as a base for ad-hoc queries. The team's first day with Seq: build signals for `Errors`, `Slow Requests`, `Orders Domain`, etc. Signals stack — open `Errors` then `Orders Domain` to see errors in that domain.

## Server-side ingestion processing — pipelines

Seq supports server-side rules that mutate events on ingest:

- **Drop noisy events** (e.g. healthcheck pings) before they hit storage.
- **Project / rename properties** (`req.path` → `RequestPath`).
- **Trigger alerts** (email, Slack, MS Teams, webhook) on matching expressions.
- **Forward** to other systems (S3, ES, generic webhooks) via apps installed in Seq.

Pipeline rules are declarative in the UI and apply per-event in real time. Cheaper than doing the same processing per-app.

## Retention and storage

Seq stores events in an embedded LMDB-shaped store. Two retention knobs:

- **Time-based** — drop events older than N days.
- **Size-based** — cap total store size; evict oldest.

Default install holds millions to tens of millions of events on a few GB. For high-volume fleets you scale **vertically** (Seq is single-node by design) — buy a bigger box. There is no built-in clustering. If you outgrow Seq, the next stop is usually [Elasticsearch](./elastic-search.md) or a SaaS.

## Trace correlation

When OpenTelemetry is wired (`AddOpenTelemetry().WithTracing(...)`) every log entry inside an `Activity` carries `TraceId`/`SpanId`. Seq groups them into a **trace view** automatically. Click an event with `TraceId` set → see every span and log on that trace, by service, in time order. This replaces a chunk of what dedicated tracing UIs (Jaeger, Tempo) do, for the .NET-shop scale where one tool covering both is enough.

## When to use Seq

- Small to medium .NET fleet (one team, < few hundred apps, < hundreds of millions of events / day) where a single Seq box is enough.
- Heavy use of structured logging — you'll get the most out of CLEF-native handling.
- Devs are the primary audience for logs (not just SREs/ops). Seq's UI is built for filter-and-explore, not dashboards.
- You want quick wins: 30-min install, indexing the next event, queryable.

## When NOT to use Seq

- Massive scale, multi-tenant, multi-region — single-node Seq becomes the bottleneck. Use [Elasticsearch](./elastic-search.md), Splunk, or a SaaS.
- Cheap-storage-priority log retention measured in months — [Loki](./loki.md) is dramatically cheaper per-GB.
- Logs are secondary to metrics/traces and you want one unified pane — go [OpenTelemetry](./opentelemetry.md) → backend (Tempo/Mimir/Loki, Datadog, Honeycomb).
- License/cost model doesn't fit. Seq is free for individual + small team; paid past that. Loki/ES/OpenSearch are open source.

## Senior-level gotchas

- **Seq's killer feature is `@MessageTemplate` grouping** — same template = same event "kind" regardless of property values. This is what makes "show me every variant of this error" trivially queryable. Lose it the moment you `string.Format` or interpolate your messages — the template becomes unique per call. Same rule as everywhere: never `$"..."` a log line.
- **Buffer to disk for resilience.** `Serilog.Sinks.Seq` accepts `bufferBaseFilename` — events that can't reach Seq queue to local disk and replay on reconnect. Without it, a Seq restart loses the last 1–2 seconds of every app's logs.
- **API keys are per-application.** Don't share one key across services — Seq's `Application` filter and per-app rate limits stop working. Cheap to mint a key; do it per service.
- **Seq is single-node.** No HA built in. The "right" answer is take regular snapshot backups and accept that a Seq restart blocks logging unless you've configured the disk buffer. Don't try to load-balance two Seq instances; events will scatter.
- **Property type sniffing happens once per property name.** First event with `OrderId = 42` types it as number; later `OrderId = "abc"` events still ingest but `OrderId = 42` queries miss them. Pick types deliberately and stick with them — don't reuse a property name for a different type per service.
- **Properties named the same across services merge.** If service A logs `Tenant` as a tenant id and service B logs `Tenant` as a tenant name, queries get confusing. Namespace user-defined properties when ambiguous (`OrdersTenantId` vs `BillingTenantName`).
- **Pipeline rules ingest-side are powerful but invisible to devs.** A rule that drops `RequestPath like '/healthz'` silently hides healthcheck logs even when something interesting happens on that path. Document pipeline rules; review them like code.
- **`Seq.Extensions.Logging` doesn't enrich with `LogContext`** the way Serilog does. If you need `CorrelationId` / `Tenant` propagated request-wide, use Serilog → Seq, not the direct provider.
- **Don't log per-request to Seq from a high-RPS service without batching.** The default batch size is fine for hundreds of events/sec; tens of thousands needs `eventBodyLimitBytes` and `batchPostingLimit` tuning, plus the disk buffer for spikes.
- **Trace view requires `TraceId` to be present and parseable.** OTel populates it; rolling-your-own correlation id with a custom property name doesn't activate Seq's trace UI. Use `Activity.Current` or OTel — not handcrafted GUIDs.
- **Free tier is per-individual, not per-team.** Production team use needs a licence. Cheap relative to the time saved over running ELK, but not free — confirm budget before standardising.
