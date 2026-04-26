# OpenAPI

_Targets .NET 10 / C# 14. See also: [Swagger / Swashbuckle](./swagger.md), [Scalar](./scalar.md), [Minimal APIs](../ASPNET_CORE_BASICS/minimal-apis.md), [Controllers](../ASPNET_CORE_BASICS/controllers.md), [Problem Details](../ASPNET_CORE_BASICS/problem-details.md), [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md)._

OpenAPI is a JSON/YAML schema for describing HTTP APIs — paths, operations, parameters, request/response bodies, security schemes. Tooling consumes it to render UIs, generate clients, drive contract tests, and feed gateways. Since .NET 9, ASP.NET Core ships its own OpenAPI document generator (`Microsoft.AspNetCore.OpenApi`) in the default project template — Swashbuckle is no longer wired up by default.

## Spec vs tool — disambiguating the names

| Name | What it actually is |
|---|---|
| OpenAPI Specification | The JSON/YAML schema (currently 3.0.x and 3.1.x). Maintained by the OpenAPI Initiative. Was called the **Swagger Specification** until 2.0 → 3.0. |
| Swagger | A **brand**, owned by SmartBear. Refers to a family of tools (Swagger UI, Swagger Editor, SwaggerHub) — *not* the spec anymore. |
| Swashbuckle | Third-party .NET package that **generates** an OpenAPI document from your ASP.NET Core app. See [swagger.md](./swagger.md). |
| `Microsoft.AspNetCore.OpenApi` | Microsoft's first-party generator, in-box since .NET 9, default in templates from .NET 9 onwards. |
| Swagger UI / Scalar / Redoc | Renderers that consume an OpenAPI document and draw an interactive page. Independent of how the document was generated. |

The takeaway: **OpenAPI = format. Swashbuckle / `Microsoft.AspNetCore.OpenApi` = generator. Swagger UI / Scalar = UI.** Mixing these is a pick-list, not a stack.

## The in-box generator

```csharp
// Program.cs — minimal API
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();              // registers the document generator

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                       // serves /openapi/v1.json
}

app.MapGet("/products/{id:int}", (int id) => new { Id = id, Name = "Hat" })
   .WithName("GetProduct")
   .WithSummary("Fetch a product by id")
   .Produces<Product>(StatusCodes.Status200OK)
   .ProducesProblem(StatusCodes.Status404NotFound);

app.Run();
```

`AddOpenApi()` registers an `IDocumentProvider` keyed by document name (`"v1"` by default). `MapOpenApi()` exposes a route — `/openapi/{documentName}.json` by default — that materializes the document on demand. The default emits **OpenAPI 3.0**; .NET 10 supports 3.1 via `options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_1`.

`AddOpenApi()` does **not** add a UI. Pair it with [Scalar](./scalar.md) or Swagger UI separately.

## What it discovers automatically

Both minimal APIs and `[ApiController]`-attributed controllers feed the same `ApiExplorer` metadata pipeline. Out of the box the generator picks up:

| Source | What ends up in the doc |
|---|---|
| `MapGet`/`MapPost` route templates | Path + method |
| Parameter binding (`[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`) | Parameter location, name, schema |
| `Produces<T>()`, `[ProducesResponseType(typeof(T), 200)]` | Typed response schemas per status code |
| `Accepts<T>("application/json")` | Request body content type + schema |
| `WithName`, `WithTags`, `WithSummary`, `WithDescription` | `operationId`, tags, summary, description |
| `RequireAuthorization()`, `[Authorize]` | Security requirements (when a security scheme is registered) |
| `ProblemDetails` returns | `application/problem+json` response schemas |

For minimal APIs, **return type inference** only works when the handler returns `T`, `Results<...>` (typed unions from `Microsoft.AspNetCore.Http.HttpResults`), or `TypedResults.X(...)`. A bare `IResult` or `Results.Ok(...)` returns no type metadata — you must add `Produces<T>()` explicitly.

```csharp
// Inferred — generator sees Ok<Product> and NotFound arms.
app.MapGet("/products/{id:int}", Results<Ok<Product>, NotFound> (int id, IRepo r) =>
    r.Find(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());
```

## Transformers — the extensibility model

Three hook points, all on `OpenApiOptions`:

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<BearerSecurityDocumentTransformer>();
    options.AddOperationTransformer<DeprecationHeaderOperationTransformer>();
    options.AddSchemaTransformer<EnumAsStringSchemaTransformer>();
});
```

### `IOpenApiDocumentTransformer` — once per document

```csharp
internal sealed class BearerSecurityDocumentTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(
        OpenApiDocument document,
        OpenApiDocumentTransformerContext context,
        CancellationToken cancellationToken)
    {
        document.Components ??= new();
        document.Components.SecuritySchemes["Bearer"] = new()
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT",
        };
        document.SecurityRequirements.Add(new OpenApiSecurityRequirement
        {
            [new OpenApiSecurityScheme { Reference = new() { Id = "Bearer", Type = ReferenceType.SecurityScheme } }] =
                Array.Empty<string>(),
        });
        return Task.CompletedTask;
    }
}
```

### `IOpenApiOperationTransformer` — once per operation

```csharp
internal sealed class DeprecationHeaderOperationTransformer : IOpenApiOperationTransformer
{
    public Task TransformAsync(
        OpenApiOperation operation,
        OpenApiOperationTransformerContext context,
        CancellationToken cancellationToken)
    {
        if (context.Description.ActionDescriptor.EndpointMetadata.OfType<ObsoleteAttribute>().Any())
            operation.Deprecated = true;
        return Task.CompletedTask;
    }
}
```

### `IOpenApiSchemaTransformer` — once per schema

```csharp
internal sealed class EnumAsStringSchemaTransformer : IOpenApiSchemaTransformer
{
    public Task TransformAsync(
        OpenApiSchema schema,
        OpenApiSchemaTransformerContext context,
        CancellationToken cancellationToken)
    {
        if (context.JsonTypeInfo.Type.IsEnum)
        {
            schema.Type = "string";
            schema.Enum = [.. Enum.GetNames(context.JsonTypeInfo.Type).Select(n => (IOpenApiAny)new OpenApiString(n))];
        }
        return Task.CompletedTask;
    }
}
```

Transformers can be registered as types (DI-resolved per request) or as inline delegates. Document/operation transformers run on every request to the OpenAPI endpoint unless you cache the document yourself.

## XML documentation comments

.NET 10 added first-party support for surfacing XML comments via the build-time `Microsoft.Extensions.ApiDescription.Server` package — no runtime XML reader needed.

```xml
<!-- csproj -->
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>  <!-- silence "missing XML comment" -->
</PropertyGroup>
```

```csharp
/// <summary>Fetch a product by id.</summary>
/// <param name="id">The product identifier.</param>
/// <response code="200">The product was found.</response>
/// <response code="404">No product with that id exists.</response>
public record Product(int Id, string Name);
```

XML comments end up as `summary` / `description` on the operation, parameters, and schemas. For .NET 8/9 you need a third-party transformer or Swashbuckle's `IncludeXmlComments`.

## Multiple documents (versioning)

Register one document per version, route by `{documentName}`:

```csharp
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");

app.MapOpenApi("/openapi/{documentName}.json");

app.MapGet("/v1/products", () => ...).WithGroupName("v1");
app.MapGet("/v2/products", () => ...).WithGroupName("v2");
```

`WithGroupName(...)` (or `[ApiExplorerSettings(GroupName = "v2")]` on controllers) is what binds an endpoint to a document. For real version routing, integrate with `Asp.Versioning` — see [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md).

## Build-time generation

`Microsoft.Extensions.ApiDescription.Server` materializes the OpenAPI doc as a build artifact, so CI can publish it without booting the app:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.ApiDescription.Server" Version="*" />
</ItemGroup>

<PropertyGroup>
  <OpenApiGenerateDocuments>true</OpenApiGenerateDocuments>
  <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/openapi</OpenApiDocumentsDirectory>
</PropertyGroup>
```

`dotnet build` now writes `openapi/myproject.json`. Useful for:
- Contract testing in CI (diff against committed snapshot, fail on breaking changes).
- Generating typed clients (`NSwag`, `Kiota`, `openapi-typescript`) without a runtime probe.
- Publishing the spec to a docs site or developer portal.

## Comparison

| Generator | Maintainer | Default in template | Extensibility | Build-time output | UI bundled |
|---|---|---|---|---|---|
| `Microsoft.AspNetCore.OpenApi` | Microsoft | ✅ (.NET 9+) | Document/Operation/Schema transformers | ✅ via `ApiDescription.Server` | ❌ — pair with Scalar / Swagger UI |
| Swashbuckle.AspNetCore | Community (domaindrivendev) | ❌ since .NET 9 | Document/Operation/Schema/Parameter filters | Limited (CLI tool) | Bundled `SwaggerUI` package |
| NSwag | Community (RicoSuter) | ❌ | `Document/Operation` processors | ✅ (CLI + MSBuild) | Bundled UI + client gen |

The in-box generator is the right default for new .NET 9/10 projects. Swashbuckle remains excellent if you already depend on its filter ecosystem or want the bundled UI in a single package. NSwag is the choice when you want generator + client codegen in one toolchain.

**Senior-level gotchas:**
- **`MapOpenApi()` should be dev-only by default.** Gating with `if (app.Environment.IsDevelopment())` is the convention — exposing the spec in prod leaks endpoints, types, and security scheme hints to anyone who guesses the URL. If you need the spec in prod, gate it behind `RequireAuthorization()` with a dedicated policy.
- **No type inference for bare `IResult` / `Results.Ok(...)`.** The generator can't see the type. Either return `TypedResults.X(...)` / `Results<...>` (the typed unions in `Microsoft.AspNetCore.Http.HttpResults`) or add `Produces<T>(...)` / `[ProducesResponseType]` explicitly.
- **Document transformers run per request.** They're not cached. If a transformer does I/O (DB lookup, `HttpClient`), every refresh of the doc page re-runs it. Cache the materialized document yourself in a singleton if needed.
- **`AddOpenApi()` ≠ adds a UI.** This trips people coming from Swashbuckle, where `AddSwaggerGen` + `UseSwagger` + `UseSwaggerUI` was a single mental unit. Add Scalar (or `Swashbuckle.AspNetCore.SwaggerUI`) separately.
- **`[ApiController]` ProblemDetails wrappers** auto-emit `400`/`415`/`422` arms — they appear in the doc whether you declared them or not. To suppress, set `ConfigureApiBehaviorOptions(o => o.SuppressMapClientErrors = true)`.
- **Default emitted version is OpenAPI 3.0.** Some downstream tools (Spectral lint rules, certain client gens) assume 3.1 features like full JSON Schema nullable. Opt in with `options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_1` only after you verify the consumers handle it.
- **Servers list is auto-populated from the request URL.** A reverse-proxied app behind `https://api.example.com` may emit `https://internal-host:8080` if forwarded headers aren't configured. Override with a document transformer that sets `document.Servers` explicitly.
- **`launchSettings.json` only configures dev URLs.** Don't rely on it as the source of truth for the published `Servers` array — it's not deployed.
- **Polymorphic types need `JsonPolymorphic` + `JsonDerivedType` attributes** for the generator to emit `oneOf` / discriminator metadata. Without them you get the base-class schema only.
- **`[FromForm]` with `IFormFile`** needs the operation marked with `Accepts<T>("multipart/form-data")` for the file parameter to render correctly in the UI.
