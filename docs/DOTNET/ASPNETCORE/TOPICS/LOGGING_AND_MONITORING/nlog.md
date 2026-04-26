# NLog

_Targets .NET 10 / C# 14. See also: [logging](../ASPNET_CORE_BASICS/logging.md), [microsoft-logging](./microsoft-logging.md), [serilog](./serilog.md), [opentelemetry](./opentelemetry.md), [seq](./seq.md), [elastic-search](./elastic-search.md)._

`NLog` is the other major third-party logger for .NET — older than Serilog, still actively maintained, and **config-file-first**. Where Serilog leans on fluent C# wiring, NLog's idiomatic shape is `nlog.config` XML with declarative rules and targets. It plugs into `Microsoft.Extensions.Logging` via `NLog.Extensions.Logging` (`UseNLog()` for the host) so application code keeps writing `ILogger<T>`. Pick NLog when an existing project already uses it, when ops controls logging via config without a deploy, or when the team prefers XML rules to fluent code.

## Wiring

```csharp
using NLog;
using NLog.Web;

var logger = LogManager.Setup()
    .LoadConfigurationFromAppSettings()  // reads "NLog" section, or nlog.config
    .GetCurrentClassLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    builder.Logging.ClearProviders();
    builder.Host.UseNLog();              // routes ILogger<T> through NLog targets

    var app = builder.Build();
    app.MapGet("/", () => "ok");
    app.Run();
}
catch (Exception ex) { logger.Fatal(ex, "Stopped due to exception"); throw; }
finally { LogManager.Shutdown(); }       // flushes async wrappers
```

`UseNLog()` (from `NLog.Web.AspNetCore`) replaces the MEL provider chain; without `ClearProviders()` you get NLog *plus* the built-in providers running side by side. `LogManager.Shutdown()` is the equivalent of Serilog's `CloseAndFlush()` — required if you use `AsyncWrapper`.

## `nlog.config` — the canonical shape

```xml
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      throwConfigExceptions="true"
      internalLogLevel="Warn"
      internalLogFile="logs/nlog-internal.log">

  <extensions>
    <add assembly="NLog.Web.AspNetCore" />
    <add assembly="NLog.Targets.Seq" />
  </extensions>

  <targets async="true">
    <target xsi:type="File" name="file"
            fileName="logs/app-${shortdate}.log"
            archiveEvery="Day" maxArchiveFiles="14"
            layout="${longdate}|${level:uppercase=true}|${logger}|${message} ${exception:format=tostring}" />

    <target xsi:type="Console" name="console">
      <layout xsi:type="JsonLayout" includeAllProperties="true" includeScopeProperties="true">
        <attribute name="ts"      layout="${longdate}" />
        <attribute name="level"   layout="${level:uppercase=true}" />
        <attribute name="logger"  layout="${logger}" />
        <attribute name="msg"     layout="${message}" />
        <attribute name="trace"   layout="${aspnet-traceidentifier}" />
        <attribute name="ex"      layout="${exception:format=tostring}" />
      </layout>
    </target>

    <target xsi:type="BufferingWrapper" name="seqBuf" bufferSize="500" flushTimeout="2000">
      <target xsi:type="Seq" serverUrl="http://seq:5341" apiKey="${configsetting:Seq.ApiKey}" />
    </target>
  </targets>

  <rules>
    <logger name="Microsoft.AspNetCore.*" maxLevel="Info" final="true" />
    <logger name="*" minlevel="Info" writeTo="console,seqBuf,file" />
  </rules>
</nlog>
```

Three concepts run NLog:

- **Targets** — where logs go (File, Console, Seq, ES, Network, Database, Mail, etc.). Compose via `BufferingWrapper`, `AsyncWrapper`, `RetryingWrapper`, `FallbackGroup`.
- **Layouts** — how each event is rendered. `JsonLayout` is the production default; raw `${longdate}|${level}|${message}` is the legacy text shape.
- **Rules** — which loggers (categories) write to which targets, with min/max levels. `final="true"` stops further rule evaluation for matching loggers (used to drop noisy categories).

## Structured logging mode

NLog defaults to old-style positional logging. To get **message-template / structured** behaviour matching `[LoggerMessage]` and Serilog:

```xml
<nlog parseMessageTemplates="true">
```

Now `logger.LogInformation("Placing order {OrderId} for {Customer}", id, customer)` populates `OrderId` / `Customer` as named properties (visible in `JsonLayout` via `includeAllProperties="true"` or via `${event-properties:item=OrderId}`). Without this flag NLog falls back to `string.Format`-style numeric placeholders — silently breaking structured log search.

## ASP.NET Core layout renderers

`NLog.Web.AspNetCore` registers extra renderers that read `HttpContext`:

| Renderer | Output |
|---|---|
| `${aspnet-request-method}` | `GET`, `POST`, … |
| `${aspnet-request-url}` | full URL |
| `${aspnet-request-ip}` | client IP |
| `${aspnet-traceidentifier}` | `HttpContext.TraceIdentifier` |
| `${aspnet-user-identity}` | authenticated user name |
| `${activity:property=TraceId}` | OpenTelemetry trace id (via `NLog.DiagnosticSource`) |

These are **layout-time** values — evaluated when the event is rendered, not when it's queued. Combined with `AsyncWrapper`, that means HttpContext must still be valid when the queue drains; for high latency before flush, capture the value into a property at log time:

```csharp
logger.LogInformation("{TraceId} Placing order", HttpContext.TraceIdentifier);
```

## Async and reliability wrappers

```xml
<target xsi:type="AsyncWrapper" name="fileAsync" overflowAction="Block" queueLimit="10000">
  <target xsi:type="File" ... />
</target>
```

| Wrapper | Effect |
|---|---|
| `AsyncWrapper` | Background thread drains the queue; `overflowAction` = `Discard` / `Grow` / `Block`. Default is `Discard` — events are silently dropped under load. |
| `BufferingWrapper` | Batches writes by size or interval (used for network sinks like Seq, ES, HTTP). |
| `RetryingWrapper` | Retries on target exception N times with backoff. |
| `FallbackGroup` | Try target A, fall through to B if A fails. |
| `AutoFlushWrapper` | Force-flush on each event matching a condition (e.g. `${level} >= Error`). |

The composition is the point: `Async → Retry → Fallback(ES, File)` is a reasonable shape — async for throughput, retry for transient ES blips, fallback to local file for "ES is dead."

## When to use NLog

- Existing project already uses it — migration to Serilog is rarely worth the churn.
- Ops wants to change logging behaviour (targets, levels, filters) by editing `nlog.config` without rebuild/redeploy. NLog watches the config file by default (`autoReload="true"`).
- You like XML/declarative config better than fluent C#.
- You need targets NLog has and Serilog doesn't (rare these days; both ecosystems are huge).

## When NOT to use NLog

- New greenfield project — Serilog has more momentum, more sinks, better OTel integration via `Serilog.Sinks.OpenTelemetry`. Pick Serilog unless something specific argues otherwise.
- All-in OpenTelemetry shop — log via OTel directly (see [opentelemetry](./opentelemetry.md)) rather than via NLog → OTLP, which is one extra hop.

## Senior-level gotchas

- **`parseMessageTemplates="true"` is OFF by default.** Without it, `{OrderId}` is treated as positional — your "structured" logs are flat strings and search by field doesn't work. First thing to check on any "we're using NLog and structured logging is broken" call.
- **Default `AsyncWrapper.overflowAction="Discard"` silently drops events under load.** Change to `Block` (back-pressure on caller) or grow the queue with bounded `queueLimit`. Pair with `internalLogLevel="Warn"` so NLog tells you when it dropped events.
- **`autoReload="true"` reloads config on file change** — handy for ops, but a malformed edit kills logging entirely until corrected. Always set `throwConfigExceptions="true"` so misconfiguration fails the host instead of degrading silently.
- **`internalLogLevel` is your only insight when targets fail.** Set it to `Warn` and an `internalLogFile` path; this is NLog's equivalent of Serilog's `SelfLog`. Otherwise Seq being down looks identical to "no events generated."
- **Target wrappers are evaluated outside-in** but render the layout inside-in — meaning `${aspnet-*}` renderers inside an `AsyncWrapper` evaluate on the **background thread**, where `HttpContext` may already be gone. Either copy the value into a logged property at call site, or use NLog's `${aspnet-*}` carefully (most are designed for this and capture eagerly).
- **`final="true"` on a rule stops further rule evaluation for matching loggers.** Used to drop noisy categories, but easy to misuse — putting `final="true"` on a high-up rule silently disables every rule below it for that category.
- **NLog and Serilog cannot share a host.** `UseNLog()` and `UseSerilog()` both replace the MEL provider chain — calling both leaves whichever wins last. Pick one back-end.
- **`LogManager.Shutdown()` matters** — async/buffering wrappers hold events. Without flush on shutdown, the last few hundred ms of logs vanish, which usually includes the fatal exception you care about.
- **`JsonLayout` with `includeAllProperties="true"` doesn't include `LogContext`-style scope by default — you need `includeScopeProperties="true"`** (and `IncludeScopes` enabled in MEL). Easy to miss; scope data gets dropped silently.
- **NLog's "database target" is real but rarely a good idea.** Logging into the same DB you're serving traffic from couples your error path to your hot path — an outage that fills the log table can take down the app. Keep logs on a different store.
- **The package split is non-obvious**: core `NLog`, MEL bridge `NLog.Extensions.Logging`, web bits `NLog.Web.AspNetCore`. Forgetting `NLog.Web.AspNetCore` means `${aspnet-*}` renderers silently render as empty.
