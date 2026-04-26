# Microsoft DI

_Targets .NET 10 / C# 14. See also: [scrutor-for-assembly-scanning](./scrutor-for-assembly-scanning.md), [scrutor-for-decorators](./scrutor-for-decorators.md), [configuration-and-options-pattern](../ASPNET_CORE_BASICS/configuration-and-options-pattern.md), [middlewares](../ASPNET_CORE_BASICS/middlewares.md), [hosted-service](../ASPNET_CORE_BASICS/hosted-service.md), [background-service](../ASPNET_CORE_BASICS/background-service.md)._

`Microsoft.Extensions.DependencyInjection` (MEDI) is the first-party container that ships in every ASP.NET Core, Worker Service, and `Microsoft.Extensions.Hosting` app. It is **deliberately minimal** — three lifetimes, constructor injection, no auto-registration, no property injection, no interception. That minimalism is a feature: it forces predictable behaviour at composition time, makes the container fast and AOT-friendly, and keeps the abstraction (`IServiceCollection` / `IServiceProvider`) tiny enough that other containers (Autofac, Lamar, Simple Injector) plug in cleanly via `UseServiceProviderFactory`. You only need to swap when you genuinely need module-scoped registration, decorators (which [Scrutor](./scrutor-for-decorators.md) can also give you), property injection, or runtime container introspection.

## The three lifetimes

| Lifetime | Instance per | Disposed by container? |
|---|---|---|
| `Singleton` | Process | Yes, on app shutdown |
| `Scoped` | `IServiceScope` (in ASP.NET Core: per HTTP request) | Yes, when the scope disposes |
| `Transient` | Every resolution | Yes, when the **owning scope** disposes |

The trap nobody warns you about: **transient `IDisposable` services are tracked by the scope that resolved them**. Resolve a transient `HttpClient` 1,000 times inside a long-lived singleton scope and you accumulate 1,000 references that the GC can't reclaim until the scope ends. For per-request transients this is fine. For long-lived consumers (singleton, root-scope, background services) it leaks until process exit.

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
builder.Services.AddTransient<IEmailSender, SmtpSender>();
```

ASP.NET Core opens a fresh `IServiceScope` per HTTP request (`HttpContext.RequestServices`). Outside of a request — in a `BackgroundService`, a hosted service, a queue consumer — **you must open your own scope** for each unit of work, otherwise scoped services share state across all your work items:

```csharp
public sealed class OrderProcessor(IServiceScopeFactory scopes, ILogger<OrderProcessor> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        await foreach (var msg in _queue.ReadAllAsync(stop))
        {
            await using var scope = scopes.CreateAsyncScope();
            var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();
            await uow.HandleAsync(msg, stop);
        }
    }
}
```

## Registration APIs

```csharp
// Closed-generic shorthand
services.AddScoped<IRepo, EfRepo>();

// Factory — any custom construction logic
services.AddScoped<IRepo>(sp =>
    new EfRepo(sp.GetRequiredService<AppDb>(), sp.GetRequiredService<IClock>()));

// Pre-built instance — always singleton-like
services.AddSingleton<IClock>(SystemClock.Instance);

// Open generic — one registration covers every closed form
services.AddSingleton(typeof(ICache<>), typeof(MemoryCache<>));

// Multiple registrations of the same service — IEnumerable<T> resolves all
services.AddScoped<IValidator<Order>, OrderRequiredFieldsValidator>();
services.AddScoped<IValidator<Order>, OrderBusinessRulesValidator>();
```

For bulk convention-based registration (every `*Handler` in an assembly, every implementation of `IRequestHandler<,>`, etc.), use [Scrutor](./scrutor-for-assembly-scanning.md) — MEDI itself has no scanning API.

## Resolution

Constructor injection is the only blessed path. With a primary constructor (C# 12+) the boilerplate disappears entirely:

```csharp
public sealed class OrdersController(IOrderRepo repo, ILogger<OrdersController> log)
    : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<IActionResult> Get(int id, CancellationToken ct)
    {
        log.LogInformation("get {OrderId}", id);
        return await repo.FindAsync(id, ct) is { } o ? Ok(o) : NotFound();
    }
}
```

Per-action resolution (only when the dep is genuinely action-specific):

```csharp
public async Task<IActionResult> Export([FromServices] IExporter exporter, CancellationToken ct) { ... }

// Minimal API — same idea
app.MapGet("/x", ([FromKeyedServices("primary")] IDb db) => db.QueryAsync());
```

Resolving `IServiceProvider` directly is a service-locator smell — see further down.

## Keyed services (.NET 8+)

When you have multiple implementations of the same interface and the consumer needs to pick by name, use keyed registrations rather than `IEnumerable<T>` + filter:

```csharp
services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");
services.AddKeyedSingleton<INotifier, WebhookNotifier>("webhook");

public sealed class AlertService([FromKeyedServices("email")] INotifier email,
                                  [FromKeyedServices("sms")]   INotifier sms) { ... }
```

Resolution by `string`, `enum`, or any `IEquatable<T>` key. `IKeyedServiceProvider.GetKeyedService(typeof(T), key)` is the imperative API; `[FromKeyedServices]` is the constructor/parameter attribute. There's no `IEnumerable<INotifier>` projection across keys — keyed and non-keyed registrations are different sets.

## TryAdd / Replace / RemoveAll

For library code that registers default implementations the host can override:

```csharp
public static IServiceCollection AddMyLib(this IServiceCollection s)
{
    s.TryAddSingleton<IClock, SystemClock>();          // skip if already registered
    s.TryAddEnumerable(ServiceDescriptor.Scoped<IRule, DefaultRule>());  // skip *this exact descriptor* if already in the IEnumerable
    return s;
}

// In the host
services.AddMyLib();
services.Replace(ServiceDescriptor.Singleton<IClock, FrozenClock>()); // swap the lib's default
services.RemoveAll<IRule>();                                          // wipe everything for this service
```

`Add*` blindly appends — call it twice and you get two registrations. `TryAdd*` is what library extension methods should use. `TryAddEnumerable` is the only way to safely contribute *one* item to a multi-binding without producing duplicates on repeated registration.

## Validation

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes  = true;   // throws on captive dependency
    o.ValidateOnBuild = true;   // throws at startup if any registered service can't be constructed
});
```

`ValidateScopes` defaults to `true` only in Development. **Turn it on everywhere.** The cost is negligible and it catches captive dependencies before they ship. `ValidateOnBuild` walks every registration at startup, attempting construction — slow on large containers but invaluable in CI ("the wiring at least compiles").

## Disposal

The container disposes anything **it constructed** that implements `IDisposable` or `IAsyncDisposable`. It does **not** dispose pre-built instances you handed it via `AddSingleton<T>(instance)` — you own those.

```csharp
services.AddSingleton<IDriver, Driver>();             // container disposes on shutdown
services.AddSingleton<IDriver>(new Driver(...));      // you own this; container leaves it alone
services.AddSingleton<IDriver>(_ => new Driver(...)); // factory result IS owned by the container
```

`IAsyncDisposable` is honoured: `await scope.DisposeAsync()` (or letting `await using var scope = ...` go out of scope) calls `DisposeAsync` if available, falling back to `Dispose`.

## Captive dependency

A singleton holding a scoped service captures it for the process lifetime — every "fresh" request uses the captured instance.

```csharp
services.AddSingleton<TenantCache>();    // wrong: TenantCache holds AppDbContext
services.AddScoped<AppDbContext>();
// First request opens AppDbContext, TenantCache caches it forever, every subsequent request
// hits a disposed DbContext on the second EF call.
```

`ValidateScopes = true` throws `InvalidOperationException` on the offending `GetService` call, telling you exactly which singleton is trying to consume which scoped dep. Fix: inject `IServiceScopeFactory` and open a scope per call, or move the dep to singleton if its real lifetime is process-wide.

## Service-locator anti-pattern

Injecting `IServiceProvider` everywhere defeats the point of DI: dependencies become invisible at the type signature, tests need a real container, and lifetime mistakes hide.

Legitimate uses are narrow:
- **Factories**: a class whose dependency is only known at runtime (`switch(kind) => sp.GetRequiredKeyedService<T>(kind)`). Even here, prefer keyed services or a typed `Func<TKey, T>` factory.
- **Conditional resolution at startup**: `if (services.BuildServiceProvider().GetService<IMetrics>() is null) ...` — but use `IServiceProviderIsService` instead, which doesn't allocate a temporary container.
- **Plug-in / late-bound scenarios**: legitimately open-ended.

Everywhere else, list real dependencies in the constructor.

## Container swaps

`Microsoft.Extensions.DependencyInjection.Abstractions` is the seam: `IServiceCollection` for registration, `IServiceProvider` for resolution. Plug in another implementation via `UseServiceProviderFactory`:

```csharp
// Autofac — modules, property injection, child scopes, registration sources
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
builder.Host.ConfigureContainer<ContainerBuilder>(c => c.RegisterModule<MyModule>());

// Lamar — Roslyn-compiled resolution, scanning conventions, WhatDoIHave() diagnostics
builder.Host.UseServiceProviderFactory(new LamarServiceProviderFactory());

// Simple Injector — fast verification at composition root
// (different integration story; uses an explicit container plus a small bridge)
```

| Container | Pick when |
|---|---|
| **MEDI** | Default. AOT/trimming friendly, fastest cold start, smallest dep. |
| **Autofac** | You need modules (encapsulated registration packs), property injection, child scopes, registration sources, or rich lifetime scopes. |
| **Lamar** | You want StructureMap-style scanning + container introspection (`WhatDoIHave`, `WhatDidIScan`) and Roslyn-compiled resolution. |
| **Simple Injector** | You want strict verification (every registration provably resolvable in tests) and zero-magic explicit configuration. |

Most teams never need to swap. If your only reason is "I want decorators and assembly scanning," [Scrutor](./scrutor-for-decorators.md) gives you both on top of MEDI without the migration tax.

## Senior-level gotchas

- **`IOptions<T>` is singleton; `IOptionsSnapshot<T>` is scoped; `IOptionsMonitor<T>` is singleton with change notifications.** Inject the wrong one and config "doesn't reload." See [configuration-and-options-pattern](../ASPNET_CORE_BASICS/configuration-and-options-pattern.md).
- **`ActivatorUtilities.CreateInstance<T>(sp, args)`** constructs a type that isn't registered, mixing positional `args` with DI-resolved deps. Indispensable for factories that accept runtime parameters; the container does **not** track or dispose the result — you own it.
- **`Task.Run` inside a request loses the scope.** The new task runs on a thread-pool thread with no `HttpContext.RequestServices`. Capture what you need before the offload, or open an explicit scope inside the task.
- **`GetService<T>()` returns `null` for unregistered services; `GetRequiredService<T>()` throws.** Prefer `GetRequiredService` everywhere except genuine optional lookups — silent `null` deps surface 200 lines away from the missing registration.
- **`IHttpContextAccessor` injected into a singleton is a captive dependency in disguise.** The accessor itself is singleton-safe (it stores `HttpContext` in an `AsyncLocal`), but reading `accessor.HttpContext` from a background thread or after the request completes returns `null`. Don't store `accessor.HttpContext` in a field.
- **`IServiceProviderIsService` lets you check whether a service is registered without resolving it.** Use it for conditional library wiring (`if (sp.IsService<ITelemetry>()) ...`) instead of `BuildServiceProvider()`-and-probe, which allocates a parallel container and fragments singletons.
- **Calling `services.BuildServiceProvider()` inside `ConfigureServices` builds a *second* root container.** Singletons resolved from it are different instances from the host's. The error message "BuildServiceProvider was called from application code" is a warning sign — refactor to `IConfigureOptions<T>` or a deferred factory.
- **Open generics + closed registration interaction**: registering `ICache<>` open *and* `ICache<User>` closed gives you two resolutions for `IEnumerable<ICache<User>>`. The closed one wins for `GetService<ICache<User>>()` (last-write-wins on the descriptor list). Be explicit about which you want.
- **Disposable transients in singletons leak.** If you must inject a disposable transient into a singleton, inject a `Func<T>` or `IServiceScopeFactory` and resolve+dispose locally instead.
- **`HttpClient` is *not* a transient you should `AddTransient` directly.** Use `AddHttpClient<T>` ([httpclient-and-httpclientfactory](../API_COMMUNICATION/httpclient-and-httpclientfactory.md)) — the factory manages handler pooling and DNS refresh; raw transient `HttpClient` exhausts sockets.
- **`AddHostedService<T>` registers as singleton.** A hosted service that wants scoped deps must take `IServiceScopeFactory` and create scopes per work item — see the `OrderProcessor` example above. The compiler will not warn you.
- **AOT / NativeAOT**: MEDI itself is AOT-safe, but `MakeGenericType`-style scanning (Scrutor, MediatR) is not without `[DynamicallyAccessedMembers]` annotations. If you trim, prefer source-generated registration helpers and skip scanning at runtime.
