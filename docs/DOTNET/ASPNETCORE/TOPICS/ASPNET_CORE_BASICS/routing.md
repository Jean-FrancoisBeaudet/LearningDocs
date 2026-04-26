# Routing

_Targets .NET 10 / C# 14. See also: [Minimal APIs](./minimal-apis.md), [Controllers](./controllers.md), [Middlewares](./middlewares.md), [Filters and Attributes](./filters-and-attributes.md)._

Routing is ASP.NET Core's URL ↔ endpoint matcher. The same engine — `Microsoft.AspNetCore.Routing` — powers Minimal APIs, MVC controllers, gRPC, SignalR, Razor Pages, and Blazor Server. Understand it once, you understand it everywhere.

The pipeline is two middlewares with everything else in between:

```csharp
app.UseRouting();          // resolve the endpoint, attach to HttpContext
app.UseAuthentication();   // can read endpoint metadata (e.g. [Authorize])
app.UseAuthorization();
app.UseEndpoints(e => { /* terminal middleware that executes the endpoint */ });
```

In modern templates, top-level `app.MapGet(...)` / `app.MapControllers()` implicitly insert `UseRouting()` and `UseEndpoints()`. You almost never call them explicitly anymore — but the model is unchanged.

## Route templates

```csharp
app.MapGet("/orders/{id:int:min(1)}",          GetOrder);
app.MapGet("/orders/{id:guid}",                GetOrderByGuid);
app.MapGet("/users/{name:alpha:length(2,50)}", GetUser);
app.MapGet("/files/{*path}",                   ServeFile);   // catch-all
app.MapGet("/blog/{year:int}/{slug?}",         GetPost);     // optional segment
app.MapGet("/feed/{format=rss}",               GetFeed);     // default value
```

Built-in constraints: `int`, `long`, `guid`, `bool`, `datetime`, `decimal`, `alpha`, `length(n)`, `min(n)`, `max(n)`, `range(a,b)`, `regex(...)`. Constraints are **route disambiguators, not validation** — a constraint mismatch produces a 404, not a 400.

A custom constraint:

```csharp
public sealed class TenantSlugConstraint : IRouteConstraint
{
    public bool Match(HttpContext? c, IRouter? r, string key, RouteValueDictionary values, RouteDirection dir)
        => values[key] is string s && s.Length is >= 3 and <= 32 && s.All(char.IsLetterOrDigit);
}

builder.Services.Configure<RouteOptions>(o => o.ConstraintMap["tenant"] = typeof(TenantSlugConstraint));
// usage:  app.MapGet("/{tenant:tenant}/orders", ...)
```

## Route groups

`MapGroup` shares prefix, filters, auth, CORS, rate-limiting, and OpenAPI metadata. Groups nest:

```csharp
var v1 = app.MapGroup("/v1").WithTags("v1").RequireAuthorization();

var orders = v1.MapGroup("/orders").AddEndpointFilter<AuditFilter>();
orders.MapGet("/",         ListOrders);
orders.MapGet("/{id:int}", GetOrder);
orders.MapPost("/",        CreateOrder).DisableAntiforgery();
```

Per-group filters run *outermost first* (parent group → child group → endpoint). Auth on a parent applies to every descendant unless a child opts out with `.AllowAnonymous()`.

## Ordering and tie-breaks

When two templates can match the same URL, the router scores by precedence (literal segments > parameters with constraints > parameters > catch-all). Equal precedence yields an `AmbiguousMatchException` *at request time* — there is no compile-time check.

```csharp
app.MapGet("/items/featured", FeaturedItem);   // wins for /items/featured
app.MapGet("/items/{id}",     GetItem);        // wins for /items/42
```

Override with `.WithOrder(int)` (lower runs first). Use sparingly — explicit literal segments are clearer than ordering hacks.

## Link generation

Don't string-concatenate URLs. Generate them from named routes so a route refactor doesn't silently break callers:

```csharp
app.MapGet("/orders/{id:int}", GetOrder).WithName("GetOrder");

app.MapPost("/orders", async (Order o, IOrderRepo repo, LinkGenerator links, HttpContext ctx, CancellationToken ct) =>
{
    var saved = await repo.SaveAsync(o, ct);
    var url   = links.GetUriByName(ctx, "GetOrder", new { id = saved.Id });
    return TypedResults.Created(url, saved);
});

// Shorter:
return TypedResults.CreatedAtRoute(saved, "GetOrder", new { id = saved.Id });
```

In controllers, `Url.RouteUrl("GetOrder", new { id })` does the same thing.

## Endpoint metadata

Routing decorates each endpoint with metadata. Downstream middleware reads it without coupling to the endpoint type:

```csharp
app.MapGet("/admin/reports", GenerateReports)
   .RequireAuthorization("AdminPolicy")
   .WithName("AdminReports")
   .WithSummary("Generate the monthly admin reports")
   .WithTags("admin")
   .Produces<Report>(200)
   .ProducesProblem(403)
   .WithMetadata(new AuditAttribute("reports"));
```

Auth, CORS (`RequireCors`), rate limiting (`RequireRateLimiting`), output cache (`CacheOutput`), and OpenAPI all participate via metadata. Your own middleware reads it the same way:

```csharp
app.Use(async (ctx, next) =>
{
    var marker = ctx.GetEndpoint()?.Metadata.GetMetadata<AuditAttribute>();
    if (marker is not null) /* ... */;
    await next();
});
```

## Senior-level gotchas

- **Middleware ordering:** `UseRouting` must run before any middleware that reads endpoint metadata (`UseAuthorization`, `UseCors`, `UseRateLimiter`, `UseOutputCache`). `UseAuthorization` must run after `UseAuthentication` and after `UseRouting`. Wrong order = silent auth bypass on some paths.
- Route constraints are **matchers, not validators**. `{id:int}` returns 404 (no match) for `abc`, not 400. Validate inside the handler — clients shouldn't have to learn the difference.
- Templates match **case-insensitively**, but `LinkGenerator` produces URLs in the casing you registered. Mixing `/Orders` registration with `/orders` link output works inbound but produces inconsistent outbound URLs. Pick a casing convention and stick to it.
- `MapControllers()` registers a *single* dynamic endpoint that delegates into the MVC route table. You can mix it with top-level `Map*` calls, but routes from both compete in the same precedence table — a top-level `app.MapGet("/orders", ...)` plus a `[HttpGet("/orders")]` controller action throws `AmbiguousMatchException` at request time.
- Catch-all `{*path}` URL-decodes captured segments; `{**path}` (double-asterisk) preserves percent-encoded slashes. Needed when the captured value is itself a path containing `%2F`.
- `IUrlHelper` and `LinkGenerator` are not the same: the former needs the current `HttpContext`/`ActionContext` (controllers, views), the latter is a singleton you can inject anywhere — including background services where there is no request. Use `LinkGenerator` outside the request pipeline.
- The matcher is built **once at host start**. Dynamically-registered endpoints require either rebuilding an `EndpointDataSource` or using `MapDynamicControllerRoute`. Dynamic routing is rarely worth the complexity — prefer parameter routes.
- Route values are stringly-typed. `RouteValueDictionary` round-trips through `ToString()` and the constraint parser, which loses precision on `decimal` and timezone info on `DateTime`. Pass complex values as headers or body, not route segments.
- `MapGroup("/foo").MapGroup("/bar")` → final prefix `/foo/bar`. Filters and metadata accumulate; **but** `RequireAuthorization` does not stack — the innermost wins for that policy slot. If you mean "all of these", express it as a single composite policy.
