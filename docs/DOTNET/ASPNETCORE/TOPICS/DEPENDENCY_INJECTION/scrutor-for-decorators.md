# Scrutor for Decorators

_Targets .NET 10 / C# 14. See also: [microsoft-di](./microsoft-di.md), [scrutor-for-assembly-scanning](./scrutor-for-assembly-scanning.md), [middlewares](../ASPNET_CORE_BASICS/middlewares.md), [filters-and-attributes](../ASPNET_CORE_BASICS/filters-and-attributes.md), [hybrid-cache](../CACHING/hybrid-cache.md), [output-cache](../CACHING/output-cache.md)._

`Scrutor.Decorate` is the decorator pattern realized as a DI registration. You register the real service, then wrap it: `services.Decorate<IRepo, CachingRepo>()` re-binds `IRepo` to a factory that constructs `CachingRepo` with the **previously-registered** `IRepo` injected as its inner dependency. That gives you onion-style cross-cutting — caching, logging, retries, validation, instrumentation — without modifying the inner type, and without adding a fourth lifetime concept like AOP interception. Same package as [scanning](./scrutor-for-assembly-scanning.md); zero runtime overhead beyond one extra interface call per layer.

## Install

```bash
dotnet add package Scrutor
```

## Basic usage

```csharp
public interface IRepo
{
    ValueTask<Order?> FindAsync(int id, CancellationToken ct);
}

public sealed class EfRepo(AppDb db) : IRepo
{
    public ValueTask<Order?> FindAsync(int id, CancellationToken ct) =>
        new(db.Orders.FindAsync([id], ct).AsTask());
}

public sealed class CachingRepo(IRepo inner, IMemoryCache cache) : IRepo
{
    public async ValueTask<Order?> FindAsync(int id, CancellationToken ct) =>
        await cache.GetOrCreateAsync($"order:{id}", _ => inner.FindAsync(id, ct).AsTask());
}

// Composition root
services.AddScoped<IRepo, EfRepo>();   // 1. register the real one
services.Decorate<IRepo, CachingRepo>(); // 2. wrap it
```

After `Decorate`, resolving `IRepo` gives you a `CachingRepo` whose `inner` field is the original `EfRepo`. Order is critical: **`Decorate` must run after the registration it wraps**, otherwise it throws `DecorationException`. Use `TryDecorate` to make this safe.

## Stacking — onion semantics

```csharp
services.AddScoped<IRepo, EfRepo>();
services.Decorate<IRepo, RetryRepo>();    // wraps EfRepo
services.Decorate<IRepo, LoggingRepo>();  // wraps RetryRepo
services.Decorate<IRepo, CachingRepo>();  // wraps LoggingRepo — outermost
```

A single resolution returns `CachingRepo → LoggingRepo → RetryRepo → EfRepo`. Each `Decorate` call wraps **whatever was registered for `IRepo` at that moment**, so the call order maps to layer order outward. Read the registration block top-to-bottom and the request flows through it that way.

## `TryDecorate`

Library scenario — the consumer may or may not have registered the inner service:

```csharp
services.TryDecorate<IRepo, MetricsRepo>(); // no-op if no IRepo is registered yet
```

Returns `true` if it decorated, `false` otherwise. The strict `Decorate` throws — pick `Decorate` when the missing registration would be a bug, `TryDecorate` when it's optional instrumentation.

## Open-generic decoration

Decorate every closed `IRequestHandler<TQuery, TResult>` with one registration:

```csharp
public sealed class LoggingHandler<TQuery, TResult>(
    IRequestHandler<TQuery, TResult> inner,
    ILogger<LoggingHandler<TQuery, TResult>> log) : IRequestHandler<TQuery, TResult>
{
    public async Task<TResult> HandleAsync(TQuery q, CancellationToken ct)
    {
        log.LogInformation("→ {Query}", typeof(TQuery).Name);
        try { return await inner.HandleAsync(q, ct); }
        finally { log.LogInformation("← {Query}", typeof(TQuery).Name); }
    }
}

services.Scan(s => s.FromAssemblyOf<GetOrderQuery>()
    .AddClasses(c => c.AssignableTo(typeof(IRequestHandler<,>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());

services.Decorate(typeof(IRequestHandler<,>), typeof(LoggingHandler<,>));
```

One line replaces a per-handler MediatR pipeline behavior, and runs against the actual handler call (no MediatR runtime in the path). Combine with [Scrutor scanning](./scrutor-for-assembly-scanning.md) to register handlers and decorators with two short blocks.

## Worked example — three layers on a repository

```csharp
public sealed class RetryRepo(IRepo inner, IAsyncPolicy<Order?> policy) : IRepo
{
    public ValueTask<Order?> FindAsync(int id, CancellationToken ct) =>
        new(policy.ExecuteAsync(t => inner.FindAsync(id, t).AsTask(), ct));
}

public sealed class LoggingRepo(IRepo inner, ILogger<LoggingRepo> log) : IRepo
{
    public async ValueTask<Order?> FindAsync(int id, CancellationToken ct)
    {
        var sw = Stopwatch.GetTimestamp();
        try { return await inner.FindAsync(id, ct); }
        finally { log.LogInformation("FindAsync({Id}) {Ms}ms", id, Stopwatch.GetElapsedTime(sw).TotalMilliseconds); }
    }
}

services.AddScoped<IRepo, EfRepo>();
services.Decorate<IRepo, RetryRepo>();
services.Decorate<IRepo, LoggingRepo>();
services.Decorate<IRepo, CachingRepo>();
```

Resolution order on `repo.FindAsync(42, ct)`:

1. `CachingRepo` — cache hit short-circuits everything below
2. `LoggingRepo` — measures wall-clock including retries
3. `RetryRepo` — Polly retries the EF call
4. `EfRepo` — the only layer that touches the database

If `CachingRepo` were the second `Decorate` instead of the last, you'd be measuring (and retrying) what comes out of the cache — usually the wrong thing. **Outer-most layers should be the cheapest and most short-circuit-prone.**

## Lifetime semantics

The decorator inherits the lifetime of the **decorated** registration. Decorating a singleton produces a singleton decorator; decorating a scoped service produces a scoped decorator. You cannot change lifetime through `Decorate`.

This matters because the decorator's *own* dependencies must be safe at that lifetime. A scoped `IMemoryCache` is process-singleton-friendly, but injecting `IDbContext` (scoped) into a decorator that wraps a singleton service silently captures the `DbContext` for the process — same captive-dependency trap as [microsoft-di](./microsoft-di.md), just hidden one layer down.

## Decorate vs middleware vs filters vs MediatR behaviors

| Cross-cut at | Use |
|---|---|
| HTTP boundary (auth, CORS, compression, exception mapping) | **Middleware** ([middlewares](../ASPNET_CORE_BASICS/middlewares.md)) |
| MVC pipeline (action result, model state, action descriptor) | **Filter** ([filters-and-attributes](../ASPNET_CORE_BASICS/filters-and-attributes.md)) |
| Bus dispatch (every command/query through MediatR / MassTransit) | **Pipeline behavior** (MediatR `IPipelineBehavior<,>`) |
| Direct service call (any consumer of a registered service) | **`Decorate`** |

Pick by where the consumer *enters*. A retry that should apply to every HTTP call belongs in middleware; a retry that should apply to every command sent through a bus belongs in a behavior; a retry that should apply to every call to `IPaymentGateway` regardless of caller belongs in a `Decorate`. The wrong choice means the cross-cut runs in the wrong cases — middleware misses internal calls; `Decorate` misses callers that resolve a different interface.

## Limitations

- **Keyed services**: `Decorate` operates on the non-keyed registration of a service. Decorating a specific keyed registration requires a custom factory or a recent Scrutor version with keyed-decoration support — verify against your installed version.
- **Decorator and decorated must share the interface.** You cannot decorate `EfRepo` with a class that exposes only `IReadRepo<T>` if `IRepo` is the registered type — Scrutor matches by service type.
- **Factory registrations**: decorating `services.AddScoped<IRepo>(sp => new EfRepo(...))` works, but the decorator's constructor sees a `Func<IServiceProvider, IRepo>` resolved each time — no special path. The factory runs every time the decorator is constructed (same as without `Decorate`).
- **No auto-scan of decorators.** You opt in per service. There is no `Scan(...).AsDecorators()` analogue. For a "decorate every handler" pattern, use the open-generic form shown above.

## Senior-level gotchas

- **`Decorate` before the registration throws.** The error (`DecorationException: Could not find any registered services for type IRepo`) is clear, but the fix is "move the `Decorate` call after the registration" — not "swap to `TryDecorate`," which silently no-ops and leaves you wondering why the decorator isn't running.
- **`TryDecorate` returning `false` is silent.** No log, no warning. If your decorator vanishes after a refactor that moved the registration, this is the cause — assert in tests that resolved `IRepo` is the expected outermost decorator type.
- **Stacking order = registration order, outermost last.** Reverse this in your head and you'll measure cached responses, retry on cache misses only, log post-cache — all wrong. Read the registration block as the request path, top to bottom = inner to outer.
- **Decorator constructors run on every resolution at the decorated lifetime.** A scoped decorator allocates per request — usually fine. A transient decorator on a transient service allocates per resolve — every consumer pays. Profile if you stack five+ decorators on a hot path.
- **Generic-variance pitfalls**: an open-generic decorator `Foo<,>` only decorates `IFoo<,>` — it won't decorate covariant or contravariant variants automatically. If you need `IFoo<out T>` covariance, you decorate the closed forms or accept the inner as `IFoo<TOut>` explicitly.
- **`Replace` strategy from Scrutor scanning does NOT compose with Decorate.** A `Replace` registration removes the decorated chain entirely and starts fresh. Decorate then re-runs the chain on the new base — but only if `Decorate` runs *after* the `Replace`. Mixing both in one composition root is a debugging nightmare; pick one.
- **Async disposal must propagate.** If your inner service implements `IAsyncDisposable`, the decorator must implement and call it too — otherwise the container only disposes the outermost layer and the inner connection leaks. Fix: implement both `IDisposable` and `IAsyncDisposable` on the decorator, forward both to `inner`.
- **"Why isn't my decorator running?"** is almost always one of: (1) `Decorate` ran before the inner registration; (2) `TryDecorate` no-op'd; (3) consumer resolves a different interface than the one decorated; (4) keyed-vs-non-keyed mismatch; (5) the scan that registers the inner runs after `Decorate` (e.g. inside an extension method called later). Add a startup assertion: `using var s = sp.CreateScope(); Debug.Assert(s.ServiceProvider.GetRequiredService<IRepo>() is CachingRepo);`.
- **Capturing `inner` in a field is the whole pattern, but capturing it in a static is a bug.** The inner is scoped to the decorator's lifetime — stash it in static state and you'll cross-contaminate scopes/requests.
- **Decorators are visible to the rest of DI as the outermost type.** `IEnumerable<IRepo>` returns one entry — the decorated chain — not all four layers. If you genuinely need to enumerate all layers, you've outgrown decoration; build a composite/strategy explicitly.
