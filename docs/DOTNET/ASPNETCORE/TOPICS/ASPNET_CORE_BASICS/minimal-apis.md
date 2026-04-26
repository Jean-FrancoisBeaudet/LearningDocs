# Minimal APIs

_Targets .NET 10 / C# 14. See also: [Controllers](./controllers.md), [FastEndpoints](./fast-endpoints.md), [Routing](./routing.md), [Filters and Attributes](./filters-and-attributes.md), [Problem Details](./problem-details.md), [Error Handling](./error-handling.md), [OpenAPI](../API_DOCUMENTATION/open-api.md)._

Minimal APIs are the modern default for new ASP.NET Core services: `Map*` directly off `WebApplication`, no controller class, no convention magic, opt-in everything. Introduced in .NET 6, reached MVC parity in .NET 8 (filters, validation, OpenAPI), and in .NET 10 ship with first-class source-generated validation and full AOT support.

You give up the MVC convention layer (model binding subtleties, action filters, view rendering) for explicit, testable, fast endpoints with full compile-time return-type info.

## A complete service

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();
builder.Services.AddScoped<IOrderRepo, OrderRepo>();
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();
app.UseAuthentication();
app.UseAuthorization();
app.MapOpenApi();

var orders = app.MapGroup("/v1/orders").WithTags("orders").RequireAuthorization();

orders.MapGet("/", static async (IOrderRepo repo, int page = 1, int size = 50, CancellationToken ct = default) =>
    TypedResults.Ok(await repo.ListAsync(page, size, ct)));

orders.MapGet("/{id:int:min(1)}", static async Task<Results<Ok<Order>, NotFound>>
    (int id, IOrderRepo repo, CancellationToken ct) =>
        await repo.FindAsync(id, ct) is { } o ? TypedResults.Ok(o) : TypedResults.NotFound())
    .WithName("GetOrder");

orders.MapPost("/", static async Task<Results<Created<Order>, ValidationProblem>>
    (CreateOrderRequest req, IOrderRepo repo, CancellationToken ct) =>
{
    var saved = await repo.CreateAsync(req, ct);
    return TypedResults.Created($"/v1/orders/{saved.Id}", saved);
});

app.Run();

public partial class Program;   // for WebApplicationFactory<Program> in tests
```

`static` handlers matter — a non-static lambda capturing `repo` allocates a closure per request. Inject everything as a parameter and keep handlers static.

## Parameter binding

The framework infers binding source from type and attributes:

| Source | When inferred | Override |
|---|---|---|
| Route values | type matches a route param name | `[FromRoute]` |
| Query string | simple types not matching a route param | `[FromQuery]` |
| Headers | never inferred — always explicit | `[FromHeader(Name="X-...")]` |
| Body (JSON) | complex types (one per request) | `[FromBody]` |
| Form | `IFormFile`, `IFormCollection`, types with `[FromForm]` | `[FromForm]` |
| Services | types registered in DI | `[FromServices]` (rarely needed) |
| Well-known | `HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken`, `Stream` | — |
| Parameter object | bundle of mixed sources | `[AsParameters]` |

Bundle related parameters with `[AsParameters]`:

```csharp
record OrderQuery(
    [FromQuery] int Page = 1,
    [FromQuery] int Size = 50,
    [FromQuery] string? Status = null,
    [FromHeader(Name="X-Tenant")] string Tenant = "default");

orders.MapGet("/", ([AsParameters] OrderQuery q, IOrderRepo repo, CancellationToken ct) =>
    repo.ListAsync(q, ct));
```

Custom types can bind themselves via the static `BindAsync` protocol:

```csharp
public readonly record struct OrderId(int Value)
{
    public static ValueTask<OrderId?> BindAsync(HttpContext ctx, ParameterInfo p)
    {
        var raw = ctx.Request.RouteValues[p.Name!]?.ToString();
        return ValueTask.FromResult<OrderId?>(int.TryParse(raw, out var v) ? new OrderId(v) : null);
    }
}
```

Or `TryParse` for query/route scalars:

```csharp
public readonly record struct Slug(string Value)
{
    public static bool TryParse(string? s, out Slug result)
        => (result = new Slug(s ?? "")).Value.Length > 0;
}
```

## Returning results

```csharp
// Older: IResult-erasing — OpenAPI sees `object`, must annotate with .Produces<T>()
app.MapGet("/a", () => Results.Ok(new Foo()));

// Modern: TypedResults — OpenAPI sees the exact union, no annotations needed
app.MapGet("/b", () => TypedResults.Ok(new Foo()));

// Discriminated union return type — OpenAPI extracts every variant
app.MapGet("/c", Task<Results<Ok<Foo>, NotFound, ProblemHttpResult>>
    (int id) => /* ... */);
```

`Results<T1, T2, ...>` is a typed union (up to six type parameters). The OpenAPI generator extracts each variant and lists the corresponding response — strictly better than `[ProducesResponseType]` annotations on `IResult` returns.

## Endpoint filters

The minimal-API equivalent of MVC action filters. They wrap the handler invocation:

```csharp
public sealed class TimingFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var sw = Stopwatch.StartNew();
        try { return await next(ctx); }
        finally { Log.Information("{Endpoint} took {Ms} ms",
            ctx.HttpContext.GetEndpoint()?.DisplayName, sw.ElapsedMilliseconds); }
    }
}

orders.AddEndpointFilter<TimingFilter>()
      .AddEndpointFilter(static async (ctx, next) =>
      {
          if (ctx.GetArgument<CreateOrderRequest>(0) is { Total: < 0 })
              return TypedResults.ValidationProblem(new Dictionary<string, string[]>
                  { ["total"] = ["must be non-negative"] });
          return await next(ctx);
      });
```

Filter ordering: outer (group) → inner (endpoint), parent groups before child. Endpoint filters run **after** parameter binding — they can read but not mutate body parameters. For pre-binding behavior (raw body, custom auth), write middleware instead.

## Validation (.NET 10)

`Microsoft.AspNetCore.Http.Validation` ships source-generated validators that read DataAnnotations (`[Required]`, `[Range]`, `[StringLength]`) and emit a `ValidationProblem` automatically:

```csharp
builder.Services.AddValidation();   // .NET 10+

record CreateOrderRequest(
    [Required, EmailAddress] string CustomerEmail,
    [Range(0.01, 1_000_000)] decimal Total);

orders.MapPost("/", (CreateOrderRequest req) => TypedResults.Created($"/orders/1", req))
      .DisableValidation();   // opt out per endpoint
```

For richer rules (cross-field, async DB checks), use [FluentValidation](../VALIDATION/fluent-validation.md) inside an endpoint filter.

## Auth, CORS, rate limiting

```csharp
orders.MapDelete("/{id:int}", DeleteOrder)
      .RequireAuthorization("AdminPolicy")
      .RequireCors("AdminOrigins")
      .RequireRateLimiting("write-policy");
```

Each metadata-attaching extension stacks; group-level applies to every endpoint inside.

## Testing

`Program.cs` uses top-level statements. To use `WebApplicationFactory<TEntryPoint>`, declare `public partial class Program;` at the bottom (shown above):

```csharp
public class OrdersTests(WebApplicationFactory<Program> factory) : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact] public async Task Get_returns_404_for_missing()
    {
        var resp = await factory.CreateClient().GetAsync("/v1/orders/99999");
        Assert.Equal(HttpStatusCode.NotFound, resp.StatusCode);
    }
}
```

## Minimal API vs Controllers vs FastEndpoints

| | Minimal API | Controllers (MVC) | FastEndpoints |
|---|---|---|---|
| Boilerplate | Lowest | Medium | Low |
| Convention layer | None | Heavy (binders, action filters, conventions) | Vertical-slice per endpoint |
| OpenAPI | Native typed results | Attribute-driven | Attribute + typed |
| AOT-friendly | Yes (with source-gen JSON) | Partial | Yes |
| Best for | Small/medium services, microservices | Large MVC apps, server-rendered views | Vertical-slice REPR-style APIs |

## Senior-level gotchas

- `Results.Ok(...)` returns `IResult`; `TypedResults.Ok(...)` returns `Ok<T>`. Use `TypedResults` so OpenAPI knows the response shape without `[ProducesResponseType]` clutter — and so the compiler verifies your `Results<...>` union covers every return path.
- Endpoint filters run **after** parameter binding. They can short-circuit but cannot rewrite the bound parameter (it's already inside `EndpointFilterInvocationContext.Arguments`). For pre-binding behavior, write middleware.
- `IFormFile` parameters require antiforgery in .NET 7+. Either call `app.UseAntiforgery()` and emit a token, or `.DisableAntiforgery()` per endpoint for raw HTTP clients (mobile apps). Never globally disable it.
- `[AsParameters]` requires a public parameterless constructor *or* a record with a primary constructor. Init-only properties bind from the source attribute on each property.
- AOT (`<PublishAot>true</PublishAot>`) requires source-generated JSON (`JsonSerializerContext`) and forbids reflection-based binders. Always run `dotnet publish -c Release` in CI — trim warnings only surface at publish time, not in `dotnet run`.
- Non-static handlers capturing local variables allocate a closure *per request*. Force `static` on every handler and inject all dependencies as parameters.
- `app.MapGroup("/v1").WithOpenApi()` does *not* propagate `WithOpenApi()` to children directly — it attaches OpenAPI metadata to the group, which children inherit. Verify with `endpoint.Metadata.GetMetadata<OpenApiOperation>()` if a child seems to lack metadata.
- `Results<T1, T2, ...>` supports up to six type parameters. If you need more, your endpoint is doing too much — split it.
- Minimal-API request delegates are built by `RequestDelegateFactory.Create`, which inlines parameter parsing into a generated method per endpoint at startup. Cold start is slightly slower than controllers (which share a single invoker), but per-request overhead is lower. Don't profile cold start in dev and panic.
- Returning a `Task<IResult>` whose runtime type is `Ok<Foo>` works but defeats OpenAPI introspection — the framework only sees the static return type. Always declare the typed union explicitly.
