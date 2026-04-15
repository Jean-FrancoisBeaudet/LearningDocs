# ASP.NET Core — Questions 1–5

_Source: `aspnetcore-interview-questions.md`. Answers target .NET 10 / C# 14._

## Q1. Explain how routing works in ASP.NET Core

Routing is the subsystem that maps an incoming HTTP request (method + path) to an **endpoint** — a delegate plus metadata. Since ASP.NET Core 3.0 the model is **endpoint routing**, a two-phase pipeline:

1. `UseRouting()` inspects the request, matches it against the registered endpoint data source, and attaches the selected `Endpoint` to `HttpContext.Features`. No handler is invoked yet.
2. Everything between `UseRouting` and `UseEndpoints` (auth, CORS, rate limiting, output caching) can now read that endpoint's metadata and make policy decisions *before* the handler runs.
3. `UseEndpoints()` (or its implicit equivalent in `WebApplication`) executes the matched endpoint's delegate.

```csharp
var app = builder.Build();

app.UseRouting();          // match phase
app.UseAuthentication();   // sees the endpoint metadata (e.g. [Authorize])
app.UseAuthorization();
app.MapGet("/orders/{id:int:min(1)}", (int id) => Results.Ok(id))
   .WithName("GetOrder");
```

Route templates support **literals**, **parameters** (`{id}`), **catch-all** (`{*path}`), **optional** (`{id?}`), **defaults** (`{id=0}`), and **constraints** (`{id:int:min(1)}`, `{slug:regex(...)}`). Constraints are matching filters, not validation — a constraint mismatch produces a 404, not a 400.

Under the hood, matching uses a **DFA tree** built from all registered endpoints. Ambiguity is resolved by route precedence (literal > constrained > parameter > catch-all) and then by `Order`. If two endpoints tie, you get `AmbiguousMatchException` — a sign you need an explicit `Order` or a more specific template.

**Modern note (.NET 10):** `WebApplication` wires `UseRouting`/`UseEndpoints` automatically; you only call them explicitly to insert middleware between the two phases. Minimal APIs, MVC controllers, Razor Pages, SignalR, gRPC, and Blazor all feed the same endpoint data source, so route conflicts across these stacks are real and diagnosed at startup when you call `app.MapXxx()`. .NET 10 also ships a native **OpenAPI document generator** (`Microsoft.AspNetCore.OpenApi`) that reads endpoint metadata directly from the route table — another reason to add `.WithName`, `.WithTags`, `.Produces<T>()` on your maps.

**Senior-level gotchas:**
- `UseRouting` **must** come before any middleware that reads endpoint metadata; otherwise `HttpContext.GetEndpoint()` returns `null` and `[Authorize]` is silently skipped.
- Route parameter names are matched **case-insensitively** but bound to method parameters **by name** — a typo is silently bound to `default`.
- Catch-all `{*path}` captures the raw, *unescaped* path. Use `{**path}` (double-star) to keep slashes intact.

---

## Q2. What is middleware and in what order do they execute?

Middleware is a component in the HTTP request pipeline with the signature `RequestDelegate(HttpContext) → Task`. Each component receives `HttpContext` and a `next` delegate; it can inspect/mutate the request, invoke `next` to pass control downstream, and inspect/mutate the response on the way back. Conceptually it's the **Russian-doll / chain-of-responsibility** pattern — the pipeline is built once at startup and composed into a single delegate.

```csharp
app.Use(async (ctx, next) =>
{
    // runs on the way IN
    var sw = Stopwatch.StartNew();
    await next();
    // runs on the way OUT
    app.Logger.LogInformation("{Path} took {Ms} ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```

**Order matters** because registration order = execution order. The canonical production pipeline:

```csharp
app.UseExceptionHandler("/error");   // 1. catch everything below
app.UseHsts();                       // 2. HSTS header (prod only)
app.UseHttpsRedirection();           // 3. upgrade http → https
app.UseStaticFiles();                // 4. short-circuit for /wwwroot
app.UseRouting();                    // 5. match endpoint
app.UseCors();                       // 6. after routing so per-endpoint policies work
app.UseRateLimiter();                // 7. reads endpoint metadata
app.UseAuthentication();             // 8. sets ctx.User
app.UseAuthorization();              // 9. enforces policies
app.UseOutputCache();                // 10. after auth so cache keys can include user
app.MapControllers();                // 11. terminal: invoke endpoint
```

Why this order:
- `UseExceptionHandler` is first so it wraps every later component.
- `UseStaticFiles` before `UseRouting` so static asset requests skip routing/auth entirely — that's the performance win.
- `UseCors` / `UseRateLimiter` after `UseRouting` so they can read `[EnableCors]` / `[EnableRateLimiting]` metadata.
- `UseAuthorization` after `UseAuthentication` — you can't authorize a user you haven't identified.

**Modern note (.NET 10):** Output caching, rate limiting, and request decompression are all first-class middleware now; you don't need third-party packages. The **`ProblemDetails`** service (`AddProblemDetails()`) integrates with `UseExceptionHandler` and `UseStatusCodePages` to produce RFC 9457 responses by default.

**Senior-level gotchas:**
- `Use` vs `Run` vs `Map`: `Run` is **terminal** (never calls next). `Map`/`MapWhen` **branch** the pipeline — the branch has its own independent pipeline.
- Anything you write to `Response.Body` after `await next()` only works if the next middleware hasn't already started the response. Once headers are sent, you can't mutate them.
- Middleware is constructed **once** (singleton-ish) via `UseMiddleware<T>`. Injecting a scoped service through the constructor captures it for the app's lifetime — inject scoped dependencies into `InvokeAsync` instead.

---

## Q3. How can you stop other middlewares from executing?

You "short-circuit" the pipeline by **not calling `next`**. Any subsequent middleware is skipped; the response travels back up through the already-executed components.

```csharp
app.Use(async (ctx, next) =>
{
    if (!ctx.Request.Headers.ContainsKey("X-Api-Key"))
    {
        ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
        await ctx.Response.WriteAsync("Missing API key");
        return;                 // <-- short-circuit
    }
    await next();
});
```

Other ways to short-circuit:
- **`app.Run(...)`** — terminal by construction. Use at the end of a branch.
- **Returning a `Results.*`** from a minimal API endpoint — the endpoint itself is terminal.
- **Built-in middleware that short-circuits on policy failure** — `UseAuthorization` writes 401/403 and stops; `UseRateLimiter` writes 429; `UseOutputCache` serves a cached response and skips the endpoint; `UseStaticFiles` serves the file and stops.
- **`ctx.Abort()`** — kills the connection (no graceful response). Reserve for abuse/DoS.

**Modern note (.NET 10):** Minimal APIs expose **`ShortCircuit()`** on endpoint builders to skip remaining middleware between `UseRouting` and the endpoint — useful for `/health` or `/favicon.ico` that should bypass auth and rate limiting:

```csharp
app.MapGet("/health", () => Results.Ok()).ShortCircuit();
```

**Senior-level gotchas:**
- Short-circuiting **after** `await next()` is too late — downstream already ran. The check must happen before the await.
- Don't `throw` to short-circuit. Exceptions are 10–100× more expensive than a status-code return and pollute logs. Return a result instead.
- If you short-circuit before `UseAuthentication`, you lose `ctx.User`. Place your custom gate **after** auth unless it genuinely needs no identity.

---

## Q4. What is the difference between MVC and Razor Pages?

Both are built on the same primitives — routing, model binding, filters, Razor view engine, tag helpers, DI — so the difference is **organizational**, not technical:

| Aspect | MVC | Razor Pages |
|---|---|---|
| Organizing unit | Controller (class) with many actions | One `.cshtml` page + `PageModel` per URL |
| Routing | Convention (`{controller}/{action}/{id?}`) or attribute | File-system based (`/Pages/Orders/Edit.cshtml` → `/Orders/Edit`) |
| Request handler | Action method returning `IActionResult` | `OnGet` / `OnPost` / `OnGetAsync` handlers |
| State/model | Passed as action parameters + `ViewModel` | `[BindProperty]` fields on the `PageModel` |
| Best fit | APIs, complex apps with many actions per resource | Form-/page-centric server-rendered UI (CRUD, admin) |

```csharp
// MVC
public class OrdersController : Controller
{
    [HttpGet("orders/{id:int}")]
    public async Task<IActionResult> Details(int id, IOrderService svc, CancellationToken ct)
        => View(await svc.GetAsync(id, ct));
}

// Razor Pages: Pages/Orders/Details.cshtml.cs
public class DetailsModel(IOrderService svc) : PageModel
{
    public Order Order { get; private set; } = default!;

    public async Task<IActionResult> OnGetAsync(int id, CancellationToken ct)
    {
        Order = await svc.GetAsync(id, ct);
        return Page();
    }
}
```

Key distinctions a senior calls out:
- Razor Pages was introduced in 2.0 specifically to reduce the ceremony of a controller+view+VM trio for simple page flows. It pushes you toward **one URL = one file pair**, which improves cohesion.
- MVC is still the right choice for **Web APIs** (controllers inheriting `ControllerBase`) because REST resources have multiple verbs per URI — a natural fit for action methods.
- They **coexist** in the same app; `AddControllersWithViews().AddRazorPages()` is common.

**Modern note (.NET 10):** Microsoft's current guidance for new UI work is **Blazor United** (server + WASM + SSR in one model), not MVC or Razor Pages. Razor Pages and MVC are fully supported and not deprecated, but greenfield interactive UIs should default to Blazor. For Web APIs, **minimal APIs** have closed most of the feature gap with controllers (filters, endpoint groups, validation) and are typically the modern default.

**Senior-level gotchas:**
- Razor Pages uses **anti-forgery tokens by default** on POST handlers; MVC does not unless you add `[ValidateAntiForgeryToken]` or the global filter. Mixing stacks without aligning this trips security reviews.
- `PageModel` is instantiated per request — a Razor Page is effectively a scoped object. Injecting singletons holding per-request state is the same footgun as in controllers.
- `[BindProperty]` defaults to `SupportsGet = false`; forgetting this on a GET-bound field is a classic "why is my value null" bug.

---

## Q5. Name 3 ways to create middleware

### 1. Inline lambda via `Use` / `Run` / `Map`

Cheapest, best for small cross-cutting concerns and branch points.

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Correlation-Id"] = Guid.NewGuid().ToString("N");
    await next();
});
```

### 2. Convention-based middleware class

A plain class with a constructor taking `RequestDelegate next` and an `InvokeAsync(HttpContext, ...)` method. The runtime discovers it by convention (no interface). Instantiated **once per app** — scoped dependencies must be passed into `InvokeAsync`, not the constructor.

```csharp
public sealed class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext ctx, IMetrics metrics)   // scoped svc here
    {
        var sw = ValueStopwatch.StartNew();
        try { await next(ctx); }
        finally { metrics.RecordRequest(ctx.Request.Path, sw.GetElapsedTime()); }
    }
}

app.UseMiddleware<RequestTimingMiddleware>();
```

### 3. Factory-based middleware (`IMiddleware`)

Implements `IMiddleware`. Instantiated **per request** from DI, so it can take scoped/transient dependencies via the constructor. Must be registered in the DI container.

```csharp
public sealed class TenantMiddleware(ITenantResolver resolver) : IMiddleware
{
    public async Task InvokeAsync(HttpContext ctx, RequestDelegate next)
    {
        ctx.Items["Tenant"] = await resolver.ResolveAsync(ctx);
        await next(ctx);
    }
}

builder.Services.AddScoped<TenantMiddleware>();
app.UseMiddleware<TenantMiddleware>();
```

**Bonus — endpoint filters (minimal APIs):** `AddEndpointFilter` is not middleware per se, but it occupies the same niche for per-endpoint cross-cutting logic (validation, logging) and is strongly typed around the endpoint's args/result.

**Senior-level gotchas:**
- Convention-based middleware is a **captive** of the root provider. Injecting `DbContext`, `HttpClient`, or anything scoped into its constructor silently pins it to the app lifetime — a classic memory leak / stale-state bug. Use `IMiddleware` or method injection.
- `Run` vs `Use`: forgetting to `await next()` in a `Use` delegate silently short-circuits the pipeline — often diagnosed as "my controller isn't hit."
- Register middleware **before** the terminal endpoint mapping or it won't run. `app.MapControllers()` is terminal for matched routes.

---
