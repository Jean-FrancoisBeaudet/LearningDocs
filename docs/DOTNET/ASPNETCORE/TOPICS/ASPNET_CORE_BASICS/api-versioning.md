# API Versioning

_Targets .NET 10 / C# 14. See also: [Controllers](./controllers.md), [Minimal APIs](./minimal-apis.md), [OpenAPI](../API_DOCUMENTATION/open-api.md), [REST](../API_COMMUNICATION/rest.md)._

HTTP has no built-in version semantics, so versioning is a contract you bolt on. You version when you ship a **breaking change**: removed field, renamed property, changed type, stricter validation, altered status code semantics. Additive changes (new optional field, new endpoint) don't need a version bump — and forcing one teaches clients to ignore your versions. The community-standard library is **`Asp.Versioning.Http`** (renamed from `Microsoft.AspNetCore.Mvc.Versioning` in 2022 — same maintainer, same APIs, new package id).

## Four placement strategies

| Where | Example | Cache-friendly | Browser-testable | Notes |
|---|---|---|---|---|
| **URL segment** | `GET /v1/orders/42` | ✅ (URL is the cache key) | ✅ | Easiest; forks the whole tree per version |
| **Query string** | `GET /orders/42?api-version=1.0` | ✅ if proxy keys on query | ✅ | Cheap to add to an unversioned API |
| **Header** | `api-version: 2.0` | ⚠️ needs `Vary: api-version` on CDN | ❌ (curl/Postman only) | Cleanest URIs; hardest to debug |
| **Media type** | `Accept: application/vnd.acme.order.v2+json` | ⚠️ `Vary: Accept` | ❌ | REST-purist; tooling friction usually outweighs aesthetic |

Pick one and stick with it. Mixing (e.g. URL **and** header) doubles the test surface and breaks negotiation rules. If you must support multiple, the library lets you compose readers (below).

## Wire-up

```csharp
using Asp.Versioning;

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;     // unversioned request → v1.0
    options.ReportApiVersions = true;                       // emits api-supported-versions / api-deprecated-versions headers
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("api-version"),
        new QueryStringApiVersionReader("api-version"));
})
.AddApiExplorer(o =>                                        // for OpenAPI
{
    o.GroupNameFormat = "'v'VVV";                           // → v1, v1.1, v2
    o.SubstituteApiVersionInUrl = true;                     // rewrites {version} in OpenAPI paths
});
```

`AssumeDefaultVersionWhenUnspecified = true` is comfortable but **trains clients to omit the version**, which then breaks the day you flip the default. For a public API, leave it `false` and reject unversioned requests with `400`.

## Minimal APIs

```csharp
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .HasDeprecatedApiVersion(new ApiVersion(1, 0))         // sunset header on v1
    .ReportApiVersions()
    .Build();

var v1 = app.MapGroup("/v{version:apiVersion}/orders").WithApiVersionSet(versionSet);

v1.MapGet("/{id:int}", (int id) => TypedResults.Ok(new { Id = id, Total = 0m }))
  .HasApiVersion(new ApiVersion(1, 0));

v1.MapGet("/{id:int}", (int id) => TypedResults.Ok(new { Id = id, TotalCents = 0L, Currency = "USD" }))
  .HasApiVersion(new ApiVersion(2, 0));
```

The `apiVersion` route constraint is registered automatically by `AddApiVersioning()`. Two handlers can share a route template — the library dispatches on declared version.

## Controllers

```csharp
[ApiController]
[ApiVersion(1.0)]
[ApiVersion(2.0)]
[Route("v{version:apiVersion}/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpGet("{id:int}"), MapToApiVersion(1.0)]
    public IActionResult GetV1(int id) => Ok(new { Id = id, Total = 0m });

    [HttpGet("{id:int}"), MapToApiVersion(2.0)]
    public IActionResult GetV2(int id) => Ok(new { Id = id, TotalCents = 0L, Currency = "USD" });
}

[ApiVersionNeutral]                                         // /health is not versioned
[Route("health")]
public sealed class HealthController : ControllerBase { /* ... */ }
```

`[ApiVersionNeutral]` is the right escape hatch for ops endpoints (`/health`, `/metrics`, `/openapi`). Without it, those endpoints respond with `400 Unsupported API Version` for any unversioned probe.

## Deprecation and sunset

```csharp
.HasDeprecatedApiVersion(new ApiVersion(1, 0))
```

With `ReportApiVersions = true`, every response includes:

```
api-supported-versions: 2.0
api-deprecated-versions: 1.0
```

For real sunset comms, also emit the [RFC 8594](https://www.rfc-editor.org/rfc/rfc8594) `Sunset` header on deprecated routes:

```csharp
v1.MapGet(...).AddEndpointFilter(async (ctx, next) =>
{
    ctx.HttpContext.Response.Headers.Append("Sunset", "Wed, 31 Dec 2025 23:59:59 GMT");
    ctx.HttpContext.Response.Headers.Append("Link", "<https://docs/api/v2>; rel=\"successor-version\"");
    return await next(ctx);
});
```

## OpenAPI: one document per version

```csharp
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");

app.MapOpenApi("/openapi/{documentName}.json");
```

Endpoints land in the right document by their declared `HasApiVersion(...)` (the `AddApiExplorer` step propagates that). The result is `/openapi/v1.json` and `/openapi/v2.json`, ready to feed Scalar/Swagger UI/Kiota independently. See [open-api](../API_DOCUMENTATION/open-api.md).

## Versioning across DTOs

The library handles **routing** to a version. It does **not** version your DTOs. Two common patterns:

1. **Suffixed types**: `OrderV1`, `OrderV2` records, mapped from the same domain entity. Verbose, but each contract is grep-able.
2. **Single type, conditional projection**: one `OrderResponse` plus a profile/projector that drops/adds fields per requested version. Less code, more risk of accidental cross-version leakage.

Pick (1) for breaking schema changes, (2) only for purely additive ones (which arguably don't need a version at all).

## Senior-level gotchas

- **Default version is a tar pit.** `AssumeDefaultVersionWhenUnspecified = true` + `DefaultApiVersion = 1.0` lets clients omit the header; when you later flip the default to `2.0`, every unversioned client breaks silently. Keep it `false` for any externally consumed API.
- **URL versioning forks the whole tree.** `/v1/...` and `/v2/...` share zero routing; a hot-fix on `/v1/orders/{id}` doesn't touch `/v2/...`. This is the *feature*, not a bug — but it means you maintain N copies of every shared filter/middleware.
- **Header versioning needs `Vary` on the CDN.** Without `Vary: api-version` (or `Vary: Accept` for media-type versioning) the CDN serves a `v2` body to a `v1` request and you debug it for a week.
- **`[ApiVersionNeutral]` is mandatory for `/health`, `/metrics`, `/openapi`.** Otherwise the version middleware rejects unversioned probes with `400`, and your ops dashboards turn red on day one.
- **Pre-release ordering is lexical, not semver.** `1.0-rc.10` sorts *before* `1.0-rc.2`. Use numeric pre-release segments only when you control them (`1.0-beta.001` style) or you'll route the wrong way.
- **`MapToApiVersion` does not coexist with `[NonAction]`.** A method tagged with both is silently dropped from the route table — no error, no log line. Add `[ApiExplorerSettings(IgnoreApi = false)]` to surface it during discovery.
- **Reader composition order matters.** `ApiVersionReader.Combine(url, header, query)` reads the URL first; if the segment is present, `?api-version=` is ignored. A client setting both gets the URL's version, not theirs — surprising during testing.
- **`api-supported-versions` is informational, not authoritative.** Clients can't programmatically discover routes from that header — they need OpenAPI for that. Don't treat it as a discovery mechanism.
- **Versioning != backward compat.** Both versions still hit the same database schema. A v2 column rename means migrating live data **and** keeping the v1 projection working until you sunset v1 — schema versioning and API versioning are independent disciplines.
