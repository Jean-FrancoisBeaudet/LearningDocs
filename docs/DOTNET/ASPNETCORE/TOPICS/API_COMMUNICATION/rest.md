# REST

_Targets .NET 10 / C# 14. See also: [gRPC](./grpc.md), [GraphQL HotChocolate](./graphql-hotchocolate.md), [SOAP](./soap.md), [HttpClient & HttpClientFactory](./httpclient-and-httpclientfactory.md). For the protocol-level details, see [REST_FUNDAMENTALS](../REST_FUNDAMENTALS/) (maturity model, constraints, methods, status codes, HATEOAS)._

REST is the default API style on ASP.NET Core: **HTTP verbs + URI-named resources + status codes + content negotiation**, with the server stateless between requests. ASP.NET Core does not "do REST" — it gives you the building blocks (routing, model binding, content negotiation, problem details, OpenAPI) and stays out of your way. Pick REST when the consumer set is heterogeneous (browsers, mobile, third-party), the operations are CRUD-shaped, and you want maximum tooling reach (caching proxies, CDNs, OpenAPI codegen). Avoid it when you need streaming, strict typed contracts, or single-roundtrip aggregation — those are gRPC and GraphQL territory.

## The two stacks

ASP.NET Core ships two endpoint authoring models, both produce REST:

```csharp
// Minimal API — the modern default for new services
var app = builder.Build();
app.MapGet("/orders/{id:int:min(1)}", async (int id, IOrderRepo repo, CancellationToken ct) =>
{
    var order = await repo.FindAsync(id, ct);
    return order is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(order);
})
.WithName("GetOrder")
.Produces<Order>(StatusCodes.Status200OK)
.ProducesProblem(StatusCodes.Status404NotFound);
```

```csharp
// Controller — preferred when you have many endpoints sharing filters/policies/conventions
[ApiController]
[Route("orders")]
public sealed class OrdersController(IOrderRepo repo) : ControllerBase
{
    [HttpGet("{id:int:min(1)}", Name = "GetOrder")]
    [ProducesResponseType<Order>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<Results<Ok<Order>, NotFound>> Get(int id, CancellationToken ct)
        => await repo.FindAsync(id, ct) is { } o
            ? TypedResults.Ok(o)
            : TypedResults.NotFound();
}
```

`TypedResults.*` (returning `Results<T1, T2, ...>`) gives you compile-time response types that the OpenAPI generator picks up — preferred over the older `IActionResult` shape.

## Verb / resource mapping

| Verb | Use | Idempotent | Safe | Body |
|---|---|---|---|---|
| `GET /orders` | list | ✓ | ✓ | no |
| `GET /orders/{id}` | fetch one | ✓ | ✓ | no |
| `POST /orders` | create (server-generated id) | ✗ | ✗ | yes |
| `PUT /orders/{id}` | replace whole resource | ✓ | ✗ | yes |
| `PATCH /orders/{id}` | partial update | usually ✗ | ✗ | yes (JSON Merge Patch / JSON Patch) |
| `DELETE /orders/{id}` | remove | ✓ | ✗ | no |

See [http-methods](../REST_FUNDAMENTALS/http-methods.md) and [http-status-codes](../REST_FUNDAMENTALS/http-status-codes.md) for the full table.

**`POST` for client-supplied id?** That's `PUT /orders/{id}` (idempotent create-or-replace). Reserve `POST` for "server picks the id."

## Content negotiation

```csharp
// Server: register output formatters (JSON is default; XML opt-in)
builder.Services.AddControllers()
    .AddXmlSerializerFormatters();   // adds application/xml support

[Produces("application/json", "application/xml")]
public class OrdersController : ControllerBase { /* ... */ }
```

The framework reads `Accept` and picks a formatter; on no match, returns `406 Not Acceptable` (only when `RespectBrowserAcceptHeader = true` or `ReturnHttpNotAcceptable = true` on `MvcOptions`). Defaults are lenient — set them strict for public APIs.

JSON serialization on .NET 10 is **`System.Text.Json` with source generators** by default. Wire it once and reap AOT-friendly, allocation-light parsing:

```csharp
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(Order[]))]
public partial class ApiJsonContext : JsonSerializerContext;

builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, ApiJsonContext.Default));
```

## Problem Details (RFC 9457)

Don't invent your own error envelope:

```csharp
builder.Services.AddProblemDetails();        // canonical RFC 9457 responses
app.UseExceptionHandler();                   // converts unhandled exceptions
app.UseStatusCodePages();                    // converts bare 4xx/5xx with no body
```

A 404 now serializes to `application/problem+json`:

```json
{ "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found", "status": 404, "traceId": "00-..." }
```

Surface domain errors with `TypedResults.Problem(...)` or `Results.ValidationProblem(errors)`. See [problem-details](../ASPNET_CORE_BASICS/problem-details.md).

## OpenAPI in .NET 10

`Microsoft.AspNetCore.OpenApi` is the **first-party** generator. No more Swashbuckle by default:

```csharp
builder.Services.AddOpenApi();                // produce /openapi/v1.json
app.MapOpenApi();                             // expose it
// Pair with Scalar, Swagger UI, or Redoc — separate package.
```

The generator reads endpoint metadata (`.WithName`, `.Produces<T>`, `.WithSummary`, `[ProducesResponseType]`) directly from the route table — another reason to keep that metadata complete. See [open-api](../API_DOCUMENTATION/open-api.md).

## Versioning

Three viable strategies:

| Style | Example | Notes |
|---|---|---|
| **URI** | `/v1/orders` | Most common, easiest to cache and proxy. Hard to evolve a single resource — you fork the whole tree. |
| **Header** | `api-version: 2.0` | Clean URIs but harder to test in a browser; CDN cache key needs `Vary`. |
| **Media type** | `Accept: application/vnd.acme.order.v2+json` | REST-purist; rarely used in practice — tooling friction outweighs the aesthetic. |

Use `Asp.Versioning.Http` (the official package — `Microsoft.AspNetCore.Mvc.Versioning` was renamed). See [api-versioning](../ASPNET_CORE_BASICS/api-versioning.md).

## Pagination, filtering, sorting, shaping

REST APIs at scale leak details through query strings, not bodies:

```
GET /orders?status=Open&customerId=42&sort=-createdAt&page=3&pageSize=50&fields=id,total,status
```

Cross-links: [pagination](../REST_FUNDAMENTALS/pagination.md), [filtering](../REST_FUNDAMENTALS/filtering.md), [sorting](../REST_FUNDAMENTALS/sorting.md), [data-shaping](../REST_FUNDAMENTALS/data-shaping.md). Use `ETag` + `If-None-Match` for cache validation; emit `Link` headers (RFC 8288) for next/prev page URIs — that's HATEOAS-lite, see [hateoas](../REST_FUNDAMENTALS/hateoas.md).

## Idempotency for `POST`

`POST` is not idempotent — but networks retry. Adopt the **`Idempotency-Key`** header (Stripe-style):

```csharp
app.MapPost("/payments", async (Payment p, [FromHeader(Name="Idempotency-Key")] Guid key,
                                IIdempotencyStore store, CancellationToken ct) =>
{
    if (await store.TryGetAsync<PaymentResult>(key, ct) is { } cached)
        return TypedResults.Ok(cached);

    var result = await ProcessAsync(p, ct);
    await store.SaveAsync(key, result, TimeSpan.FromHours(24), ct);
    return TypedResults.Ok(result);
});
```

Without this, a client retry on a timed-out `POST /payments` charges twice.

## REST vs gRPC vs GraphQL vs SOAP

| Need | Pick |
|---|---|
| Public API, browser/mobile clients, CDN-cacheable reads | **REST** |
| Internal microservice with typed contracts, low latency, streaming | **[gRPC](./grpc.md)** |
| Aggregating/composing many backends; client picks fields; mobile bandwidth-sensitive | **[GraphQL](./graphql-hotchocolate.md)** |
| Integrating with legacy enterprise/government/banking | **[SOAP](./soap.md)** (only if forced) |
| Fire-and-forget, async workflows, decoupled producers/consumers | Not REST — see [MESSAGING](../MESSAGING/) |

## Senior-level gotchas

- **`[ApiController]`** changes binding rules: complex types default to `[FromBody]`, simple to `[FromQuery]`/`[FromRoute]`. It also emits automatic `400` on `ModelState` errors — disable that response only if you fully own validation flow (`SuppressModelStateInvalidFilter = true`).
- Returning `Task<IActionResult>` makes OpenAPI guess your response shape from `[ProducesResponseType]` only. Returning `Results<Ok<T>, NotFound>` gives the OpenAPI generator the exact union — strictly better.
- `204 No Content` must have an empty body. `TypedResults.NoContent()` does it; don't `return Ok(null)` — that emits `null` as JSON and a `200`.
- A `PATCH` endpoint that takes the full DTO and overlays non-null fields is **not** JSON Patch (RFC 6902). It's JSON Merge Patch (RFC 7396) at best, "homegrown patch" at worst. Document which one, and implement `Microsoft.AspNetCore.JsonPatch` properly if a client asks for the standard.
- `ETag` must change when the representation changes, not when the entity changes — including content negotiation. A weak `W/"..."` ETag is fine for that.
- Don't return entity types directly — they leak EF navigation properties on serialization, drag in lazy loads inside the serializer (the worst place to throw), and couple your wire contract to your storage schema. Project to a DTO/record at the boundary.
- The default `JsonSerializerOptions.PropertyNameCaseInsensitive = false` on the server means a client sending `Id` to a property named `id` silently binds `default`. The .NET HTTP client default is the opposite. Match your contract.
