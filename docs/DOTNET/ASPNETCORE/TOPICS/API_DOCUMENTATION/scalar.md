# Scalar

_Targets .NET 10 / C# 14. See also: [OpenAPI (in-box generator)](./open-api.md), [Swagger / Swashbuckle](./swagger.md), [Minimal APIs](../ASPNET_CORE_BASICS/minimal-apis.md), [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md), [JWT](../SECURITY/jwt.md), [OAuth 2.0](../SECURITY/oauth-2.md)._

Scalar is an open-source (MIT) interactive API reference renderer — the modern alternative to Swagger UI. `Scalar.AspNetCore` is the integration package that mounts the UI as ASP.NET Core middleware. It **does not generate** an OpenAPI document; it consumes one. The canonical .NET 9/10 setup pairs `Microsoft.AspNetCore.OpenApi` (the in-box generator) with Scalar (the UI). It's prettier than Swagger UI out of the box, ships nicer client snippets, and supports OpenAPI 3.1 features like full JSON Schema nullability.

## Setup

```csharp
// dotnet add package Microsoft.AspNetCore.OpenApi
// dotnet add package Scalar.AspNetCore

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();        // generator

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                 // serves /openapi/v1.json
    app.MapScalarApiReference();      // serves UI at /scalar/v1
}

app.MapGet("/products/{id:int}", (int id) => new { Id = id, Name = "Hat" })
   .WithName("GetProduct")
   .Produces<Product>();

app.Run();
```

`MapScalarApiReference()` defaults to reading `/openapi/{documentName}.json` and serves the UI at `/scalar/{documentName}`. There's no separate request/response cycle for the UI assets — the package returns a single HTML page that pulls Scalar's JS bundle from a CDN and renders client-side.

## Configuration

```csharp
app.MapScalarApiReference(options =>
{
    options.Title = "MyApi — interactive docs";
    options.Theme = ScalarTheme.BluePlanet;          // Saturn, Mars, Kepler, Mono, Solarized, ...
    options.DefaultHttpClient = new(ScalarTarget.CSharp, ScalarClient.HttpClient);
    options.HideModels = false;
    options.HideClientButton = false;
    options.OpenApiRoutePattern = "/openapi/{documentName}.json";  // where to fetch the doc
    options.Servers = [new ScalarServer("https://api.example.com")];
    options.CustomCss = "body { font-family: 'JetBrains Mono'; }";
    options.ShowSidebar = true;
    options.DarkMode = true;
});
```

| Option | Purpose |
|---|---|
| `Title` | Browser tab title |
| `Theme` | Built-in colour theme |
| `DefaultHttpClient` | Initial language/library for the "request snippet" pane |
| `HideModels` | Hide the schema list in the sidebar |
| `HideClientButton` | Suppress the "open in client" CTA |
| `OpenApiRoutePattern` | Doc URL (templated by `{documentName}`) |
| `Servers` | Override the doc's `servers` array (useful behind reverse proxies) |
| `CustomCss` | Inline CSS injected into the page |
| `Layout` | `Modern` (sidebar + content) or `Classic` (Stripe-style three-column) |

## Auth in the UI

Configure how the "Authorize" button behaves:

```csharp
app.MapScalarApiReference(options =>
{
    options.Authentication = new ScalarAuthenticationOptions
    {
        PreferredSecurityScheme = "Bearer",
        HttpBearer = new() { Token = "" },              // empty = user pastes
    };
});
```

For OAuth 2.0:

```csharp
options.Authentication = new ScalarAuthenticationOptions
{
    PreferredSecurityScheme = "OAuth2",
    OAuth2 = new()
    {
        ClientId = "scalar-ui",
        Scopes = ["openid", "profile", "api"],
    },
};
```

The security schemes themselves are declared on the OpenAPI document via the generator (a transformer in the in-box case — see [open-api.md](./open-api.md)). Scalar only configures **how the UI prompts for credentials**. See [JWT](../SECURITY/jwt.md) and [OAuth 2.0](../SECURITY/oauth-2.md) for the underlying flows.

## Multiple OpenAPI documents

```csharp
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");

app.MapOpenApi("/openapi/{documentName}.json");

app.MapScalarApiReference(options =>
{
    options.AddDocument("v1", "Public API v1");
    options.AddDocument("v2", "Public API v2", routePattern: "/openapi/v2.json");
});
```

Scalar's sidebar gets a doc switcher. Cross-reference [API Versioning](../ASPNET_CORE_BASICS/api-versioning.md) for endpoint grouping.

## Hosting model — CDN vs self-host

By default, Scalar's HTML page references the JS bundle from `cdn.jsdelivr.net`. That's fine for dev; for air-gapped, strict-CSP, or zero-third-party-dependency deployments, point at a self-hosted bundle:

```csharp
app.MapScalarApiReference(options =>
{
    options.CdnUrl = "/assets/scalar/standalone.js";  // self-hosted
    // or pin a specific version of the public CDN:
    // options.CdnUrl = "https://cdn.jsdelivr.net/npm/@scalar/api-reference@1.x.x";
});
```

You'll need to deploy `@scalar/api-reference`'s `standalone.js` as a static asset. The package doesn't bundle the JS — that's a deliberate choice to keep the NuGet small.

## Canonical .NET 10 setup

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((doc, ctx, ct) =>
    {
        doc.Info = new() { Title = "Catalog API", Version = "v1" };
        return Task.CompletedTask;
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference(options =>
    {
        options.Title = "Catalog API";
        options.Theme = ScalarTheme.BluePlanet;
        options.DefaultHttpClient = new(ScalarTarget.CSharp, ScalarClient.HttpClient);
    });
}

app.MapGet("/products", (int page = 1, int size = 20) => Results.Ok(Array.Empty<Product>()))
   .WithSummary("List products")
   .Produces<Product[]>();

app.Run();

public sealed record Product(int Id, string Name, decimal Price);
```

That's the entire .NET 10 OpenAPI + UI story: in-box generator emits the spec, Scalar renders it, dev-only by default. No filter classes, no XML packages, no CDN customization unless you need it.

## Comparison

| UI | Aesthetics | Search | Try it out | Theming | Asset hosting | OpenAPI 3.1 |
|---|---|---|---|---|---|---|
| **Scalar** | Modern, clean | Excellent | ✅ + multi-language snippets | 8+ themes, custom CSS | CDN by default, self-host opt-in | ✅ first-class |
| Swagger UI | Dated but familiar | Basic | ✅ | Limited (CSS only) | Bundled in `Swashbuckle.AspNetCore.SwaggerUI` | Partial |
| Redoc | Reference-style (read-heavy) | Excellent | ❌ in OSS (paid in Premium) | Themeable | Self-host or CDN | ✅ |
| RapiDoc | Compact, single-page | Good | ✅ | Theme attributes on `<rapi-doc>` element | Web component | ✅ |

For new code-first .NET projects: Scalar. For read-only public reference docs published to a docs site: Redoc. For minimum-friction "click 'Try it out' on the dev cluster": Swagger UI is fine and familiar.

**Senior-level gotchas:**
- **Dev-only by default.** Same discipline as Swagger UI — `MapScalarApiReference()` exposes your full API surface. Wrap in `if (app.Environment.IsDevelopment())` or `.RequireAuthorization("InternalOnly")`. The middleware composes normally.
- **CDN dependency = outbound network call from every viewer's browser.** A "private" Scalar page hosted on an internal URL still loads JS from `cdn.jsdelivr.net`. If your security model bans third-party asset loads (CSP `script-src 'self'`), self-host the bundle via `CdnUrl`.
- **CSP headers need the CDN whitelisted** when default-served. Add `script-src 'self' cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' cdn.jsdelivr.net` or self-host. Scalar's inline `<script type="application/json">` config block also requires `'unsafe-inline'` on `script-src` unless you use a nonce-based CSP — check the rendered HTML.
- **"Try it out" hits the real API.** Mounting Scalar at a host that's network-reachable to production — even on a "staging" subdomain that points at prod data — turns the UI into a one-click prod-modification tool. Pair with API auth scopes that distinguish read vs write.
- **`MapScalarApiReference()` is just middleware.** `RequireAuthorization()`, `RequireHost()`, `RequireCors()` all work on the returned `IEndpointConventionBuilder`. Use them.
- **Custom CSS is client-side, not a security boundary.** Anyone can dev-tools their way around it. Don't put secret URLs (signed asset endpoints, internal hostnames) in `CustomCss` thinking it's gated by your auth — the CSS is in the page source.
- **Scalar emits an OpenAPI 3.1-aware UI; the in-box generator emits 3.0 by default.** Visual rendering is fine, but advanced 3.1-only features — `webhooks`, full JSON Schema nullable (`type: ["string", "null"]`), `$schema` per-schema — won't appear until you set `options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_1` on the generator.
- **`OpenApiRoutePattern` must match what `MapOpenApi()` actually serves.** The defaults align (`/openapi/{documentName}.json`); if you customize one, customize both, or Scalar will fetch a 404 and show a blank page with no error.
- **Versioned-doc switcher requires `AddDocument(...)` per version.** Just having multiple `AddOpenApi("vN")` registrations isn't enough — Scalar only renders documents you've explicitly told it about.
- **Sidebar grouping comes from OpenAPI tags.** Use `WithTags("Products")` on minimal APIs or `[Tags("Products")]` on controllers to get sensible navigation. Without tags, every endpoint piles into "default."
- **CDN version pinning matters in CI.** A floating `@latest` URL means a Scalar JS update can change rendered output overnight, breaking visual regression tests. Pin the version in `CdnUrl`.
