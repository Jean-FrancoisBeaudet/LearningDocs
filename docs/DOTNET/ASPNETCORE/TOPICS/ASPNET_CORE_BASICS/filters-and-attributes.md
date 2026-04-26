# Filters & Attributes

_Targets .NET 10 / C# 14. See also: [middlewares](./middlewares.md), [controllers](./controllers.md), [minimal-apis](./minimal-apis.md), [error-handling](./error-handling.md), [authentication-and-authorization](./authentication-and-authorization.md), [jwt](../SECURITY/jwt.md)._

Filters are ASP.NET Core's MVC-aware cross-cutting hook. Where [middleware](./middlewares.md) sees a raw `HttpContext`, a filter sees the **action** about to run, the **model state** that bound to it, and the **`IActionResult`** it produced. That extra knowledge is exactly when filters earn their keep — for everything else, prefer middleware. Minimal APIs don't have MVC filters at all; they use **`IEndpointFilter`**, which is a different type with a similar shape and the same purpose.

## The MVC filter pipeline

Per request, MVC runs filters in this order:

```
Authorization Filter   → can the user even attempt this action?
Resource Filter        → wraps model binding + everything below; cache/short-circuit territory
   Model Binding
   Action Filter       → before / after the action method
      Action Method
   Action Filter       → (after)
   Exception Filter    → only catches exceptions from action + action filters
   Result Filter       → before / after the IActionResult is executed
   Result Execution
Resource Filter        → (after)
```

Two implications:

- An exception in **middleware** never reaches an exception filter. `UseExceptionHandler` is the only thing that catches it — see [error-handling](./error-handling.md).
- A `Resource` filter that short-circuits skips model binding entirely. That's how `[ResponseCache]` and the output-cache filter work.

## The five filter types

### `IAuthorizationFilter`

```csharp
public sealed class TenantAuthorizationFilter : IAsyncAuthorizationFilter
{
    public async Task OnAuthorizationAsync(AuthorizationFilterContext ctx)
    {
        var tenant = ctx.HttpContext.Request.Headers["X-Tenant"].ToString();
        if (string.IsNullOrEmpty(tenant))
            ctx.Result = new UnauthorizedResult();   // short-circuit
    }
}
```

For most apps, prefer the policy-based authorization system (`[Authorize(Policy=…)]` + `IAuthorizationHandler`) over a hand-rolled authorization filter — it's the supported extension point.

### `IResourceFilter`

```csharp
public sealed class CacheFilter(IMemoryCache cache) : IAsyncResourceFilter
{
    public async Task OnResourceExecutionAsync(ResourceExecutingContext ctx, ResourceExecutionDelegate next)
    {
        var key = ctx.HttpContext.Request.Path + ctx.HttpContext.Request.QueryString;
        if (cache.TryGetValue(key, out IActionResult? cached))
        {
            ctx.Result = cached;                     // short-circuit before model binding
            return;
        }
        var executed = await next();
        if (executed.Result is OkObjectResult ok)
            cache.Set(key, ok, TimeSpan.FromSeconds(30));
    }
}
```

Wraps everything below — model binding, the action, result execution. Only place where a cache hit can avoid binding cost.

### `IActionFilter`

```csharp
public sealed class ValidationLogFilter(ILogger<ValidationLogFilter> log) : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next)
    {
        if (!ctx.ModelState.IsValid)
            log.LogWarning("Invalid binding for {Action}", ctx.ActionDescriptor.DisplayName);

        var executed = await next();                 // run the action
        if (executed.Exception is not null)
            log.LogError(executed.Exception, "Action threw");
    }
}
```

Receives bound arguments in `ctx.ActionArguments`, runs before/after the action body. Common home for input mutation, logging, validation gating.

### `IExceptionFilter`

```csharp
public sealed class DomainExceptionFilter : IAsyncExceptionFilter
{
    public Task OnExceptionAsync(ExceptionContext ctx)
    {
        if (ctx.Exception is DomainException dx)
        {
            ctx.Result = new ObjectResult(new { error = dx.Code }) { StatusCode = 422 };
            ctx.ExceptionHandled = true;             // suppress propagation
        }
        return Task.CompletedTask;
    }
}
```

Catches throws from the action and downstream action/result filters — **not** model binding or middleware. For unhandled middleware exceptions, use `UseExceptionHandler`/`UseProblemDetails`.

### `IResultFilter`

```csharp
public sealed class HateoasFilter : IAsyncResultFilter
{
    public async Task OnResultExecutionAsync(ResultExecutingContext ctx, ResultExecutionDelegate next)
    {
        if (ctx.Result is ObjectResult { Value: IResource r })
            r.AddLink("self", ctx.HttpContext.Request.Path);
        await next();
    }
}
```

Last chance to mutate the response shape before it goes through the formatter. After `next()`, the body has been written — header changes are too late (`Response.HasStarted == true`).

## Sync vs async — pick one

Every filter has a sync (`OnActionExecuting/OnActionExecuted`) and an async (`OnActionExecutionAsync`) shape. **Implement only one.** If both are implemented, MVC runs the async one and silently ignores the sync ones.

## Registering filters

Three flavours, three lifetimes:

| Mechanism | Where applied | DI lifetime | Use when |
|---|---|---|---|
| `[ServiceFilter(typeof(MyFilter))]` | controller / action | resolved from DI (singleton, scoped, transient — whatever you registered) | filter has dependencies you want managed by DI |
| `[TypeFilter(typeof(MyFilter))]` | controller / action | activated once per request via `ObjectFactory` | filter takes constructor args you want to pass *and* DI-resolve mixed |
| `MvcOptions.Filters.Add<T>()` | global, every action | per-request transient | applies broadly, no opt-in/opt-out per action |
| `[MyFilter]` (filter is an attribute) | declarative | constructed by activator — **no DI** | the filter has no dependencies |

```csharp
builder.Services.AddScoped<DomainExceptionFilter>();
builder.Services.AddControllers(o =>
{
    o.Filters.Add<RequestLoggingFilter>();           // global
});

[ServiceFilter(typeof(DomainExceptionFilter))]      // per-controller
public sealed class OrdersController : ControllerBase { /* ... */ }
```

A filter that is itself an `Attribute` (e.g. `MyFilterAttribute : Attribute, IActionFilter`) is convenient at call sites but **cannot use constructor DI** — attribute arguments must be `const`. If it needs services, switch to `[ServiceFilter]` or `[TypeFilter]` and register the filter type in DI.

### Order

`Order` property + scope determines execution. Lower `Order` runs first (outer); ties resolve by scope: **global → controller → action**. After-callbacks run in reverse. Don't rely on `Order` for correctness; if you do, comment why.

## `IEndpointFilter` for minimal APIs

Minimal APIs have **no** `IActionFilter`. Use endpoint filters:

```csharp
app.MapPost("/orders", (CreateOrderRequest req) => Results.Ok())
   .AddEndpointFilter(async (ctx, next) =>
   {
       var sw = Stopwatch.StartNew();
       var result = await next(ctx);
       ctx.HttpContext.Response.Headers["X-Elapsed"] = sw.ElapsedMilliseconds.ToString();
       return result;
   });
```

The `EndpointFilterInvocationContext` exposes bound arguments via `ctx.Arguments` (object[]). Endpoint filters compose left-to-right (first registered runs first on the way in, last on the way out).

For class-based filters implement `IEndpointFilter` and register with `.AddEndpointFilter<MyFilter>()` — they get DI through their constructor.

```csharp
public sealed class IdempotencyFilter(IIdempotencyStore store) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var key = ctx.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (await store.TryGetAsync(key, ctx.HttpContext.RequestAborted) is { } cached)
            return cached;

        var result = await next(ctx);
        await store.SaveAsync(key, result!, TimeSpan.FromHours(24), ctx.HttpContext.RequestAborted);
        return result;
    }
}
```

## Common attributes inventory

| Attribute | Purpose |
|---|---|
| `[ApiController]` | Activates automatic 400 on `ModelState` errors, infers binding sources, and enforces attribute routing. |
| `[Authorize]` / `[AllowAnonymous]` | Authorization gating. With `Policy = "..."` selects a registered policy. |
| `[Route("…")]`, `[HttpGet/Post/Put/Patch/Delete("…")]` | Attribute routing. Mandatory under `[ApiController]`. |
| `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromHeader]`, `[FromForm]`, `[FromServices]` | Explicit binding source override. |
| `[Produces("application/json")]`, `[Consumes("application/xml")]` | Content negotiation hints to model binder + OpenAPI. |
| `[ProducesResponseType<T>(200)]`, `[ProducesProblem(404)]` | OpenAPI documentation of response shapes. Without these the generator can't infer the shape from `IActionResult`. |
| `[ValidateAntiForgeryToken]` | Form-post CSRF protection. |
| `[ResponseCache]` | Sets `Cache-Control` headers (and integrates with output caching). |
| `[NonAction]` | Stops a public method on a controller from becoming an endpoint. |
| `[ServiceFilter]`, `[TypeFilter]` | Apply a filter from DI. |

## When to write a filter vs middleware

Use **middleware** when the concern doesn't depend on MVC: HTTPS redirection, compression, CORS, exception → ProblemDetails mapping, request logging at the HTTP boundary, JWT validation.

Use a **filter** when you need:

- the matched action descriptor (controller/action name, attributes)
- bound arguments before they reach the action
- the produced `IActionResult` before/after formatting
- short-circuiting that returns a typed `IActionResult`, not a manually-written response

Use an **endpoint filter** when you'd write a filter, but the endpoint is a Minimal API.

## Senior-level gotchas

- **`[ApiController]` changes binding rules**: complex types default to `[FromBody]`, primitives to `[FromQuery]`/`[FromRoute]`. It also auto-emits a 400 on `ModelState` errors — disable with `ApiBehaviorOptions.SuppressModelStateInvalidFilter = true` if you fully own validation flow.
- **Attribute filters can't take services**: an attribute's arguments must be compile-time constants. If you need DI, use `[ServiceFilter(typeof(MyFilter))]` and register the filter type. People hit this when they "make a filter into an attribute" and the constructor parameter resolution silently fails.
- **Exception filters miss middleware throws**: anything that throws in `UseRouting`, `UseAuthentication`, `UseAuthorization`, model binding (when `[ApiController]` is off and a binder throws), or the formatter never reaches an exception filter. `UseExceptionHandler` is the catch-all.
- **`ctx.Result = ...` short-circuits**: setting `Result` on an authorization or resource filter context skips downstream pipeline. On an `ActionExecutedContext`, setting `Result` *replaces* what the action returned — fine, but it also suppresses any thrown exception unless you also re-throw or set `ctx.Exception = null` deliberately.
- **`OnResultExecuting` is not the place to add headers** if anything below has already started the response — check `ctx.HttpContext.Response.HasStarted` before mutating headers, or move to `OnActionExecuted` / `OnResultExecuting` *before* `next()`.
- **MVC `IActionFilter` ≠ `IEndpointFilter`**: they live in different namespaces, target different pipelines, and are not interchangeable. A filter you wrote for controllers will not apply to a `MapPost(...)` endpoint and vice versa. Decide upfront which authoring model the project uses.
- **Global filters run on every action** — including action methods you forgot were endpoints (`[HttpGet] public string Health() => "ok";`). If your filter is expensive (DB call, HTTP probe), gate it on `ctx.ActionDescriptor` metadata.
- **Filter ordering across scopes**: a global filter with `Order = 100` still runs *before* a controller filter with `Order = 0`, because scope wins ties only when `Order` is equal. Read order as `(Order, Scope)` — sort by `Order` first, then by scope (global before controller before action).
- **`[NonAction]` on a method that overrides a controller base** doesn't prevent the base method from being a route — only the override is excluded. Place the attribute on the method MVC actually inspects.
