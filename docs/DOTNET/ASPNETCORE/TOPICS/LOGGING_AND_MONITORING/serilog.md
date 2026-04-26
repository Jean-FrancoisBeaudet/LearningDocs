# Serilog

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [microsoft-logging](./microsoft-logging.md), [nlog](./nlog.md), [opentelemetry](./opentelemetry.md), [seq](./seq.md), [loki](./loki.md), [elastic-search](./elastic-search.md)._

`Serilog` is the most-used third-party logger in the .NET ecosystem. It plugs into `Microsoft.Extensions.Logging` as a provider via `Serilog.Extensions.Hosting` (and `Serilog.AspNetCore` for web apps), so application code keeps writing `ILogger<T>` and you swap the back-end. Two reasons people choose it: a huge **sink** ecosystem (Seq, Elasticsearch, Loki, Splunk, Datadog, files, syslog, …) and **enrichers** that decorate every event with machine name, environment, trace ids, request data — without touching call sites.

## The shape

```
ILogger<T>  ──►  MEL pipeline  ──►  Serilog provider  ──►  Pipeline (filter / enrich / write)
                                                            └─► Sinks (Console, File, Seq, Loki, ES, …)
```

Application code stays MEL-shaped. Serilog is the back-end. You only touch `Log.Logger` directly for **bootstrap logging** (errors during host startup, before DI is built).

## Wiring

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();           // captures Program.cs / startup errors

try
{
    var builder = WebApplication.CreateBuilder(args);

    builder.Host.UseSerilog((ctx, sp, cfg) => cfg
        .ReadFrom.Configuration(ctx.Configuration)
        .ReadFrom.Services(sp)           // pull enrichers / destructurers from DI
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName());

    var app = builder.Build();
    app.UseSerilogRequestLogging();      // one structured event per request

    app.MapGet("/", () => "ok");
    app.Run();
}
catch (Exception ex) { Log.Fatal(ex, "Host terminated unexpectedly"); }
finally { Log.CloseAndFlush(); }
```

Three pieces matter:

1. **Bootstrap logger** — `CreateBootstrapLogger()` captures errors that happen before the host starts. Without it, an exception in `WebApplication.CreateBuilder()` produces a silent crash.
2. **`UseSerilog`** — replaces the MEL provider chain. Combined with `builder.Logging.ClearProviders()` (which `UseSerilog` does implicitly in newer versions), only Serilog writes.
3. **`UseSerilogRequestLogging`** — one structured event per request with method, path, status, elapsed ms, and any properties the user attached via `LogContext`. Replaces ASP.NET Core's noisy 4–8-line per-request output.

## Configuration in `appsettings.json`

```jsonc
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.Seq", "Serilog.Sinks.File" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
      }
    },
    "WriteTo": [
      { "Name": "Console", "Args": { "formatter": "Serilog.Formatting.Json.JsonFormatter, Serilog" } },
      { "Name": "Seq",     "Args": { "serverUrl": "http://seq:5341" } },
      { "Name": "File",    "Args": { "path": "logs/app-.log", "rollingInterval": "Day", "retainedFileCountLimit": 14 } }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ],
    "Properties": { "Application": "OrdersApi" }
  }
}
```

`ReadFrom.Configuration` parses this section. Sinks named in `WriteTo` must be referenced in `Using` (or the corresponding NuGet package must be referenced — Serilog probes assemblies by name).

## LogContext — request-scoped properties

```csharp
app.Use(async (ctx, next) =>
{
    using (LogContext.PushProperty("CorrelationId", ctx.TraceIdentifier))
    using (LogContext.PushProperty("Tenant", ctx.GetTenant()))
    {
        await next();
    }
});
```

Anything logged within that scope (any `ILogger<T>.LogX(...)` from any layer) carries `CorrelationId` and `Tenant` as **structured properties** — not interpolated into the message, but as fields the sink can index. This is the killer feature: zero per-call ceremony, every event tagged.

`LogContext` works because `Enrich.FromLogContext()` is in the pipeline. Forget that line and pushed properties vanish.

## Enrichers — adding properties without touching code

| Enricher | Adds | Package |
|---|---|---|
| `FromLogContext` | Properties pushed via `LogContext.PushProperty` | core |
| `WithMachineName` | `MachineName` | `Serilog.Enrichers.Environment` |
| `WithEnvironmentName` | `EnvironmentName` (Development/Staging/Production) | `Serilog.Enrichers.Environment` |
| `WithThreadId` | `ThreadId` | `Serilog.Enrichers.Thread` |
| `WithProcessId` / `WithProcessName` | process metadata | `Serilog.Enrichers.Process` |
| `WithCorrelationIdHeader("X-Correlation-Id")` | request header | `Serilog.Enrichers.CorrelationId` |
| `WithSpan()` | OTel `TraceId`/`SpanId` | `Serilog.Enrichers.Span` |

Enrichers run **once per event in the pipeline** — cheap, and far better than asking developers to remember to add fields per call.

## Filtering — beyond log level

```csharp
.Filter.ByExcluding(e => e.Properties.TryGetValue("RequestPath", out var p) && p.ToString().Contains("/healthz"))
.Filter.ByIncludingOnly(e => e.Level >= LogEventLevel.Warning || e.Properties.ContainsKey("Audit"))
```

You can drop or keep events based on any property. Use this to quiet `/healthz` and `/metrics` polling without raising the global level.

## Destructuring — controlling object expansion

```csharp
logger.LogInformation("Saving {@Order}", order);   // @ → expand object as a structured map
logger.LogInformation("Saving {$Order}", order);   // $ → call ToString()
logger.LogInformation("Saving {Order}", order);    // default — uses ToString() unless a destructurer matches
```

Configure depth and policies globally:

```csharp
.Destructure.ToMaximumDepth(3)
.Destructure.ToMaximumStringLength(1024)
.Destructure.ByTransforming<User>(u => new { u.Id, u.Email })   // PII redaction by type
```

## Dynamic level reload

`ReadFrom.Configuration(ctx.Configuration)` honours `IConfiguration` change tokens. Editing `appsettings.json` in a running container re-applies `MinimumLevel`. For programmatic level changes (e.g. a "verbose mode" toggle), use `LoggingLevelSwitch`:

```csharp
var levelSwitch = new LoggingLevelSwitch(LogEventLevel.Information);
new LoggerConfiguration().MinimumLevel.ControlledBy(levelSwitch);
// later:
levelSwitch.MinimumLevel = LogEventLevel.Debug;   // takes effect immediately, no restart
```

## When to use Serilog

- You need a sink other than stdout: [Seq](./seq.md), [Loki](./loki.md), [Elasticsearch](./elastic-search.md), Splunk, Datadog.
- You want enricher-based metadata (correlation id, tenant, machine name) without per-call boilerplate.
- You're routing the same events to multiple sinks with different filters (e.g. all → Loki, audit-only → S3).
- File logging with rolling, async buffering, and self-log when sinks fail.

## When NOT to use Serilog

- Containerised stdout-only deployment with the platform doing collection — `AddJsonConsole` from [microsoft-logging](./microsoft-logging.md) is enough.
- You're going all-in on [OpenTelemetry](./opentelemetry.md) and the OTLP log exporter — Serilog and OTel logs overlap; pick one as the primary path. (Serilog still has value as the *exporter to OTLP* for sink ergonomics — `Serilog.Sinks.OpenTelemetry`.)

## Senior-level gotchas

- **Forgetting `Enrich.FromLogContext()` silently breaks `LogContext.PushProperty`.** Properties go nowhere; events look bare. Always include it.
- **`UseSerilogRequestLogging` replaces the framework's per-request logs.** If you also leave `Microsoft.AspNetCore` at `Information`, you'll get both — duplicates that confuse log search. Pin `Microsoft.AspNetCore` to `Warning`.
- **Bootstrap logger is mandatory in production.** `Log.Logger = new LoggerConfiguration().CreateBootstrapLogger()` before `WebApplication.CreateBuilder` — otherwise startup exceptions go to a debug-only sink and you debug blind.
- **`Log.CloseAndFlush()` in `finally` is non-negotiable for async sinks.** Sinks that batch (Seq, ES, File) hold buffered events; without flush on shutdown you lose the last seconds of logs — including the fatal error that crashed you.
- **`{@Obj}` destructuring on EF entities explodes.** Tracked entities have circular references and lazy-loading proxies — Serilog will stack-overflow or pull half your DB. Project to a DTO first or use `.Destructure.ByTransforming<TEntity>(...)`.
- **Sink failures are silent unless you wire `SelfLog`.** `Serilog.Debugging.SelfLog.Enable(Console.Error)` writes Serilog's internal errors to stderr. Without it, "Seq is down" looks like "logs are missing."
- **`MinimumLevel.Override` is a prefix match like MEL filters.** `"Microsoft": "Warning"` matches both `Microsoft.AspNetCore` and `Microsoft.EntityFrameworkCore`. Use the most-specific prefix.
- **`Serilog.AspNetCore` ≠ `Serilog.Extensions.Hosting`.** AspNetCore wraps Hosting and adds the request middleware; for non-web hosts use just `Extensions.Hosting`. Don't pull both.
- **`writeTo: { Name: "Async" }` (`Serilog.Sinks.Async`) wraps slow sinks** (File, ES) in a background queue. Without it, a slow sink blocks the calling thread on every event. The async wrapper has its own `bufferSize` — overflow drops events; size it to your event rate.
- **`{Property}` in the message template captures the value only if the type is known to be primitive.** For complex types, use `{@Property}`. For "render `ToString()` but treat as structured", use `{Property:l}` (string literal, no quotes around strings).
- **Don't mix `Log.Information(...)` (static) with `ILogger<T>.LogInformation(...)`.** The static path bypasses MEL category filtering — your category-level overrides (e.g. `Microsoft.AspNetCore: Warning`) won't apply to those calls. Stick to `ILogger<T>` everywhere except bootstrap.
