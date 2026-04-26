# OpenTelemetry

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [microsoft-logging](./microsoft-logging.md), [serilog](./serilog.md), [seq](./seq.md), [loki](./loki.md), [elastic-search](./elastic-search.md)._

OpenTelemetry (OTel) is the **vendor-neutral standard** for emitting telemetry — three signals, one SDK, one wire format (OTLP). For ASP.NET Core it has become the default monitoring stack: Microsoft itself routes Application Insights through the OTel exporter (`Azure.Monitor.OpenTelemetry.AspNetCore`), and the in-box `Activity` / `Meter` APIs *are* the OTel API. The mental model worth carrying: **logs / metrics / traces are three signals describing the same request**, correlated by `TraceId` and `SpanId`. The whole point of OTel is that you stop choosing between them.

## The three signals

| Signal | API | What it records |
|---|---|---|
| **Traces** | `ActivitySource` / `Activity` | Spans of work (HTTP request → DB call → outbound API). Each span has parent, attributes, events, status. |
| **Metrics** | `Meter` / `Counter<T>` / `Histogram<T>` | Aggregatable numeric data (RPS, p99 latency, queue depth). Pre-aggregated in process. |
| **Logs** | `ILogger<T>` | Discrete events — exactly the existing MEL pipeline, exported as OTel logs. |

Logs and traces correlate automatically when both are wired: every log emitted inside an active `Activity` carries `TraceId`/`SpanId` automatically (the framework adds them via `Activity.Current`).

## Wiring all three

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService(serviceName: "orders-api", serviceVersion: "1.4.2")
        .AddAttributes([new("deployment.environment", builder.Environment.EnvironmentName)]))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("Orders.*")                    // your custom ActivitySource names
        .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.10)))
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Orders.*")
        .AddOtlpExporter())
    .WithLogging(l => l
        .AddOtlpExporter(),
        options => options.IncludeFormattedMessage = true);
```

Three things to notice:

1. **One Resource describes the service** (`service.name`, `service.version`, `deployment.environment`). Every signal carries it. This is what your back-end groups by.
2. **Instrumentation packages auto-emit** standard spans/metrics for ASP.NET Core, HttpClient, EF Core, gRPC, Runtime, etc. You don't write them — you turn them on.
3. **OTLP is the export protocol** — gRPC by default (`http://otel-collector:4317`) or HTTP/protobuf (`http://otel-collector:4318/v1/{traces|metrics|logs}`). Configure once via `OTEL_EXPORTER_OTLP_ENDPOINT`; every signal uses it.

## Custom traces — `ActivitySource`

```csharp
public sealed class OrderService(IOrderRepo repo)
{
    private static readonly ActivitySource _activity = new("Orders.OrderService");

    public async Task PlaceAsync(Order o, CancellationToken ct)
    {
        using var span = _activity.StartActivity("PlaceOrder");
        span?.SetTag("order.id", o.Id);
        span?.SetTag("order.customer", o.CustomerId);

        try { await repo.SaveAsync(o, ct); }
        catch (Exception ex)
        {
            span?.SetStatus(ActivityStatusCode.Error, ex.Message);
            span?.AddException(ex);
            throw;
        }
    }
}
```

`ActivitySource` is **`System.Diagnostics`** — first-party in .NET. The OTel SDK *subscribes* to it. That means you don't take an OTel dependency in domain code; you sprinkle `ActivitySource` and let the host wire it up.

## Custom metrics — `Meter`

```csharp
public sealed class CheckoutMetrics
{
    private static readonly Meter _meter = new("Orders.Checkout", "1.0");
    public static readonly Counter<long> OrdersPlaced = _meter.CreateCounter<long>("orders.placed", unit: "{order}");
    public static readonly Histogram<double> CheckoutLatency = _meter.CreateHistogram<double>("orders.checkout.duration", unit: "ms");
}

CheckoutMetrics.OrdersPlaced.Add(1, new("currency", order.Currency));
CheckoutMetrics.CheckoutLatency.Record(elapsedMs, new("payment.method", "card"));
```

Same story: `Meter` is in `System.Diagnostics.Metrics`, the OTel SDK subscribes via the meter name registered in `AddMeter("Orders.*")`. Tag values become **dimensions** in your back-end (Prometheus labels, App Insights customDimensions, etc.). Watch cardinality — see gotchas.

## Sampling

Tracing every request at 100% is expensive and rarely useful. Common shapes:

| Sampler | Behaviour |
|---|---|
| `AlwaysOnSampler` | Trace everything. Dev only. |
| `AlwaysOffSampler` | Trace nothing. Useful as a kill-switch. |
| `TraceIdRatioBasedSampler(0.1)` | 10% of root traces. Random by trace id. |
| `ParentBasedSampler(child)` | Honour upstream's sampling decision; for roots, defer to `child`. **Default in production.** |

Production default is `ParentBased(Ratio(0.05–0.20))`. Tail-based sampling (sample only errors / slow requests) requires the **OTel Collector** with the `tail_sampling` processor — not done in-process.

## Logs through OTel

`WithLogging(...)` adds an OTel exporter to the MEL pipeline. Every `ILogger<T>.LogX(...)` flows out as an OTel log record with `TraceId`/`SpanId` already attached if there's an active `Activity`. This is how you replace Application Insights' legacy SDK for logs.

```csharp
builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;
    o.IncludeScopes = true;
    o.ParseStateValues = true;       // structured properties become OTel log attributes
});
```

You can run OTel logs **and** Serilog/JsonConsole simultaneously — the back-end might want OTLP, the platform stdout shipper might still want JSON. They don't fight.

## OTel Collector — the agent in front of vendors

```
App ──OTLP──► Collector ──► Tempo (traces) / Prometheus (metrics) / Loki (logs)
                       └──► Azure Monitor / Datadog / Honeycomb / …
```

The collector is a separate binary that receives OTLP and fans out. Why bother:

- **Vendor swap is config.** Ship app once, change the collector to retarget.
- **Tail-based sampling.** `tail_sampling` processor sees the full trace before deciding.
- **Batching, retry, redaction, attribute mutation** — all out-of-process so your app doesn't pay.
- **Multiple back-ends** in parallel without doubling your in-app exporters.

For small deployments, exporting OTLP directly to the back-end (Tempo, Honeycomb, App Insights) is fine. At scale, run the collector.

## Application Insights via OTel

```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor();   // Azure.Monitor.OpenTelemetry.AspNetCore
```

That single call wires logs/metrics/traces to App Insights with sensible defaults. It's the **recommended path** going forward — the legacy `Microsoft.ApplicationInsights.AspNetCore` SDK is in maintenance mode. Combining `UseAzureMonitor()` with `.AddSource(...)` / `.AddMeter(...)` for your custom names gets you everything.

## When to use OTel

- Greenfield observability — start here. Don't pick a vendor SDK first; pick OTel and an OTLP-speaking back-end.
- Multi-cloud / multi-vendor deployments — vendor-neutral export.
- You want logs/traces/metrics correlated by `TraceId` for free.

## When NOT to (yet)

- Single-stack legacy already on App Insights SDK — migration has cost, payoff is mostly future-proofing.
- You only need logs to one stdout shipper — [microsoft-logging](./microsoft-logging.md)'s `JsonConsole` is enough.

## Senior-level gotchas

- **`AddSource("Foo.*")` matches by name; you must register every prefix you emit from.** A custom `ActivitySource("Payments.Checkout")` is silently dropped if your registration only includes `AddSource("Orders.*")`. Same for `AddMeter`.
- **Metric tag cardinality is a billing meteor.** A `Histogram.Record(ms, new("user.id", userId))` with millions of users explodes the time-series count — providers charge per series. Tag with **bounded** dimensions (status, route, payment.method); never with user/order/request id.
- **`ParentBased(Ratio(x))` propagates parent decision through `traceparent` header.** If an upstream sampled, you sample; if it didn't, you don't. This means edge services control sampling; downstream "I want 100% on this service" doesn't work without a `ParentBased` override or `AlwaysOn` for that service.
- **EF Core instrumentation captures SQL by default but not parameter values.** `SetDbStatementForText = true` includes the SQL; **don't** enable parameter capture in production — PII risk. The `db.statement` attribute can also blow span size; consider redacting at the collector.
- **Span attributes cap at 128 by default** (configurable via `SpanLimits`). Beyond that they're dropped silently. Don't dump entire request bodies as attributes.
- **`Activity.Current` is `AsyncLocal`** — flows across `await` correctly. But starting an activity in a fire-and-forget `Task.Run(...)` without explicit parenting loses the link; use `_source.StartActivity("name", ActivityKind.Internal, parentContext)` to stitch.
- **OTLP gRPC vs HTTP** — gRPC (port 4317) is the default; HTTP (port 4318) is the firewall-friendly fallback. Cloud Run and similar PaaS sometimes ban arbitrary gRPC; pick HTTP and pay a small CPU tax.
- **Resource attributes are immutable per-app.** `service.name` is set at startup and applies to every signal; you can't change it per-request. For per-request grouping, use span attributes / log properties.
- **Logs through OTel and Serilog can both export — don't double-count.** If you export `Serilog.Sinks.OpenTelemetry` AND enable `Logging.AddOpenTelemetry()`, every log goes out twice. Pick one path: either MEL → OTel directly, or MEL → Serilog → OTel sink.
- **Histogram bucket boundaries default to a generic set** that may not fit your latency profile. Override with `View(...)` to set explicit buckets (e.g. `[5, 10, 25, 50, 100, 250, 500, 1000, 2500]` ms) or your p99 will be coarse and useless.
- **`AddRuntimeInstrumentation()` (from `OpenTelemetry.Instrumentation.Runtime`) is a freebie** — emits GC counts, thread pool queue, exception rate, working set. Always include it; runtime regressions show up here before they show up in business metrics.
- **`Activity.RecordException` / `AddException` doesn't set the span status** — it just attaches an event. Set `SetStatus(ActivityStatusCode.Error, ex.Message)` explicitly, otherwise dashboards counting "errored spans" miss yours.
