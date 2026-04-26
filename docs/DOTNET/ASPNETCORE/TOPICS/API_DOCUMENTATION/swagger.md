# Swagger / Swashbuckle

_Targets .NET 10 / C# 14. See also: [OpenAPI (in-box generator)](./open-api.md), [Scalar](./scalar.md), [Controllers](../ASPNET_CORE_BASICS/controllers.md), [Minimal APIs](../ASPNET_CORE_BASICS/minimal-apis.md), [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md), [JWT](../SECURITY/jwt.md)._

"Swagger" is a brand, not a tool. In .NET work it usually means one of three things: (1) the JS UI rendered at `/swagger`, (2) the `Swashbuckle.AspNetCore` NuGet package that generates the OpenAPI document, or (3) the spec itself (renamed to OpenAPI in 3.0 — see [open-api.md](./open-api.md)). Since .NET 9, Swashbuckle is **no longer in the default project template** — Microsoft's in-box generator replaced it. Swashbuckle is still maintained, still production-grade, and still the right pick for projects that depend on its filter ecosystem or want UI + generator in a single package.

## Names that overlap

| Term | What it actually is |
|---|---|
| Swagger Specification | Old name for OpenAPI 2.0. Deprecated as a name; the format lives on as OpenAPI 3.x. |
| Swagger UI | Open-source JS app (Apache 2.0) that renders an OpenAPI document into an interactive page. Hosted by `Swashbuckle.AspNetCore.SwaggerUI`. |
| Swagger Editor | Browser-based YAML/JSON editor for hand-writing OpenAPI docs. Mostly irrelevant in code-first .NET workflows. |
| Swashbuckle (.AspNetCore) | Community .NET package (`domaindrivendev/Swashbuckle.AspNetCore`) that generates an OpenAPI document from your ASP.NET Core app and ships Swagger UI middleware. |
| SwaggerHub | SmartBear's hosted SaaS for OpenAPI design + governance. Paid. |

When somebody says "add Swagger to the project," they usually mean "add Swashbuckle + Swagger UI." Be explicit in design discussions — the spec, the generator, and the renderer are three independent decisions.

## Swashbuckle.AspNetCore — canonical setup

```csharp
// dotnet add package Swashbuckle.AspNetCore

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "MyApi", Version = "v1" });
});

// ...

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();                         // serves /swagger/v1/swagger.json
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "MyApi v1");
        c.RoutePrefix = "swagger";            // UI at /swagger
    });
}
```

`AddSwaggerGen` is the **document generator**. `UseSwagger` is the route that exposes the JSON. `UseSwaggerUI` is the **UI** middleware. They're three separate concerns — you can mix and match (e.g. UI without the generator, see below).

## Filters — the extensibility model

Swashbuckle's hook points predate the in-box transformer model and cover more axes:

| Interface | Runs | Used for |
|---|---|---|
| `IDocumentFilter` | Once per document generation | Add tags, servers, security, global responses |
| `IOperationFilter` | Once per operation | Add headers, modify summaries, conditional auth requirements |
| `ISchemaFilter` | Once per schema | Tweak property formats, enum rendering, examples |
| `IParameterFilter` | Once per parameter | Adjust descriptions, examples |
| `IRequestBodyFilter` | Once per request body | Modify content types, examples |

```csharp
// Add an Authorization header to every operation
internal sealed class AuthHeaderOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters ??= new List<OpenApiParameter>();
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Tenant-Id",
            In = ParameterLocation.Header,
            Required = false,
            Schema = new OpenApiSchema { Type = "string" },
        });
    }
}

// Register
builder.Services.AddSwaggerGen(c => c.OperationFilter<AuthHeaderOperationFilter>());
```

Filter execution order is determined by registration order. There's no built-in priority system — order matters and is fragile when filters depend on each other's output.

## XML comments + annotations

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

```csharp
builder.Services.AddSwaggerGen(c =>
{
    var xml = Path.Combine(AppContext.BaseDirectory, $"{Assembly.GetExecutingAssembly().GetName().Name}.xml");
    c.IncludeXmlComments(xml, includeControllerXmlComments: true);
});
```

For richer metadata that XML can't express, add `Swashbuckle.AspNetCore.Annotations`:

```csharp
[SwaggerOperation(
    Summary = "Fetch a product by id",
    Description = "Returns a single product or 404 if not found.",
    OperationId = "GetProduct",
    Tags = new[] { "Products" })]
[SwaggerResponse(200, "The product", typeof(Product))]
[SwaggerResponse(404, "Not found")]
[HttpGet("{id:int}")]
public IActionResult Get(int id) => ...;
```

Annotations are explicit but verbose — XML comments are usually enough.

## Multiple docs / versioning

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "MyApi", Version = "v1" });
    c.SwaggerDoc("v2", new OpenApiInfo { Title = "MyApi", Version = "v2" });
    c.DocInclusionPredicate((docName, apiDesc) => apiDesc.GroupName == docName);
});

app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "v1");
    c.SwaggerEndpoint("/swagger/v2/swagger.json", "v2");
});
```

Bind endpoints to a doc with `[ApiExplorerSettings(GroupName = "v2")]` (controllers) or `WithGroupName("v2")` (minimal APIs). For real version routing integrate `Asp.Versioning.Mvc.ApiExplorer` — see [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md).

## Securing the UI

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        [new OpenApiSecurityScheme { Reference = new() { Id = "Bearer", Type = ReferenceType.SecurityScheme } }] =
            Array.Empty<string>(),
    });
});
```

This adds the **"Authorize" button** to Swagger UI. The user pastes a JWT, the UI then sends it as a header on every "Try it out". See [JWT](../SECURITY/jwt.md) for token issuance.

To gate the UI itself behind auth in non-dev environments:

```csharp
app.MapSwagger().RequireAuthorization("AdminOnly");   // requires Swashbuckle 6+
// Or wrap the middleware:
app.UseWhen(
    ctx => ctx.Request.Path.StartsWithSegments("/swagger"),
    branch => branch.UseAuthentication().UseAuthorization());
```

## Swagger UI on top of `Microsoft.AspNetCore.OpenApi`

The UI and the generator are **not** coupled. You can use the in-box generator and still serve Swagger UI by referencing `Swashbuckle.AspNetCore.SwaggerUI` only:

```csharp
// dotnet add package Microsoft.AspNetCore.OpenApi
// dotnet add package Swashbuckle.AspNetCore.SwaggerUI

builder.Services.AddOpenApi();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                                       // serves /openapi/v1.json
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/openapi/v1.json", "MyApi v1");  // point UI at the new generator
        c.RoutePrefix = "swagger";
    });
}
```

This is a useful migration step: keep the familiar UI, drop the Swashbuckle generator, move filters → transformers incrementally.

## Migrating Swashbuckle → in-box generator

Practical checklist:

1. Add `Microsoft.AspNetCore.OpenApi`, replace `AddSwaggerGen()` with `AddOpenApi()`.
2. Replace `UseSwagger()` with `MapOpenApi()` — note the route changes from `/swagger/v1/swagger.json` to `/openapi/v1.json`.
3. Replace each `IOperationFilter` / `IDocumentFilter` / `ISchemaFilter` with the equivalent transformer (see [open-api.md](./open-api.md)). The shapes are similar but the metadata access patterns differ.
4. `IncludeXmlComments` → drop it; .NET 10's in-box XML comment support runs at build time via `Microsoft.Extensions.ApiDescription.Server`.
5. Test the UI: if you keep Swagger UI, repoint `SwaggerEndpoint(...)`; if you switch to Scalar, see [scalar.md](./scalar.md).
6. Verify auth flows in the UI — `AddSecurityDefinition` becomes a document transformer.
7. Diff the generated JSON against the old one. Expect minor ordering / null-handling differences; track significant schema diffs as breaking changes for clients.

## Comparison

| Axis | `Microsoft.AspNetCore.OpenApi` | Swashbuckle.AspNetCore |
|---|---|---|
| Maintainer | Microsoft (in-box) | Community |
| Default in template | ✅ since .NET 9 | ❌ since .NET 9 |
| Extensibility | 3 transformer types | 5 filter types (more granular) |
| UI bundled | ❌ | ✅ via `.SwaggerUI` package |
| Build-time output | ✅ via `ApiDescription.Server` | Limited (separate CLI) |
| OpenAPI 3.1 | Opt-in (.NET 10) | Opt-in (`UseOpenApiVersion`) |
| Maturity | New (.NET 9+) | 10+ years, battle-tested |
| Custom example generation | Manual via transformer | First-class via `IExamplesProvider` (extension packages) |

For new .NET 9/10 projects, default to the in-box generator + Scalar UI. Pick Swashbuckle when you have a deep filter library, need its example generators, or want a single package for generator + UI.

**Senior-level gotchas:**
- **Don't run both generators in the same app.** Swashbuckle and `Microsoft.AspNetCore.OpenApi` each maintain their own document. Two endpoints, two slightly-different JSONs, two sources of truth — confusion guaranteed. Pick one.
- **`OperationFilter` ordering is fragile.** Filters mutate the same `OpenApiOperation`; if filter B depends on filter A having added a parameter, registration order silently determines correctness. Document the order explicitly in code.
- **`[ApiExplorerSettings(IgnoreApi = true)]`** on a controller or action excludes it from the doc — useful for internal endpoints. Minimal APIs use `.ExcludeFromDescription()`.
- **"Try it out" + cookie auth = CSRF risk.** The UI sends real requests. If the API uses cookie auth and lacks CSRF protection, anyone with access to `/swagger` can trigger state-changing calls as the logged-in user. Either disable Try It Out (`c.SupportedSubmitMethods(...)` empty list) or enforce anti-forgery on mutating endpoints.
- **`EnableTryItOutByDefault()` in shared environments** trains users to fire calls without thinking. Leave it off; let them click.
- **Default UI URL is `/swagger`, JSON is `/swagger/v1/swagger.json`.** Both are guessable. If you must expose the UI in prod (you usually shouldn't), at minimum change `RoutePrefix` and route the JSON to a non-default path.
- **CSP headers can break the UI.** Swagger UI uses inline scripts/styles; a strict `Content-Security-Policy` blocks them. Either relax CSP for the `/swagger/*` path or use Scalar/Redoc which are easier to harden.
- **Swashbuckle 6+ stopped pinning the OpenAPI spec version.** Default emitted version may shift across minor releases. Pin explicitly with `c.OpenApiVersion = OpenApiSpecVersion.OpenApi3_0` if downstream tools care.
- **`SchemaFilter` runs per-type, not per-property.** If you need per-property tweaks (e.g. format an `Email` property as `string` with `format: email`), use `[JsonPropertyName]` + `[StringSyntax(StringSyntaxAttribute.EmailAddress)]` or write an `IOperationFilter` that walks the parameters.
- **`IncludeXmlComments` requires `<GenerateDocumentationFile>true</GenerateDocumentationFile>`** in the csproj **and** the XML file present at `BaseDirectory` at runtime — easy to miss in containerized publishes that strip non-DLL files.
- **Don't ship the UI to production by default.** It's fine for staging, internal tools, or behind auth — but a public `/swagger` is a roadmap of your attack surface. Gate with `IsDevelopment()` or `RequireAuthorization()`.
