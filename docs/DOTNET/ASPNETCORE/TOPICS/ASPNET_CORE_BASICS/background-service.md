# BackgroundService

_Targets .NET 10 / C# 14. See also: [HostedService](./hosted-service.md), [Health Checks](./health-checks.md), [Configuration and Options Pattern](./configuration-and-options-pattern.md), [Logging](./logging.md)._

`BackgroundService` is the abstract base class for long-running work hosted alongside an ASP.NET Core (or Worker) process: queue consumers, polling jobs, periodic reconciliation, log shippers. It is **sugar over `IHostedService`**: the base class implements `StartAsync`/`StopAsync` for you and gives you a single `ExecuteAsync(CancellationToken stoppingToken)` to override. Pick `BackgroundService` for "I want to run a loop until shutdown"; pick raw `IHostedService` for "I need fine-grained control over startup/teardown" — see [hosted-service](./hosted-service.md).

## Anatomy

```csharp
public sealed class OutboxPublisher(
    IServiceScopeFactory scopeFactory,
    ILogger<OutboxPublisher> logger) : BackgroundService
{
    private readonly TimeSpan _interval = TimeSpan.FromSeconds(10);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await Task.Yield();                                  // don't block host startup
        using var timer = new PeriodicTimer(_interval);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var publisher = scope.ServiceProvider.GetRequiredService<IOutboxRunner>();
                await publisher.PublishPendingAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;                                       // graceful shutdown
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Outbox publish failed; retrying after delay");
            }

            try { await timer.WaitForNextTickAsync(stoppingToken); }
            catch (OperationCanceledException) { break; }
        }
    }
}

builder.Services.AddHostedService<OutboxPublisher>();        // registers as IHostedService
```

`AddHostedService<T>` registers the type as both `IHostedService` and singleton. The host calls `StartAsync` (sequentially, in registration order) at boot and `StopAsync` (in reverse order) at shutdown.

## `Task.Yield()` at the top of `ExecuteAsync`

The host awaits `StartAsync` synchronously during boot. `BackgroundService.StartAsync` calls `ExecuteAsync` and returns the resulting `Task` — it doesn't `await` it. **But** if `ExecuteAsync` is synchronous up to the first `await`, that synchronous prefix runs on the host startup thread. A 5-second DB connection check at the top of your `ExecuteAsync` delays *every* other hosted service from starting and stalls Kestrel binding.

`await Task.Yield()` forces the rest of the method onto the thread pool and lets the host continue booting. Cheap, idiomatic, never wrong.

## `PeriodicTimer` over `Task.Delay`

```csharp
// Old pattern — works, but drifts and creates a Task per tick
while (!stoppingToken.IsCancellationRequested)
{
    await DoWorkAsync(stoppingToken);
    await Task.Delay(_interval, stoppingToken);
}

// Modern (.NET 6+) — no drift, single allocation
using var timer = new PeriodicTimer(_interval);
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoWorkAsync(stoppingToken);
}
```

`PeriodicTimer` ticks at fixed intervals from the start, regardless of how long the work takes — so a slow run doesn't push subsequent runs further into the future. Critical for "process every minute" semantics; irrelevant for "wait 1 minute between runs."

For variable-interval work (exponential backoff, cron schedules), use `Task.Delay` or libraries like Hangfire/Quartz/TickerQ — see [task-scheduling](../TASK_SCHEDULING/).

## Scoped services from a singleton

`BackgroundService` is a singleton; `DbContext`, repositories, MediatR handlers are usually scoped. Reaching them requires a manual scope per iteration:

```csharp
await using var scope = scopeFactory.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
```

`CreateAsyncScope()` returns an `AsyncServiceScope` whose `DisposeAsync` runs async-disposable services correctly (relevant for Npgsql data sources, EF Core in some scenarios). Plain `CreateScope()` works for sync-disposable graphs.

**Don't capture scoped services in fields.** A captured `DbContext` is reused across iterations, accumulates change-tracking state forever, and will eventually OOM the process. Always create a scope inside the loop body.

## Channels for producer/consumer

For in-process pipelines (HTTP request enqueues work, background service processes), `System.Threading.Channels` beats a hosted-service-with-a-queue rolled by hand:

```csharp
var channel = Channel.CreateBounded<EmailJob>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait
});
builder.Services.AddSingleton(channel);
builder.Services.AddSingleton(_ => channel.Reader);
builder.Services.AddSingleton(_ => channel.Writer);

public sealed class EmailWorker(ChannelReader<EmailJob> reader, IEmailSender sender, ILogger<EmailWorker> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in reader.ReadAllAsync(stoppingToken))
        {
            try { await sender.SendAsync(job, stoppingToken); }
            catch (Exception ex) { log.LogError(ex, "Send failed: {Id}", job.Id); }
        }
    }
}
```

Bounded channels backpressure the producer (HTTP handler awaits `WriteAsync`) when the consumer falls behind — no in-memory queue blowout.

## Graceful shutdown

```csharp
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(60);            // default 30s
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;  // default in .NET 6+
});
```

When `IHost.StopAsync` is called (SIGTERM, container stop, `app.Stop()`):
1. The host signals `stoppingToken` for every `BackgroundService`.
2. The host waits up to `ShutdownTimeout` for *all* `StopAsync` calls to complete.
3. After timeout, the process exits regardless.

Your `ExecuteAsync` must return promptly when `stoppingToken` is cancelled. If you swallow the token (e.g. wrap an inner loop in `try/catch (OperationCanceledException)` and continue), `StopAsync` blocks until `ShutdownTimeout`, after which Kubernetes SIGKILLs you and you lose in-flight work.

## Exception behavior

```csharp
public enum BackgroundServiceExceptionBehavior
{
    StopHost,        // .NET 6+ default — unhandled in ExecuteAsync brings the host down
    Ignore,          // pre-.NET 6 default — silently logs and the service is dead but the app runs
}
```

`StopHost` is correct: a crashed worker that the orchestrator can't see is invisible failure. Let it crash and let Kubernetes/AppService restart you. **Always** wrap your loop body in `try/catch` for *recoverable* errors so transient failures don't bring the host down — but let unrecoverable errors propagate.

## Health checks integration

Surface "is the worker alive and making progress" via a health check, not via `BackgroundService` itself:

```csharp
public sealed class OutboxHealthCheck(IOutboxStateProbe probe) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext _, CancellationToken ct)
    {
        var lag = probe.LastSuccessfulRunAge();
        return Task.FromResult(lag < TimeSpan.FromMinutes(5)
            ? HealthCheckResult.Healthy($"Lag {lag}")
            : HealthCheckResult.Unhealthy($"No successful run in {lag}"));
    }
}

builder.Services.AddSingleton<IOutboxStateProbe, OutboxStateProbe>();
builder.Services.AddHealthChecks().AddCheck<OutboxHealthCheck>("outbox");
```

The state probe is a singleton the worker writes to and the health check reads. See [health-checks](./health-checks.md).

## Multiple background services

```csharp
builder.Services.AddHostedService<OutboxPublisher>();
builder.Services.AddHostedService<EmailWorker>();
builder.Services.AddHostedService<MetricsFlusher>();
```

`StartAsync` is awaited **sequentially in registration order**. If `OutboxPublisher.StartAsync` blocks for 10 seconds (synchronous prefix), `EmailWorker` doesn't start until then. `Task.Yield()` at the top of each `ExecuteAsync` avoids this.

`ExecuteAsync` instances run concurrently — they don't share threads, they don't coordinate. If two services need to coordinate, use a singleton coordinator service injected into both, or a shared `Channel<T>`.

## Senior-level gotchas

- **No `Task.Yield()` at the top of `ExecuteAsync`** is the most common cause of "Kestrel takes 30s to bind during deploy" complaints. The synchronous prefix runs on the host startup thread. Always yield first.
- **Throwing in `ExecuteAsync` brings down the host** (default in .NET 6+). Wrap the *iteration body* in try/catch for transient errors; let truly unrecoverable errors escape so the orchestrator sees the crash and restarts.
- **`stoppingToken` is the host token, not your work-item token.** Plumbing it into every `await` is non-negotiable — without it, an SIGTERM-ed process spends `ShutdownTimeout` waiting for your in-flight DB call to finish on its own clock.
- **Scoped services captured in fields leak across iterations.** A field-held `DbContext` accumulates tracked entities, never releases connections, eventually OOMs. Always `using/await using var scope = scopeFactory.CreateAsyncScope()` *inside* the loop body.
- **`PeriodicTimer.WaitForNextTickAsync(token)` throws `OperationCanceledException` on cancel** — and you almost always want to break the loop, not log it. Catch it explicitly at the timer call site, distinct from your work-item exception handler.
- **`BackgroundService` is a singleton.** Constructor-injected services are resolved at host startup, before the configuration system has finished hot-reloading anything. Use `IOptionsMonitor<T>` rather than `IOptions<T>` if the worker needs to react to config changes — see [configuration-and-options-pattern](./configuration-and-options-pattern.md).
- **`HostOptions.ShutdownTimeout` is per-host, not per-service.** All `StopAsync` calls share the same budget. Ten services that each need 5 seconds to drain need 50 seconds — but the default is 30. Bump it for slow-draining workers (queue consumers committing offsets, EF SaveChanges flushes).
- **`StartAsync` returning a non-completed `Task` blocks the host.** `BackgroundService.StartAsync` is implemented to return the *first iteration's* Task only up to its first await — but if you override `StartAsync`, returning a long-running Task without awaiting it freezes boot. Don't override `StartAsync` unless you know exactly what you're doing.
- **`AddHostedService<T>` registers a singleton for `T`.** Resolving `T` from DI returns the same instance the host owns. That's usually fine but means injecting `OutboxPublisher` from a controller is sharing state with the worker — almost never what you want. Inject a probe/state interface instead.
- **`Environment.Exit(0)` in a worker bypasses host shutdown.** No `StopAsync` is called, in-flight work is abandoned. Use `IHostApplicationLifetime.StopApplication()` for clean shutdown from inside the host.
- **Logging from `ExecuteAsync` after the host stops** can throw — the logger provider may be disposed. Catch logger exceptions at the loop boundary or check `stoppingToken.IsCancellationRequested` before logging.
- **`IHostedService.StartAsync` should return promptly.** The host awaits each one sequentially. A 30-second migration in `StartAsync` blocks every other service and Kestrel. Move slow init *into* `ExecuteAsync` after `Task.Yield()`, or use `IHostApplicationLifetime.ApplicationStarted` to defer it.
