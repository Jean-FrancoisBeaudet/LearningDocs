# Authentication and Authorization

_Targets .NET 10 / C# 14. See also: [Middlewares](./middlewares.md), [Controllers](./controllers.md), [Minimal APIs](./minimal-apis.md), [Configuration and Options Pattern](./configuration-and-options-pattern.md)._

**Authentication** answers *who is the caller*; **authorization** answers *are they allowed*. They are two separate middlewares in ASP.NET Core, registered in two separate DI calls, and they fail independently — `401 Unauthorized` (no/invalid identity) is not the same as `403 Forbidden` (identity present, policy denied). Both are claims-based: an authenticated request carries a `ClaimsPrincipal` with one or more `ClaimsIdentity` instances, and authorization policies inspect those claims to decide.

## The pipeline order is non-negotiable

```csharp
app.UseRouting();           // 1. matches an endpoint (so policies on the endpoint are visible)
app.UseAuthentication();    // 2. populates HttpContext.User from the configured scheme
app.UseAuthorization();     // 3. evaluates [Authorize] / RequireAuthorization()
app.MapControllers();
```

Forgetting `UseAuthentication()` is the #1 bug: `User.Identity.IsAuthenticated` silently returns `false` for everyone and `[Authorize]` rejects every request as if the bearer token were missing. Forgetting `UseAuthorization()` makes every `[Authorize]` a no-op and every endpoint anonymous.

## Authentication: schemes

A scheme is a `(handler, options)` pair registered with a string name. Most apps register one scheme:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.example.com/";
        options.Audience  = "api://orders";
        options.MapInboundClaims = false;                 // keep raw JWT claim names ("sub", not "nameidentifier")
        options.TokenValidationParameters = new()
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30),         // default is 5 minutes — too generous
        };
    });
```

For multi-scheme (e.g. an API that accepts both bearer tokens *and* session cookies), register both and let endpoints opt in:

```csharp
builder.Services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme    = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(/* ... */)
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, o =>
    {
        o.LoginPath        = "/login";
        o.SlidingExpiration = true;
        o.Cookie.SameSite  = SameSiteMode.Strict;
        o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    });

[Authorize(AuthenticationSchemes = "Cookies,Bearer")]     // accept either
public sealed class HybridController : ControllerBase { }
```

`MapInboundClaims = false` matters: by default `JwtBearerHandler` rewrites `sub` → `ClaimTypes.NameIdentifier`, `email` → `ClaimTypes.Email`, etc. — a legacy compatibility shim from `System.IdentityModel`. Modern code reads `"sub"` directly; the rewrite breaks dictionary lookups against the JWT spec.

## Claims, principals, identities

```csharp
ClaimsPrincipal user = httpContext.User;       // request-scoped, injected as well
string?         sub  = user.FindFirstValue("sub");
bool            isAdmin = user.IsInRole("admin");
IEnumerable<Claim> tenants = user.FindAll("tenant");
```

A `ClaimsPrincipal` may carry multiple `ClaimsIdentity` (one per authentication scheme that succeeded). `IsInRole`/`FindFirst` look across all identities. `Identity` (singular) is the *primary* identity — usually the first one added.

Issuing claims (after a sign-in form, OIDC callback, etc.):

```csharp
var identity = new ClaimsIdentity(
    claims: [new Claim("sub", userId), new Claim("role", "admin"), new Claim("tenant", "acme")],
    authenticationType: CookieAuthenticationDefaults.AuthenticationScheme);

await httpContext.SignInAsync(
    CookieAuthenticationDefaults.AuthenticationScheme,
    new ClaimsPrincipal(identity));
```

## OIDC for delegated sign-in

```csharp
.AddOpenIdConnect("oidc", o =>
{
    o.Authority    = "https://login.example.com/";
    o.ClientId     = builder.Configuration["Auth:ClientId"];
    o.ClientSecret = builder.Configuration["Auth:ClientSecret"];
    o.ResponseType = "code";                          // PKCE auth code flow
    o.UsePkce      = true;
    o.SaveTokens   = true;                            // store id_token/access_token in the cookie
    o.Scope.Add("openid"); o.Scope.Add("profile"); o.Scope.Add("email");
});
```

Pair OIDC with `AddCookie()` — OIDC handles sign-in, Cookie persists the resulting principal across requests. For server-rendered MVC/Razor apps that's the canonical setup; for SPAs use BFF (backend-for-frontend) or PKCE in the browser with a token-handling library.

## Authorization: four flavors

| Style | Where | Example |
|---|---|---|
| **Role** | Inline | `[Authorize(Roles = "admin,ops")]` |
| **Claim** | Policy | `RequireClaim("scope", "orders:write")` |
| **Policy** | Named, reusable | `[Authorize(Policy = "AdultsOnly")]` |
| **Resource-based** | Imperative, in-handler | `await authz.AuthorizeAsync(user, doc, "EditDocument")` |

Roles are a special-case claim (`ClaimTypes.Role`) — convenient but flat. Anything richer (multi-tenant, scope-based, hierarchical) becomes a policy.

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdultsOnly", p => p.RequireAssertion(ctx =>
        ctx.User.FindFirstValue("birthdate") is { } b
        && DateOnly.Parse(b).AddYears(18) <= DateOnly.FromDateTime(DateTime.UtcNow)));

    options.AddPolicy("OrdersWrite", p => p.RequireClaim("scope", "orders:write"));

    options.AddPolicy("TenantAdmin", p => p
        .RequireAuthenticatedUser()
        .RequireClaim("tenant_role", "admin"));

    // Default policy applies when [Authorize] has no policy specified.
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Fallback policy applies when an endpoint has no [Authorize] / [AllowAnonymous] at all.
    options.FallbackPolicy = options.DefaultPolicy;       // makes the API authenticated-by-default
});
```

The **default vs fallback** distinction trips people up:
- *Default policy*: what `[Authorize]` (no args) resolves to. Out of the box: `RequireAuthenticatedUser()`.
- *Fallback policy*: applied to endpoints that have **no authorization metadata at all**. Out of the box: `null` (anonymous). Setting it to `RequireAuthenticatedUser()` is the safest pattern for a private API — every new endpoint is locked by default unless explicitly `[AllowAnonymous]`.

## Custom requirements + handlers

When `RequireAssertion` doesn't fit (testable, DI-resolvable logic, or per-resource decisions):

```csharp
public sealed record MinimumAgeRequirement(int Age) : IAuthorizationRequirement;

public sealed class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        if (context.User.FindFirstValue("birthdate") is { } b
            && DateOnly.Parse(b).AddYears(requirement.Age) <= DateOnly.FromDateTime(DateTime.UtcNow))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
options.AddPolicy("18Plus", p => p.AddRequirements(new MinimumAgeRequirement(18)));
```

A requirement can have **multiple handlers**; if any calls `Succeed`, the requirement is satisfied. Don't call `Fail()` unless you want to *override* a sibling success — by default, simply not calling `Succeed` is enough to deny.

## Resource-based authorization

`[Authorize]` evaluates before the action runs, so it can't see the resource you're about to mutate:

```csharp
public sealed class DocumentHandler(IAuthorizationService authz) : Endpoint
{
    public async Task<IResult> Edit(Guid id, ClaimsPrincipal user, IDocRepo repo, CancellationToken ct)
    {
        var doc = await repo.FindAsync(id, ct);
        if (doc is null) return TypedResults.NotFound();

        var result = await authz.AuthorizeAsync(user, doc, "EditDocument");
        if (!result.Succeeded) return TypedResults.Forbid();

        // ... edit ...
    }
}
```

The handler receives both `User` and the resource (`AuthorizationHandler<TRequirement, TResource>`), so you can encode "owner only", "team member only", "tenant matches" rules.

## Endpoint-side syntax

```csharp
// Minimal API
app.MapGet("/orders", () => /* ... */).RequireAuthorization("OrdersWrite");
app.MapGet("/health", () => "OK").AllowAnonymous();

// Group-level
var secured = app.MapGroup("/admin").RequireAuthorization("Admin");

// Controller
[Authorize(Policy = "OrdersWrite")]
public sealed class OrdersController : ControllerBase
{
    [AllowAnonymous]
    [HttpGet("public-summary")] public IActionResult Summary() => Ok();
}
```

`[AllowAnonymous]` wins over `[Authorize]` at any level — including the fallback policy. It's the only escape hatch.

## Senior-level gotchas

- **`UseAuthentication()` missing** → `User.Identity.IsAuthenticated` is `false` for every request. No 401 from the auth middleware itself; the failure surfaces only when authorization rejects. Easy to miss because a vanilla `WebApplication.CreateBuilder` template doesn't include it for a no-auth scaffold.
- **Order matters: `UseAuthentication` → `UseAuthorization`.** Reversing them silently breaks policy evaluation — auth still works, but middleware that runs before `UseAuthentication` sees `User.Identity == null`.
- **Default vs Fallback policy is the security-by-default lever.** Setting `FallbackPolicy = RequireAuthenticatedUser()` is the difference between "every new endpoint is locked unless I write `[AllowAnonymous]`" and "every new endpoint is wide open unless I write `[Authorize]`". Pick the secure default.
- **`MapInboundClaims = true` (default!) rewrites `sub` → `ClaimTypes.NameIdentifier`.** If your code does `User.FindFirstValue("sub")`, it returns `null` and you blame the IdP. Set `MapInboundClaims = false` once, globally, and read raw JWT claim names everywhere.
- **`ClockSkew = TimeSpan.FromMinutes(5)` (default!)** means a token issued 5 minutes ago and "valid for 1 minute" still passes. Tighten it to 30 seconds for low-tolerance APIs. The flip side: too-tight skew breaks under client clock drift.
- **Resource-based policies cannot be expressed as `[Authorize]` attributes.** They require an `IAuthorizationService` call inside the action — there's no declarative alternative. Don't invent one with a custom filter that loads the resource twice.
- **`[Authorize(Roles = "admin")]` is a CSV match, not a logical AND.** `Roles = "admin,ops"` means "admin OR ops". For AND, use a policy that calls `RequireRole("admin").RequireRole("ops")`.
- **Multi-scheme requires explicit `AuthenticationSchemes` on `[Authorize]`** — without it, only the *default* scheme is consulted. A user with a valid cookie but no bearer token gets rejected on a Bearer-default API.
- **`SignOutAsync` for cookies needs the scheme name explicitly** — `httpContext.SignOutAsync()` (no args) signs out the *default* scheme, which may be Bearer (which can't sign out). Always pass `CookieAuthenticationDefaults.AuthenticationScheme`.
- **Sliding expiration on cookies updates the cookie in the response only when half the lifetime has elapsed.** A 1-hour sliding cookie hit at the 5-minute mark does not extend until minute 30. Surprises on session-timeout dashboards.
- **`IAuthorizationHandler` instances are singletons by default** if registered as such — capture-by-closure of request-scoped services is a captive dependency. Inject `IServiceProvider` and create a scope for resource-bound dependencies, or register the handler scoped (ASP.NET Core 8+ supports this).
- **`User.IsInRole(...)` walks every identity** and compares to `ClaimsIdentity.RoleClaimType`. If your IdP emits roles under a non-standard claim name (e.g. `roles`), `IsInRole` returns `false` until you map it via `TokenValidationParameters.RoleClaimType = "roles"`.
- **JWT validation does not check revocation.** A leaked token is valid until expiry. For high-value tokens use short lifetimes + refresh tokens, or implement a revocation list checked in a custom `JwtBearerEvents.OnTokenValidated`.
