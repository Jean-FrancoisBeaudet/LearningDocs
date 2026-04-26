# FastEndpoints

_Targets .NET 10 / C# 14 with FastEndpoints v6. See also: [minimal-apis](./minimal-apis.md), [controllers](./controllers.md), [fluent-validation](../VALIDATION/fluent-validation.md), [swagger](../API_DOCUMENTATION/swagger.md), [open-api](../API_DOCUMENTATION/open-api.md)._

FastEndpoints is a third-party library that pushes ASP.NET Core into the **REPR** (Request → Endpoint → Response) pattern: one endpoint = one class = one file. It sits between Minimal APIs (concise but flat) and Controllers (organized but ceremonial), and is the natural fit for vertical-slice / feature-folder architectures where every API operation owns its handler, validator, mapper, and DTOs together. It generates strongly-typed, fast (source-generated registration on .NET 8+) endpoints and integrates cleanly with FluentValidation and Swashbuckle. Pick it on services with **many endpoints** where one-class-per-endpoint reduces cognitive load; skip it on small services where Minimal APIs are already the smallest unit you need.

## Project shape

```xml
<PackageReference Include="FastEndpoints" Version="6.*" />
<PackageReference Include="FastEndpoints.Swagger" Version="6.*" />
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddFastEndpoints()                  // discovers endpoints + validators in the assembly
    .SwaggerDocument();                  // OpenAPI / Swagger UI integration

var app = builder.Build();
app.UseFastEndpoints()
   .UseSwaggerGen();

app.Run();
```

`AddFastEndpoints()` scans the entry assembly by reflection (or via source generator on .NET 8+) for every `Endpoint<TRequest>` / `Endpoint<TRequest, TResponse>` / `EndpointWithoutRequest<TResponse>` derived class. Validators (`Validator<TRequest>`) are picked up the same way.

## The endpoint class

```csharp
public sealed record CreateOrderRequest(string Sku, int Quantity);
public sealed record CreateOrderResponse(Guid Id, DateTimeOffset CreatedAt);

public sealed class CreateOrder(IOrderRepo repo) : Endpoint<CreateOrderRequest, CreateOrderResponse>
{
    public override void Configure()
    {
        Post("/orders");
        Roles("buyer");                  // policy/role-based auth — see SECURITY/jwt.md
        Description(b => b.Produces<CreateOrderResponse>(201).ProducesProblem(400));
        Summary(s =>
        {
            s.Summary = "Place a new order";
            s.ExampleRequest = new("SKU-42", 1);
        });
    }

    public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var order = await repo.CreateAsync(req.Sku, req.Quantity, ct);
        await SendCreatedAtAsync<GetOrder>(new { id = order.Id }, new(order.Id, order.CreatedAt), cancellation: ct);
    }
}
```

Three building blocks:

- **`Configure()`** — declarative routing, auth, response types. Maps to the metadata that OpenAPI/middleware reads.
- **`HandleAsync`** — the unit of work; receives the bound + validated request.
- **`SendXxxAsync`** helpers — `SendOkAsync`, `SendCreatedAtAsync`, `SendNoContentAsync`, `SendErrorsAsync` — write the response and short-circuit. Don't `return` a body from `HandleAsync`; you `Send`.

## Validators (FluentValidation under the hood)

```csharp
public sealed class CreateOrderValidator : Validator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.Sku).NotEmpty().Matches("^SKU-\\d+$");
        RuleFor(x => x.Quantity).GreaterThan(0).LessThanOrEqualTo(1000);
    }
}
```

Auto-discovered, auto-wired. On invalid input the framework returns `400 Bad Request` with a `ProblemDetails`-shaped body (errors keyed by property) — no manual `if (!ModelState.IsValid)` ceremony. Override `ValidationFailed` if you need a custom shape, or call `ThrowError` / `AddError` from `HandleAsync` to emit business validation errors with the same wire format.

## Pre/post processors

Cross-cutting concerns without filters or middleware:

```csharp
public sealed class AuditPreProcessor : IPreProcessor<CreateOrderRequest>
{
    public Task PreProcessAsync(IPreProcessorContext<CreateOrderRequest> ctx, CancellationToken ct)
    {
        ctx.HttpContext.Items["audit-start"] = DateTimeOffset.UtcNow;
        return Task.CompletedTask;
    }
}

// inside Configure()
PreProcessor<AuditPreProcessor>();
```

Register globally with `app.UseFastEndpoints(c => c.Endpoints.Configurator = ep => ep.PreProcessor<AuditPreProcessor>(Order.Before))`. Post-processors mirror the API and run after `HandleAsync`.

## Versioning

```csharp
public override void Configure()
{
    Get("/orders/{id}");
    Version(2, deprecateAt: 3);          // v2 active, will deprecate in v3
}
```

Wire the global config:

```csharp
app.UseFastEndpoints(c =>
{
    c.Versioning.Prefix = "v";           // /v1/orders, /v2/orders
    c.Versioning.DefaultVersion = 1;
});
```

## Auth

```csharp
public override void Configure()
{
    Post("/admin/orders/{id}/refund");
    Policies("AdminOnly");
    Roles("admin", "ops");               // OR semantics — any role is enough
    Permissions("orders:refund");        // FastEndpoints' own permission claims
    AllowAnonymous();                    // override on a specific endpoint
}
```

Policies map to the regular `AddAuthorization(o => o.AddPolicy(...))` registrations — FastEndpoints just exposes shorthand. See [authentication-and-authorization](./authentication-and-authorization.md), [jwt](../SECURITY/jwt.md).

## When to pick FastEndpoints vs Minimal API vs Controller

| Need | FastEndpoints | Minimal API | Controller |
|---|---|---|---|
| One class per operation (vertical slice) | ✅ enforced | possible (one file per endpoint) | one class per *resource* |
| Validation auto-wired with rich errors | ✅ FluentValidation built-in | manual or `MinimalApis.Extensions` | DataAnnotations or FV via filters |
| Per-endpoint auth on a config DSL | ✅ `Policies/Roles/Permissions()` | `RequireAuthorization(...)` | `[Authorize]` |
| OpenAPI integration | ✅ `SwaggerDocument()` | ✅ `AddOpenApi()` | ✅ `AddOpenApi()` / Swashbuckle |
| Performance | comparable to minimal API; source-gen reduces startup | fastest by a hair | slowest of the three (filter pipeline) |
| Discoverability for newcomers | learning curve | trivial | familiar |
| Best fit | **medium-to-large** APIs, vertical slices, CQRS handlers | **small** services, prototypes | **legacy** projects, large MVC apps with shared filters |

## Testing

```csharp
public sealed class CreateOrderTests
{
    [Fact]
    public async Task Returns_201_with_created_order()
    {
        var ep = Factory.Create<CreateOrder>(new FakeOrderRepo());
        await ep.HandleAsync(new("SKU-1", 2), CancellationToken.None);

        ep.HttpContext.Response.StatusCode.Should().Be(201);
        ep.Response.Sku.Should().Be("SKU-1");      // not actually a property — illustrative
    }
}
```

`Factory.Create<TEndpoint>(deps)` instantiates the endpoint with a fake `HttpContext` and your dependencies, no `WebApplicationFactory` needed. For full end-to-end use a regular `WebApplicationFactory<Program>`.

## Senior-level gotchas

- **Reflection-scan startup cost**: by default the assembly scan runs at startup. On .NET 8+ enable the source generator (`<FastEndpointsRcl>true</FastEndpointsRcl>` plus the analyzer) — moves discovery to compile time, drops cold start by 100s of ms in large APIs.
- **Mixing with controllers**: legal but messy. Both pipelines coexist and route conflicts are silent winners-take-all (whichever endpoint registered last). Prefer one model per service.
- **`Send*Async` writes immediately** — once called, you cannot return early or change the status code. Don't gate `SendOkAsync` behind a `try/catch` that wants to convert exceptions to `400`s; let the validator do that.
- **Validator vs `HandleAsync` errors**: validation runs *before* `HandleAsync`. Business errors (out-of-stock, payment declined) raised inside `HandleAsync` should use `ThrowError("…")` or `AddError("…")` + `ThrowIfAnyErrors()` so the wire format matches validator output. Don't throw raw exceptions for expected business failures — they 500.
- **Endpoint instances are transient**: each request gets a new endpoint instance, so injecting `DbContext` etc. via primary constructor is safe. But fields on `Endpoint<>` (like `Response`, `HttpContext`) are populated by the framework and are not thread-safe — never share an instance across requests.
- **OpenAPI integration is `FastEndpoints.Swagger`, not `Microsoft.AspNetCore.OpenApi`**: in .NET 9/10 the first-party generator is `Microsoft.AspNetCore.OpenApi` ([open-api](../API_DOCUMENTATION/open-api.md)), but FastEndpoints reads endpoint metadata via its own pipeline. They can coexist, but if you migrate, expect to redo `Description(...)` blocks.
- **Global preprocessors run for all endpoints** — including `/swagger` and `/openapi/v1.json`. If your preprocessor logs or counts, exclude infrastructure paths or you'll skew metrics.
- **Throwing inside `Configure()` is silent**: it runs at app startup but exceptions there can be swallowed by the discovery pipeline. Validate config-time invariants explicitly with a test that calls `Factory.Create<>` for every endpoint.
