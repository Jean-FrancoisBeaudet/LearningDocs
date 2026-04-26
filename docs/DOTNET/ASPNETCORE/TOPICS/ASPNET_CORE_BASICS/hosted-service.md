# Hosted Services

_Targets .NET 10 / C# 14. See also: [background-service](./background-service.md), [health-checks](./health-checks.md), [hangfire](../TASK_SCHEDULING/hangfire.md), [quartz](../TASK_SCHEDULING/quartz.md), [tickerq](../TASK_SCHEDULING/tickerq.md)._

`IHostedService` is the lifetime hook the .NET Generic Host gives you for **long-running work that the host owns** — message consumers, periodic jobs, queue drainers, warm-up routines. The host calls `StartAsync` once at boot and `StopAsync` once at shutdown; what you do in between is up to you. `BackgroundService` is the helper subclass you should reach for 90% of the time — it manages the long-running task for you so all you write is `ExecuteAsync`. In an ASP.NET Core app, the same host runs both Kestrel and your hosted services, which means a slow `StartAsync` blocks request serving and an unhandled exception inside `ExecuteAsync` can take the whole process down.

## `IHostedService` contract

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);   // boot
    Task StopAsync (CancellationToken cancellationToken);   // shutdown (timeout = HostOptions.ShutdownTimeout, default 30s)
}
```

```csharp
builder.Services.AddHostedService<MyWorker>();              // singleton lifetime
```

`StartAsync` runs sequentially, in registration order, **before the host signals "started"**. Anything you `await` here delays Kestrel starting to accept requests. Two correct shapes:

- **Quick startup work** (open a connection, register with a coordinator): `await` it inside `StartAsync`.
- **Long-running loops**: don't `await` them in `StartAsync` — kick them off and return immediately, or just use `BackgroundService` (preferred).

## `BackgroundService` — the right default

```csharp
public sealed class OutboxRelay(
    IServiceScopeFactory scopes,
    ILogger<OutboxRelay> logger,
    TimeProvider time)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5), time);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();
                var sender = scope.ServiceProvider.GetRequiredService<IOutboxSender>();
                await sender.FlushAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { /* shutdown */ }
            catch (Exception ex)
            {
                logger.LogError(ex, "Outbox flush failed");                 // swallow → keep loop alive
            }
        }
    }
}

builder.Services.AddHostedService<OutboxRelay>();
```

What `BackgroundService` does for you:

- Returns from `StartAsync` synchronously after spawning `ExecuteAsync` on the thread pool — Kestrel boots immediately.
- Wires `stoppingToken` to host shutdown.
- On `StopAsync`, signals `stoppingToken`, then awaits `ExecuteAsync` to complete (subject to `HostOptions.ShutdownTimeout`).

Use `PeriodicTimer` (allocates once, await-friendly) over `Task.Delay(...)` in a loop — the latter creates a new `Task` per tick.

## DI scopes from a singleton

`AddHostedService<T>` registers `T` as a **singleton**. Direct injection of `DbContext`, `IUnitOfWork`, anything **scoped** is wrong — you'd capture one scope for the process lifetime. Use `IServiceScopeFactory` and open a fresh scope per work-item:

```csharp
public sealed class CommandConsumer(IServiceScopeFactory scopes, IConnection rabbit) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var msg in rabbit.ConsumeAsync(stoppingToken))
        {
            await using var scope = scopes.CreateAsyncScope();
            var handler = scope.ServiceProvider.GetRequiredService<ICommandHandler>();
            await handler.HandleAsync(msg, stoppingToken);
        }
    }
}
```

One scope per message → matches the per-request scope that an HTTP handler would have, keeps `DbContext` lifetime short and predictable.

## Startup ordering

```csharp
builder.Services.AddHostedService<DbMigrator>();    // 1. Migrate schema first
builder.Services.AddHostedService<CacheWarmup>();   // 2. Warm caches
builder.Services.AddHostedService<OutboxRelay>();   // 3. Start workers
```

`StartAsync` runs in registration order; `StopAsync` runs in reverse. The migrator must `await` its work in `StartAsync` (you want failures to abort boot); the workers should be `BackgroundService`s so their loops don't block subsequent registrations.

In an ASP.NET Core web app, **Kestrel binds in `IServer.StartAsync`** which itself is a hosted service registered by the framework. Your hosted services and Kestrel are siblings. By default they all start in parallel after the registration phase — so a slow `StartAsync` on your worker won't delay Kestrel binding, but Kestrel will start receiving traffic before your warm-up has finished. If that's a problem, surface readiness through [health-checks](./health-checks.md) and configure the orchestrator's readiness probe to gate traffic.

## Graceful shutdown

```csharp
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(60);
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});
```

On SIGTERM (or Ctrl-C, or `IHostApplicationLifetime.StopApplication()`):

1. The host's `StoppingToken` fires.
2. Each hosted service's `StopAsync` is called in reverse registration order.
3. For `BackgroundService`, `StopAsync` triggers `stoppingToken` then awaits `ExecuteAsync`.
4. After `ShutdownTimeout`, the process exits whether services finished or not — log loudly if you couldn't drain in time.

```csharp
public sealed class OutboxRelay(IHostApplicationLifetime lifetime, …) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try { /* loop */ }
        catch (FatalException ex)
        {
            logger.LogCritical(ex, "Outbox died — stopping host");
            lifetime.StopApplication();          // request graceful shutdown
        }
    }
}
```

## Exception behaviour

Default in .NET 6+: an unhandled exception in `ExecuteAsync` triggers `BackgroundServiceExceptionBehavior.StopHost` — the entire process exits. The opposite (`Ignore`) silently kills your worker but keeps the host running, which is **almost never what you want** because you have no signal. The right pattern is to swallow expected failures inside the loop (DB blip, transient network error) and let truly fatal ones tear the host down so the orchestrator restarts you.

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    try { await DoOneIteration(stoppingToken); }
    catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { return; }
    catch (TransientException ex) { logger.LogWarning(ex, "transient — retrying"); }
    // anything else falls through → host stops → K8s restarts the pod
}
```

## Web host vs Worker SDK

- **Web app** (`Microsoft.NET.Sdk.Web`): same host runs Kestrel + hosted services. Hosted services share the request thread pool; back-pressure them so they don't starve HTTP threads.
- **Worker service** (`Microsoft.NET.Sdk.Worker`): no Kestrel, no MVC, no auth — just the host running hosted services. Use this when the service has no HTTP API at all (queue-only consumer, schedule processor).

If a worker also needs an HTTP endpoint (health probe), add `Microsoft.AspNetCore.App` framework reference and `app.MapHealthChecks(...)` — you don't need the full Web SDK.

## When **not** to use a hosted service

| Need | Better tool |
|---|---|
| Cron-style schedules (`0 0 * * *`) | **[Quartz.NET](../TASK_SCHEDULING/quartz.md)** or **[Hangfire](../TASK_SCHEDULING/hangfire.md)** — they own the cron, persistence, and missed-fire logic. |
| Persistent / durable jobs that must survive restarts | **[Hangfire](../TASK_SCHEDULING/hangfire.md)** (SQL/Redis backed) — a hosted service has no persistence. |
| Distributed locking across replicas | **[Quartz](../TASK_SCHEDULING/quartz.md)** clustering or **[TickerQ](../TASK_SCHEDULING/tickerq.md)** — out of the box, every replica runs the loop. |
| Fan-out work to multiple consumers | A real broker (RabbitMQ, Service Bus) with a hosted-service consumer. |

A hosted service is the right tool when you own the loop and there's exactly one of you (or you've already solved coordination elsewhere).

## Senior-level gotchas

- **Don't `await` long work in `StartAsync`**: it delays *every* subsequent hosted service and the host's "started" signal. Kick off background loops; only `await` startup-critical, fast work.
- **Scoped DI from a singleton**: capture `IServiceScopeFactory` (or `IServiceProvider` + manually `CreateScope`) and open a scope per work-item. Constructor-injecting a `DbContext` into a hosted service silently makes it singleton — change tracker, connection, everything — leaking across all "requests".
- **`stoppingToken` vs the parameter to `StopAsync`**: `BackgroundService` exposes the `stoppingToken` field for its loop. The `cancellationToken` passed to `StopAsync` is a *different* token that fires on `HostOptions.ShutdownTimeout` — so during shutdown, both can be triggered. Use `stoppingToken` inside the loop; `BackgroundService.StopAsync` already coordinates them.
- **`PeriodicTimer` is single-consumer**: `WaitForNextTickAsync` cannot be called from multiple awaiters concurrently. One loop, one timer.
- **Multiple replicas all run the loop**: a hosted service has zero coordination. Run two pods, the loop runs twice. Either accept it (idempotent work), use a leader election, or move to a real scheduler.
- **`BackgroundServiceExceptionBehavior.Ignore` hides bugs**: if you set this, you must surface failures elsewhere (metric, alert) or you'll have a "running but doing nothing" replica indefinitely. The default `StopHost` is more honest.
- **HTTP traffic during boot**: in a web app, Kestrel may be accepting requests before your warm-up hosted service has finished. Don't rely on cache being warm in the first request — gate traffic via the readiness probe, not by ordering.
- **Disposable state**: implement `IAsyncDisposable` on the hosted service if it owns connections (RabbitMQ, Kafka, sockets). The host calls `DisposeAsync` after `StopAsync`, but only if the type implements it — otherwise leaked.
- **Logging during shutdown**: console / file log providers may already be flushing during host shutdown. A log line emitted in a `catch` after `stoppingToken` fires can be lost. Log the *intent* before the await, not just on completion.
- **Don't put cron logic in `BackgroundService`**: `Task.Delay(TimeUntilNext3AM())` works on a single replica but breaks under DST transitions, missed fires (host was down at 3 AM), and clustering. Use a scheduler library — see [task-scheduling](../TASK_SCHEDULING/).
