# Middlewares

_Targets .NET 10 / C# 14. See also: [filters-and-attributes](./filters-and-attributes.md), [error-handling](./error-handling.md), [routing](./routing.md), [cors](../SECURITY/cors.md), [authentication-and-authorization](./authentication-and-authorization.md)._

Middleware **is** the request pipeline. ASP.NET Core has nothing else: authentication, routing, CORS, static files, MVC — every one of them is a middleware registered into the same pipeline. The mental model is a stack of `Func<HttpContext, Task>` delegates wrapped one inside the next; a request descends through `await next()` calls and the response unwinds back up. Order is the entire game — get it wrong and auth runs before routing knows which endpoint matched, or static files leak past authorization, or `UseExceptionHandler` can't catch what threw before it.

## The pipeline

```csharp
var app = builder.Build();

app.Use(async (ctx, next) =>
{
    Console.WriteLine("→ inbound");
    await next();                       // hand control downstream
    Console.WriteLine("← outbound");    // runs after the response is built
});

app.Run(ctx => ctx.Response.WriteAsync("hello"));   // terminal — never calls next
```

Two things to internalise:

- `Use` adds a middleware that **may** call `next` (or short-circuit by writing a response and returning).
- `Run` is terminal — pipeline ends here. It's just `Use` without a `next` parameter.

Anything you do *after* `await next()` runs on the way back up — perfect for logging response status, adding response headers (only if the response hasn't started — see gotchas), measuring elapsed time.

## Three authoring shapes

```csharp
// 1. Inline lambda — fine for one-off logic
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Trace"] = ctx.TraceIdentifier;
    await next();
});
```

```csharp
// 2. Convention-based class — singleton, deps via Invoke parameters
public sealed class TimingMiddleware(RequestDelegate next, ILogger<TimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext ctx, IClock clock)   // scoped/transient deps land here
    {
        var start = clock.UtcNow;
        await next(ctx);
        logger.LogInformation("{Path} took {Ms}ms", ctx.Request.Path, (clock.UtcNow - start).TotalMilliseconds);
    }
}

app.UseMiddleware<TimingMiddleware>();
```

```csharp
// 3. IMiddleware — DI-resolved per request, scoped lifetime supported
public sealed class TenantMiddleware(ITenantResolver resolver) : IMiddleware
{
    public async Task InvokeAsync(HttpContext ctx, RequestDelegate next)
    {
        ctx.Items["tenant"] = await resolver.ResolveAsync(ctx, ctx.RequestAborted);
        await next(ctx);
    }
}

builder.Services.AddScoped<TenantMiddleware>();
app.UseMiddleware<TenantMiddleware>();
```

The convention-based class is **a singleton** — its constructor runs once. Putting a `DbContext` in the constructor silently captures it for the process lifetime. Either accept scoped deps as `InvokeAsync` parameters (the framework resolves them per-request) or use `IMiddleware`.

## Use vs Run vs Map vs MapWhen vs UseWhen

| API | Behaviour |
|---|---|
| `Use` | Continues pipeline (may call `next`). |
| `Run` | Terminal. Ignores anything registered after it. |
| `Map("/admin", branch)` | Branches the pipeline on path prefix; the branch has its own pipeline. |
| `MapWhen(predicate, branch)` | Branches on any predicate. The branch is a **separate pipeline** — middlewares before the branch don't carry over unless you register them inside too. |
| `UseWhen(predicate, branch)` | Conditionally inserts middlewares into the **main** pipeline; rejoins after. Use this when you want most of the pipeline to keep running. |

```csharp
app.UseWhen(ctx => ctx.Request.Path.StartsWithSegments("/api"), api =>
{
    api.UseMiddleware<RateLimitMiddleware>();
});
```

## Canonical order

```csharp
app.UseExceptionHandler("/error");      // outermost — must wrap everything that can throw
app.UseHsts();                          // prod only
app.UseHttpsRedirection();
app.UseStaticFiles();                   // before routing — short-circuits on file hit, no endpoint match needed
app.UseRouting();                       // populates ctx.GetEndpoint()
app.UseCors("default");                 // after routing so CORS sees endpoint metadata; before auth
app.UseAuthentication();                // identifies the user
app.UseAuthorization();                 // decides whether the identified user may proceed
app.UseAntiforgery();                   // .NET 8+ — required for forms with [ValidateAntiForgeryToken]
app.MapControllers();                   // (or MapEndpoints / MapRazorPages)

app.Run();
```

Why this order:

- `UseExceptionHandler` must be first to catch downstream throws. It re-executes the pipeline against the error path; if it weren't outermost, exceptions from `UseRouting` or auth would escape.
- `UseStaticFiles` before `UseRouting` is a perf choice: serving `wwwroot/site.css` shouldn't pay the routing cost. Trade-off: static files run **before** auth — don't put secrets in `wwwroot`.
- `UseCors` after `UseRouting` lets CORS read per-endpoint policy via endpoint metadata; before `UseAuthentication` so preflight `OPTIONS` succeeds without credentials.
- `UseAuthentication` only **identifies** (`HttpContext.User`); `UseAuthorization` enforces. They are not interchangeable and the order matters.

`UseRouting` and `UseEndpoints` are now implicit when you use `Map*` directly on `WebApplication`, but if you call `UseRouting` explicitly, **`UseAuthorization` must be between `UseRouting` and the endpoint mapping**, otherwise endpoint metadata isn't yet available to the policy evaluator.

## Short-circuiting

Don't call `next` and write a response — that ends the pipeline cleanly:

```csharp
app.Use(async (ctx, next) =>
{
    if (!ctx.Request.Headers.ContainsKey("X-Api-Key"))
    {
        ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
        await ctx.Response.WriteAsync("missing api key");
        return;                          // do NOT call next
    }
    await next();
});
```

Once you've called `next`, you can no longer change `StatusCode` or headers if `Response.HasStarted` is `true`. Set headers and status **before** `await next()`, or use `OnStarting` to register a callback that fires just before the response is flushed.

## Middleware vs filters vs endpoint filters

| Concern | Tool |
|---|---|
| Cross-cutting at the **HTTP** boundary (auth, compression, exception mapping, CORS, rate limit) | **Middleware** |
| MVC-aware concern needing access to action descriptor, model state, or `IActionResult` | **Filter** ([filters-and-attributes](./filters-and-attributes.md)) |
| Per-endpoint cross-cut on a minimal API endpoint | **`IEndpointFilter`** |

Middleware sees an `HttpContext` — it doesn't know which controller action will eventually run (until after `UseRouting`, via `ctx.GetEndpoint()`). Filters know about MVC's model. Use the lowest-knowledge tool that does the job.

## Senior-level gotchas

- **`UseStaticFiles` after auth**: anonymous static assets stop loading. The fix is almost always to keep static files before auth and put truly protected files behind an authenticated endpoint, not `wwwroot/`.
- **Response already started**: writing headers (including via `OnStarting`) after `Response.HasStarted == true` throws. Capture state before `await next()` if you'll need it on the way back.
- **Convention-based middleware DI**: the constructor is singleton. Scoped deps land in `InvokeAsync` parameters — never the constructor. The compiler won't catch the mistake; you'll just see captured-instance bugs in production.
- **`UseRouting` without `UseAuthorization`**: when you bring routing back in explicitly, you must also place `UseAuthorization` between routing and endpoint mapping, or per-endpoint policies are silently ignored.
- **Body buffering for inspection**: reading the request body in middleware consumes it. Call `ctx.Request.EnableBuffering()` first; for the response, swap `ctx.Response.Body` with a `MemoryStream`, capture, then copy back to the original — and do it inside a `try/finally` to restore on exception.
- **Calling `next` twice**: the pipeline is not idempotent. Calling it twice will re-run downstream middleware against a half-written response and usually throw `InvalidOperationException`.
- **Async fire-and-forget after `next`**: `_ = SomeAsync(ctx)` after `await next()` races with the response being sent and the request being aborted. The framework will dispose `HttpContext` once the response completes; your "background" task crashes on a disposed context. Use a hosted service / channel for that work — see [hosted-service](./hosted-service.md).
- **`Map` creates a separate pipeline**: middlewares registered before `app.Map(...)` do **not** apply inside the branch. Re-register or prefer `UseWhen`.
- **Endpoint metadata is null before routing**: `ctx.GetEndpoint()` returns `null` until `UseRouting` runs. Middleware that needs endpoint metadata (CORS, authorization) must be ordered after it.
