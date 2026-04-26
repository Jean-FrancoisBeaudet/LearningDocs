# Health Checks

_Targets .NET 10 / C# 14. See also: [hosted-service](./hosted-service.md), [background-service](./background-service.md), [opentelemetry](../LOGGING_AND_MONITORING/opentelemetry.md)._

Health checks are how an orchestrator (Kubernetes, App Service, ECS, a load balancer) decides whether your process is **alive**, **ready to serve**, or **healthy** without you having to invent your own protocol. ASP.NET Core ships `Microsoft.Extensions.Diagnostics.HealthChecks` with a tiny abstraction (`IHealthCheck`) and the community provides 40+ ready-made checks (DB, Redis, RabbitMQ, S3, Azure Service Bus, …) under the `AspNetCore.HealthChecks.*` package family. The contract that orchestrators read is dead simple: **HTTP 200 = healthy, 503 = unhealthy**. Everything else (the JSON body, the durations, the data dictionaries) is for humans, dashboards, and alerting.

## Minimum setup

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("db", tags: ["ready"])
    .AddRedis(redisConn, name: "redis", tags: ["ready"])
    .AddCheck<DiskSpaceCheck>("disk", tags: ["ready"]);

var app = builder.Build();

app.MapHealthChecks("/health/live", new()
{
    Predicate = _ => false                       // skip every registered check; just answer "the process responds"
});

app.MapHealthChecks("/health/ready", new()
{
    Predicate = c => c.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.Run();
```

Two endpoints — **liveness** (am I a healthy process?) and **readiness** (am I ready to take traffic?). Tags are how you filter checks per endpoint.

## Custom check

```csharp
public sealed class DiskSpaceCheck(IOptionsMonitor<DiskSpaceOptions> opt) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct = default)
    {
        var drive = new DriveInfo(opt.CurrentValue.Drive);
        var freeGb = drive.AvailableFreeSpace / 1_073_741_824d;

        var data = new Dictionary<string, object> { ["freeGb"] = freeGb, ["drive"] = drive.Name };

        return Task.FromResult(freeGb switch
        {
            < 1  => HealthCheckResult.Unhealthy($"Disk full ({freeGb:F1} GB free)", data: data),
            < 5  => HealthCheckResult.Degraded($"Disk low ({freeGb:F1} GB free)", data: data),
            _    => HealthCheckResult.Healthy($"OK ({freeGb:F1} GB free)", data)
        });
    }
}

builder.Services.AddHealthChecks().AddCheck<DiskSpaceCheck>("disk", tags: ["ready"]);
```

`HealthCheckResult` has three states — **Healthy**, **Degraded**, **Unhealthy** — plus optional `description`, `exception`, and a `data` dictionary surfaced in the JSON response. The HTTP status code is derived from the worst result among checks evaluated for that endpoint.

## Tags & predicate — the liveness/readiness split

```csharp
app.MapHealthChecks("/health/live", new() { Predicate = _ => false });        // returns 200 if the process answers
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
app.MapHealthChecks("/health/startup", new() { Predicate = c => c.Tags.Contains("startup") });
```

| Endpoint | Probes | Checks included |
|---|---|---|
| `/health/live` | K8s `livenessProbe` | none — just "the process responds at all" |
| `/health/ready` | K8s `readinessProbe` | DB, cache, message broker, anything you must reach to serve |
| `/health/startup` | K8s `startupProbe` | one-time migrations, warm-up tasks |

The reason liveness checks **nothing** in most apps: if liveness depends on the DB and the DB blips, K8s restarts every replica simultaneously, your DB now sees a thundering herd of reconnects, and one transient failure becomes an outage. Liveness is "is the process wedged?", readiness is "should we route traffic to it?".

## JSON response writer

The default `ResponseWriter` writes `Healthy` / `Unhealthy` as plain text. For human-friendly JSON, install `AspNetCore.HealthChecks.UI.Client` and use:

```csharp
app.MapHealthChecks("/health/ready", new()
{
    Predicate = c => c.Tags.Contains("ready"),
    ResponseWriter = HealthChecks.UI.Client.UIResponseWriter.WriteHealthCheckUIResponse
});
```

```json
{
  "status": "Unhealthy",
  "totalDuration": "00:00:00.0421",
  "entries": {
    "db":    { "status": "Healthy",   "duration": "00:00:00.0030" },
    "redis": { "status": "Unhealthy", "description": "timeout", "duration": "00:00:00.0391" }
  }
}
```

## HealthChecks UI dashboard

`AspNetCore.HealthChecks.UI` provides a polling dashboard that aggregates multiple services:

```csharp
builder.Services
    .AddHealthChecksUI(s => s.AddHealthCheckEndpoint("orders", "https://orders.internal/health/ready"))
    .AddInMemoryStorage();                       // or AddSqlServerStorage / AddPostgresStorage

app.MapHealthChecksUI(o => o.UIPath = "/health-ui");
```

Useful for a small fleet; for anything large, push the data to your existing observability stack (Grafana, Datadog) instead of running a separate dashboard.

## Push-based publishing

For pushing health into App Insights / Datadog / a metrics endpoint without an external poller:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("db");

builder.Services.AddSingleton<IHealthCheckPublisher, ApplicationInsightsPublisher>();
builder.Services.Configure<HealthCheckPublisherOptions>(o =>
{
    o.Period = TimeSpan.FromSeconds(30);
    o.Timeout = TimeSpan.FromSeconds(10);
    o.Predicate = c => c.Tags.Contains("ready");
});
```

Publisher runs as a hosted service inside your app, polling registered checks on a timer. **Set the period on the publisher**, not by hammering the HTTP endpoint with a sidecar — running the same checks twice is wasteful and may cause connection-pool churn.

## Kubernetes wiring

```yaml
livenessProbe:
  httpGet:    { path: /health/live,  port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:    { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2
startupProbe:
  httpGet:    { path: /health/startup, port: 8080 }
  periodSeconds: 5
  failureThreshold: 30                          # gives 150s for migrations to finish
```

The startup probe gates the other two — once it passes, K8s switches to live/ready. Use it for slow boot scenarios; without it, a slow startup looks like a liveness failure and the pod gets killed in a loop.

## Senior-level gotchas

- **Liveness should never depend on dependencies**. Checking the DB in `/health/live` causes cascading restarts — when the DB blips, every replica's liveness fails together, K8s kills them all, the survivors face a reconnect storm. Keep liveness as `Predicate = _ => false` unless you know exactly why you're widening it.
- **The contract is the status code**, not the JSON body. Orchestrators read 200 vs 503; everything else is for humans. Don't put business signals in the body and expect K8s to react.
- **`Degraded` returns 200 by default**. Configure `ResultStatusCodes` if you want degraded readiness to take a pod out of rotation: `o.ResultStatusCodes[HealthStatus.Degraded] = StatusCodes.Status503ServiceUnavailable`.
- **Probes have short timeouts** (often 1s). A check that does a DB query without a `CancellationToken` or with a slow connection blocks the entire probe response. Always plumb `ct` and use a tight `using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct); cts.CancelAfter(...)`.
- **Don't repeat work in checks**: an `AddRedis` check that opens a new `ConnectionMultiplexer` per probe is a connection-storm waiting to happen. Resolve the same singleton your app uses; the community packages already do this if you register the connection in DI first.
- **Health checks run synchronously per endpoint hit** unless you use a publisher. Heavy probes from a load balancer can compete with real traffic — set the LB's polling interval (or use an `OutputCache` for the response) accordingly.
- **Auth on `/health/*`**: liveness and readiness must be reachable without auth (the orchestrator has no token). Use `[AllowAnonymous]` (controllers) or `.RequireAuthorization()` only on UI/dashboard endpoints, not the probe endpoints.
- **Versioning the response shape**: the default writer writes plain text; the UI writer writes JSON. Switching versions without updating consumers (Datadog parsers, log scrapers) breaks dashboards silently. Treat the response shape as a public contract.
- **`AddDbContextCheck` runs `CanConnectAsync()`**, which opens a connection but doesn't query. That's fine for readiness; if you want a deeper check (e.g. a query against a critical table) write a custom check — but every health-probe second is a second of DB time you're paying for.
- **Don't include health endpoints in your OpenAPI document** — they show up in client codegen and confuse consumers. Exclude with `.ExcludeFromDescription()` (minimal API) or by gating the API explorer.
