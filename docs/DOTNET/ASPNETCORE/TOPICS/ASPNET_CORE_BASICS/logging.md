# Logging

_Targets .NET 10 / C# 14. See also: [microsoft-logging](../LOGGING_AND_MONITORING/microsoft-logging.md), [serilog](../LOGGING_AND_MONITORING/serilog.md), [nlog](../LOGGING_AND_MONITORING/nlog.md), [opentelemetry](../LOGGING_AND_MONITORING/opentelemetry.md), [seq](../LOGGING_AND_MONITORING/seq.md), [loki](../LOGGING_AND_MONITORING/loki.md), [elastic-search](../LOGGING_AND_MONITORING/elastic-search.md)._

`Microsoft.Extensions.Logging` is **the abstraction**, not a logger. The framework writes to `ILogger<T>`, you choose providers (Console, JsonConsole, Serilog, NLog, OpenTelemetry, Application Insights, Seq, Loki). The right hot-path shape is **structured logging via source-generated `[LoggerMessage]`** — message templates with named placeholders, zero allocations on disabled levels, compile-time validation. Two rules carry most of the value: never `string.Format` a log line, and always pass exceptions as the first argument so the provider can serialize the stack trace.

## Injection and categories

```csharp
public sealed class OrderService(ILogger<OrderService> logger, IOrderRepo repo)
{
    public async Task PlaceAsync(Order o, CancellationToken ct)
    {
        logger.LogInformation("Placing order {OrderId} for {Customer}", o.Id, o.CustomerId);
        await repo.SaveAsync(o, ct);
    }
}
```

`ILogger<T>` resolves a logger whose **category** is the fully-qualified type name. Category drives provider routing and `appsettings.json` filtering — every provider can include/exclude by category prefix.

## Log levels

| Level | Production meaning |
|---|---|
| `Trace` | Method entry/exit, parameter dumps. **Off** in prod. |
| `Debug` | Diagnostic detail useful during a triage session. **Off** in prod by default. |
| `Information` | Routine business events: "Order placed", "User signed in". |
| `Warning` | Something unusual but recoverable: "retry attempt 2", "deprecated API used". |
| `Error` | Operation failed; no recovery, but the process keeps running. |
| `Critical` | Catastrophe — host should stop or page someone. |
| `None` | Filter sentinel; no log produced. |

Don't promote everything to `Warning` to make sure it's seen — that's how you train a team to ignore the warning channel. `Information` is the default; `Warning+` should be alertable.

## Structured logging — message templates, not interpolation

```csharp
// Wrong — string is pre-formatted, providers can't index OrderId.
logger.LogInformation($"Placing order {o.Id} for {o.CustomerId}");

// Wrong — same problem, plus the message text is now unique per order.
logger.LogInformation("Placing order " + o.Id);

// Right — message template, structured fields.
logger.LogInformation("Placing order {OrderId} for {Customer}", o.Id, o.CustomerId);
```

Why: providers like Seq, Loki, App Insights, ES capture `OrderId` and `Customer` as **searchable, typed fields**. Interpolation throws that away — the providers only see one big string per call site, deduplication in dashboards collapses, and you can't `where OrderId = 42`. The CA1848 / CA2254 analyzers will flag interpolation; turn them on.

Placeholder names are **PascalCase** by convention. The order in the format string must match the argument order — there's no name-based binding at runtime.

## Source-generated `[LoggerMessage]`

```csharp
public sealed partial class OrderService(ILogger<OrderService> logger, IOrderRepo repo)
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
                   Message = "Placing order {OrderId} for {Customer}")]
    private partial void LogOrderPlacing(long orderId, string customer);

    [LoggerMessage(EventId = 1002, Level = LogLevel.Error,
                   Message = "Order {OrderId} save failed")]
    private partial void LogSaveFailed(Exception ex, long orderId);

    public async Task PlaceAsync(Order o, CancellationToken ct)
    {
        LogOrderPlacing(o.Id, o.CustomerId);
        try { await repo.SaveAsync(o, ct); }
        catch (Exception ex) { LogSaveFailed(ex, o.Id); throw; }
    }
}
```

What you get:

- **No boxing** of value-type args (the template-string overload boxes everything to `object`).
- **No allocations** when the level is disabled — the generated method checks `IsEnabled` first.
- **Compile-time validation**: the placeholder count and types must match the method parameters; mismatches are build errors.
- **EventId** is a stable handle for dashboards / suppression rules — assign deliberately and don't reshuffle.

For a hot path (per-request logging, tight loops) this is the production-grade shape. The interpolation-style overload is fine for slow paths and one-off info logs.

## Scopes — propagating context

```csharp
using (logger.BeginScope("CorrelationId={CorrelationId}, Tenant={Tenant}",
                         ctx.TraceIdentifier, ctx.GetTenant()))
{
    logger.LogInformation("Looking up account");
    await accounts.LoadAsync(id, ct);          // any logs inside here inherit the scope
}
```

Scopes propagate through `AsyncLocal` — they flow across `await`, but only providers that opt in (Serilog enriches, the JSON console writes them as a `Scopes` array, the plain Console provider drops them unless `IncludeScopes = true` is configured). Use scopes for **request-scoped** context (correlation id, tenant, user) and let middleware set them once at the top of the pipeline.

ASP.NET Core's `RequestLoggingMiddleware` already adds a scope with `RequestId`/`SpanId`/`TraceId` when OpenTelemetry is wired — see [opentelemetry](../LOGGING_AND_MONITORING/opentelemetry.md).

## Configuration

```jsonc
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    },
    "Console": {
      "FormatterName": "json",
      "FormatterOptions": { "IncludeScopes": true, "TimestampFormat": "O" }
    }
  }
}
```

Filter precedence: most-specific category prefix wins. `Microsoft.EntityFrameworkCore.Database.Command: Warning` silences the famously chatty EF query logs without turning down the rest of EF.

Per-provider rules go under `Logging:<ProviderAlias>:LogLevel` (e.g. `Logging:Console:LogLevel:Default`).

## Choosing a provider

| Provider | Use when |
|---|---|
| **Console / JsonConsole** | Local dev; containerized prod where stdout is collected by the platform (K8s, App Service, Azure Container Apps). |
| **[Serilog](../LOGGING_AND_MONITORING/serilog.md)** | You want enrichers, sinks, dynamic level reload, and a mature ecosystem. Most common third-party choice. |
| **[NLog](../LOGGING_AND_MONITORING/nlog.md)** | Legacy projects already using it; rule-based config. |
| **[OpenTelemetry Logs](../LOGGING_AND_MONITORING/opentelemetry.md)** | Unified telemetry — logs correlated with traces and metrics. Use as a *secondary* provider alongside Serilog/Console. |
| **[Seq](../LOGGING_AND_MONITORING/seq.md)** | Centralised structured-log search for small/medium fleets. |
| **[Loki](../LOGGING_AND_MONITORING/loki.md)** | Grafana-native; cheap storage; label-based querying — beware label cardinality. |
| **[Elastic Search](../LOGGING_AND_MONITORING/elastic-search.md)** | Full-text + structured search at scale; ELK or Elastic Cloud. |
| **Application Insights** (`Microsoft.ApplicationInsights.AspNetCore`) | Azure-first; live metrics, distributed traces, kusto. |

Use the framework's abstraction in your code (`ILogger<T>`) — switching providers is a registration change, not a rewrite.

## Performance & hot paths

```csharp
// 1. Check IsEnabled before expensive arg construction
if (logger.IsEnabled(LogLevel.Debug))
    logger.LogDebug("Payload: {Json}", JsonSerializer.Serialize(payload));   // serialization skipped if Debug is off

// 2. Better: source-generated LoggerMessage handles this for you
[LoggerMessage(Level = LogLevel.Debug, Message = "Payload: {Json}")]
private partial void LogPayload(string json);
// caller:
LogPayload(JsonSerializer.Serialize(payload));   // still pays Serialize cost — pass the object instead and let a custom formatter handle it
```

Real wins come from `[LoggerMessage]` (no boxing) plus **not constructing the log payload at all when the level is off**. For tight loops, log a counter, not every iteration.

## Senior-level gotchas

- **Pass `Exception` as the first arg, not `ex.Message`**: `logger.LogError(ex, "Save failed for {OrderId}", id)` writes the full stack to providers that support it. `logger.LogError("Save failed: " + ex.Message)` loses the stack and breaks structured search.
- **Interpolation kills structured logging**: `$"Order {id}"` produces a unique message string per call → providers can't aggregate. Analyzer **CA2254** catches this; enable `<AnalysisMode>All</AnalysisMode>` and treat it as an error.
- **High-cardinality fields blow up label/index storage**: `OrderId`, `UserId`, `RequestId` as Loki labels will explode the label index — use them as *fields* (Loki: log line content) or just send to Seq/ES. App Insights customDimensions are fine for cardinality; Loki labels are not.
- **PII redaction is a provider concern, not the caller's**: write a `LoggerFilter` / Serilog enricher that scrubs by field name (`Email`, `Ssn`, `CardNumber`). Asking developers to remember in every call site fails by lunch.
- **`BeginScope` doesn't flow without provider support**: the plain Console provider drops scopes by default. Set `IncludeScopes = true` or use `JsonConsole` / Serilog. Don't assume scopes appear just because the API exists.
- **`EventId` uniqueness matters**: `[LoggerMessage(EventId = 1001)]` defines a dashboard-stable handle. Reusing the same `EventId` across two methods or shuffling them between releases breaks alert rules and saved searches.
- **Default Console provider is sync**: in throughput-heavy services, switch to `JsonConsole` with the system collecting from stdout, or pipe through Serilog with an async sink. Sync writes serialise threads.
- **`ILoggerFactory.CreateLogger("CategoryName")` works** but loses the strongly-typed category. Prefer `ILogger<T>` so refactoring renames update the category for free.
- **Don't log inside formatters or `Dispose`** when the provider may already be shutting down — log lines emitted during host shutdown can be silently dropped.
- **Trace ID correlation**: with OpenTelemetry wired, the framework adds `TraceId`/`SpanId` to every log scope automatically. Don't roll your own correlation id middleware; use `Activity.Current?.TraceId` or just enable OTel — see [opentelemetry](../LOGGING_AND_MONITORING/opentelemetry.md).
- **Log level reload**: `Microsoft.Extensions.Logging` re-reads `appsettings.json` via `IOptionsMonitor` change tokens — bumping `Microsoft.AspNetCore` to `Debug` in a running process works without restart, provided your provider honours change tokens (Serilog needs `ReadFrom.Configuration` to be wired with reload tracking).
