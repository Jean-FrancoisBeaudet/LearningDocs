# Microsoft.Extensions.Logging — built-in providers

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [serilog](./serilog.md), [nlog](./nlog.md), [opentelemetry](./opentelemetry.md), [seq](./seq.md), [loki](./loki.md), [elastic-search](./elastic-search.md)._

`Microsoft.Extensions.Logging` (MEL) is two things in one namespace: **the abstraction** (`ILogger<T>`, `ILoggerFactory`, message templates, scopes — covered in [logging](../ASPNET_CORE_BASICS/logging.md)) and **a small set of first-party providers** that ship with the framework. The providers below are what you get without taking a third-party dependency. For most production workloads you'll combine **JsonConsole** (so the platform's log collector ingests structured JSON) with **EventSource** (so `dotnet-trace`/PerfView can attach in-process). Reach for Serilog/NLog only when you need their sinks or enrichers.

## What `WebApplication.CreateBuilder` already wires

The default host builder calls `ConfigureLogging` and registers, in order:

1. `AddConfiguration` (binds `Logging` section)
2. `AddConsole`
3. `AddDebug`
4. `AddEventSourceLogger`
5. `AddEventLog` (Windows only)

So a fresh app already logs to stdout, the debugger output window, ETW/EventPipe, and the Windows Event Log — without you doing anything. To replace this baseline, call `builder.Logging.ClearProviders()` first.

## The providers

| Provider | Package | Use it for |
|---|---|---|
| `AddConsole` (default) | in-box | Local dev, containers (stdout). **Sync writes** by default. |
| `AddSimpleConsole` | in-box | Same as Console with knobs (`SingleLine`, `ColorBehavior`, timestamps). |
| `AddJsonConsole` | in-box | Production stdout when a log collector parses JSON (K8s, App Service, Container Apps). |
| `AddDebug` | in-box | `System.Diagnostics.Debug.WriteLine` — VS Output window only. Off in Release builds. |
| `AddEventSourceLogger` | in-box | ETW / EventPipe — `dotnet-trace`, PerfView, dotnet-monitor can attach without restart. |
| `AddEventLog` | `Microsoft.Extensions.Logging.EventLog` (in-box on Windows) | Windows Event Log — only for service apps where ops watches Event Viewer. |
| `AddAzureWebAppDiagnostics` | `Microsoft.Extensions.Logging.AzureAppServices` | Azure App Service log streaming and blob/file diagnostics. |
| `AddApplicationInsights` | `Microsoft.Extensions.Logging.ApplicationInsights` | Azure Application Insights (now usually wired via OpenTelemetry — see [opentelemetry](./opentelemetry.md)). |

## JsonConsole — the production default for containers

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddJsonConsole(o =>
{
    o.IncludeScopes      = true;
    o.TimestampFormat    = "O";       // ISO 8601 with offset
    o.UseUtcTimestamp    = true;
    o.JsonWriterOptions  = new JsonWriterOptions { Indented = false };
});
```

Output is one JSON object per line — exactly what Fluent Bit, Vector, the App Service log shipper, or the Loki Docker driver want. Don't invent your own format; the platform side already knows this one.

## EventSource — free in-process telemetry

`AddEventSourceLogger` writes to the `Microsoft-Extensions-Logging` ETW provider. With it on, you can attach with no code change and no restart:

```bash
dotnet-trace collect --name MyApp --providers Microsoft-Extensions-Logging:4:5
```

The `:4:5` is `Keywords:Level` (4 = `FormattedMessage`, 5 = `Information`). This is how senior teams debug prod without flipping config files — keep it on, sample on demand.

## Filtering — the rule that always wins

Filter resolution is **most-specific category prefix wins**, scoped per provider if you specify one. Configuration shape:

```jsonc
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    },
    "Console": { "LogLevel": { "Microsoft": "Information" } }
  }
}
```

That config: most categories at `Information`, ASP.NET Core itself at `Warning`, the famously chatty EF SQL command logger at `Warning` — **except** the Console provider, which gets `Microsoft` at `Information` (overrides the global rule for Console only).

Code-side filter (programmatic, e.g. for tests):

```csharp
builder.Logging.AddFilter("Microsoft.AspNetCore.Routing", LogLevel.Warning);
builder.Logging.AddFilter<ConsoleLoggerProvider>("Microsoft", LogLevel.Information);
```

## Hot-reload of log levels

`IOptionsMonitor<LoggerFilterOptions>` watches `appsettings.json` change tokens. Edit the file in a running container and the filter rules re-evaluate — no restart. This works for the in-box providers and any third-party provider that subscribes to the change token (Serilog needs `ReadFrom.Configuration(... reloadOnChange: true)` plumbing — see [serilog](./serilog.md)).

## Performance characteristics

| Provider | Sync/async | Hot-path safe? |
|---|---|---|
| Console (default) | sync | No — every call serialises threads on `Console.Out`. |
| JsonConsole | sync (writes to stdout buffer) | OK in containers; the kernel buffer absorbs spikes. Still cheaper than Console because no formatting branch logic. |
| Debug | sync, no-op in Release | Free in production. |
| EventSource | async-ish (ETW kernel) | Cheap; designed for high volume. |
| EventLog | sync, P/Invoke | Slow. Don't log per-request. |

For real high-throughput paths, **the bottleneck isn't the provider — it's the message-template overload boxing arguments**. Use source-generated `[LoggerMessage]` (see [logging](../ASPNET_CORE_BASICS/logging.md)) regardless of which provider you pick.

## Custom provider — minimal shape

```csharp
public sealed class MyProvider : ILoggerProvider
{
    public ILogger CreateLogger(string categoryName) => new MyLogger(categoryName);
    public void Dispose() { }
}

public sealed class MyLogger(string category) : ILogger
{
    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => null;
    public bool IsEnabled(LogLevel level) => level >= LogLevel.Information;
    public void Log<TState>(LogLevel level, EventId id, TState state, Exception? ex,
                            Func<TState, Exception?, string> formatter)
        => Console.WriteLine($"[{category}] {formatter(state, ex)}");
}

builder.Logging.AddProvider(new MyProvider());
```

You almost never need this — Serilog/NLog have done it well already. But knowing the shape demystifies what providers actually are: **a factory that returns an `ILogger` per category, that the framework calls into.**

## When to use built-in only

- Containerised app whose platform already collects stdout (K8s, App Service, Container Apps, ECS).
- You want zero third-party dependencies (compliance, supply-chain policy).
- You're shipping to App Insights / OTel anyway — those handle storage/search; built-in just handles emission.

## When to add Serilog/NLog/OTel

- You need rich enrichers, log context, async file rolling, or a non-stdout sink (Seq/Loki/ES/Splunk) — see [serilog](./serilog.md), [nlog](./nlog.md).
- You want unified logs/metrics/traces with correlation — see [opentelemetry](./opentelemetry.md).

## Senior-level gotchas

- **`AddConsole` is synchronous and contended.** Under high RPS it serialises threads on `Console.Out`. Switch to `AddJsonConsole` (cheaper) and let the platform's log shipper buffer.
- **`ClearProviders()` is required when replacing.** Otherwise `UseSerilog()` / `AddOpenTelemetry()` runs *alongside* the defaults — you get duplicate Console output and double the CPU cost.
- **Filter precedence is per-provider, then global.** A category set to `Warning` globally but `Information` for `Console` will log `Information` to Console only. People miss this when "logs disappear after I added App Insights."
- **`AddDebug` is not stripped automatically in Release.** It's just no-op'd in release builds because `Debug.WriteLine` is `[Conditional("DEBUG")]`. Leaving it on in production registration is harmless — but doesn't help either.
- **`AddEventLog` writes to the Application log by default and requires the source to be registered.** First write under a non-admin process fails silently if the source doesn't exist. Pre-register sources at install time (MSI / WiX) or pick a source already created.
- **`AddEventSourceLogger` is a senior's secret weapon.** Keep it enabled in prod — it's nearly free, and gives you `dotnet-monitor` / `dotnet-trace` access to live logs without a redeploy.
- **`Microsoft.AspNetCore` at `Information` is loud.** Every request emits 4–8 lines (routing, auth, model binding, response). Default new-template `appsettings.json` already pins it to `Warning` — keep it that way and let your request-logging middleware (Serilog's `UseSerilogRequestLogging`, OTel's instrumentation) do request summaries.
- **JsonConsole emits scopes only when `IncludeScopes = true`.** Default is false. Without it, your `BeginScope("CorrelationId={Id}", id)` propagates inside ASP.NET Core but never reaches stdout — and you'll wonder why the field isn't in your log shipper.
- **Don't write to multiple providers that all hit stdout.** Console + JsonConsole simultaneously prints each line twice. Pick one console-shaped provider and call it done.
- **Application Insights via the legacy SDK is on the way out.** New apps should wire AI through the OpenTelemetry exporter (`Azure.Monitor.OpenTelemetry.AspNetCore`) — see [opentelemetry](./opentelemetry.md). The `Microsoft.Extensions.Logging.ApplicationInsights` package still works but is in maintenance mode.
