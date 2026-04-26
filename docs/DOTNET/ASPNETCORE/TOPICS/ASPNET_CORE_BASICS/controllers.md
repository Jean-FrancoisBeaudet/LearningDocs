# Controllers

_Targets .NET 10 / C# 14. See also: [Minimal APIs](./minimal-apis.md), [Routing](./routing.md), [Filters and Attributes](./filters-and-attributes.md), [Error Handling](./error-handling.md), [API Versioning](./api-versioning.md), [REST](../API_COMMUNICATION/rest.md)._

Controllers are the original ASP.NET Core endpoint authoring model: a class per resource, methods per operation, attribute routing, filters, model binding via attributes, ProblemDetails on validation failure. They feel heavyweight next to minimal APIs but they earn their keep when you have many endpoints sharing filters, conventions, or policies — the per-class scope is exactly what you want for cross-cutting concerns. The mental model: **`Controller` for views, `ControllerBase` for APIs, `[ApiController]` for sane defaults**.

## `Controller` vs `ControllerBase`

| Type | Use | Pulls in |
|---|---|---|
| `Controller` | MVC views, Razor, `View()`, `RedirectToAction()` | `ViewData`, `TempData`, antiforgery — bloat for an API |
| `ControllerBase` | JSON/XML APIs | Just routing, model binding, action results |

Pick `ControllerBase` for any API. `Controller` is for server-rendered HTML — and if that's what you're building, Razor Pages is usually a better fit than MVC views anyway.

## `[ApiController]` — what it changes

```csharp
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

`[ApiController]` opts the class into:
- **Automatic `400` on invalid `ModelState`** — bypasses your action body entirely.
- **`[FromBody]` inference** for complex parameters; `[FromQuery]`/`[FromRoute]` for simple ones.
- **Attribute routing required** — convention routing throws at startup.
- **ProblemDetails responses** (RFC 9457) for `400`/`415`, with the full RFC 9457 envelope.
- **Automatic multipart binding** for `IFormFile`.

Every API controller should have it. To turn off the auto-`400` (e.g. you want FluentValidation to own the response):

```csharp
builder.Services.Configure<ApiBehaviorOptions>(o => o.SuppressModelStateInvalidFilter = true);
```

## Action result types

Three return shape options, in increasing OpenAPI fidelity:

```csharp
// 1. IActionResult — opaque to OpenAPI, type comes only from [ProducesResponseType]
public IActionResult Get(int id) => Ok(new Order(id, ...));

// 2. ActionResult<T> — generic, OpenAPI infers T as the success type
public ActionResult<Order> Get(int id) => new Order(id, ...);

// 3. Results<T1, T2, ...> — typed union, OpenAPI infers each arm
public Results<Ok<Order>, NotFound> Get(int id)
    => repo.Find(id) is { } o ? TypedResults.Ok(o) : TypedResults.NotFound();
```

Prefer (3). The OpenAPI generator reads each `Results<...>` arm directly — no `[ProducesResponseType]` ceremony. `TypedResults.X(...)` (returning a concrete `Ok<T>`/`NotFound`/...) is what the union expects; `Results.Ok(...)` returns `IResult` and erases the type.

`ActionResult<T>` exists because `IActionResult` doesn't carry `T` for OpenAPI. Implicit conversions from `T` and `ActionResult` make it ergonomic, but it doesn't compose with multi-arm responses. For anything beyond "200 with body or non-200 without", reach for `Results<...>`.

## Routing

```csharp
[Route("orders")]                              // class-level prefix
public sealed class OrdersController : ControllerBase
{
    [HttpGet]                                  // GET /orders
    [HttpGet("{id:int:min(1)}")]               // GET /orders/123
    [HttpPost]                                 // POST /orders
    [HttpDelete("{id:int}")]                   // DELETE /orders/123

    [HttpGet("by-customer/{customerId:guid}")] // route constraints inline
    [HttpGet("recent", Name = "GetRecent")]    // named for url generation
}
```

`[Route("[controller]")]` is the auto-naming convention (`OrdersController` → `/orders`). It saves a copy-paste but breaks if you ever rename the class — search-replace pain. Prefer the explicit string for stability.

Convention routing (`MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}")`) is the MVC heritage; `[ApiController]` forbids it. For an API, attribute routing is the only sensible option.

## Model binding sources

| Attribute | Reads from |
|---|---|
| `[FromRoute]` | Route values (`/orders/{id}`) |
| `[FromQuery]` | `?key=value` |
| `[FromHeader]` | Request headers |
| `[FromBody]` | Request body (one per action; JSON by default) |
| `[FromForm]` | `multipart/form-data` or `application/x-www-form-urlencoded` |
| `[FromServices]` | DI container — preferable to constructor injection for per-action deps |
| `[AsParameters]` | Bind a struct/record's properties from multiple sources |

With `[ApiController]`, complex types default to `[FromBody]` and simple types to route/query — explicit attributes only when you need to override (e.g. read a complex object from query string with `[FromQuery]`).

```csharp
public sealed record OrderQuery(
    [FromQuery] int Page,
    [FromQuery] int Size,
    [FromQuery(Name = "sort")] string? SortKey);

[HttpGet]
public Ok<Order[]> List([AsParameters] OrderQuery query)
    => TypedResults.Ok(repo.List(query.Page, query.Size, query.SortKey));
```

## Validation

```csharp
public sealed record CreateOrder(
    [Required, StringLength(50)] string Customer,
    [Range(0.01, double.MaxValue)] decimal Total);

[HttpPost]
public Results<Created<Order>, ValidationProblem> Create(CreateOrder dto)
{
    // [ApiController] already returned 400 if ModelState was invalid.
    var order = service.Create(dto);
    return TypedResults.Created($"/orders/{order.Id}", order);
}
```

DataAnnotations is built-in but limited. For real validation logic (cross-field, conditional, async), use **FluentValidation** with an endpoint filter. Don't push validation into the action body — the action shouldn't know how to be invalid.

## Filters

Five hook points, in execution order:
1. **Authorization filters** (`[Authorize]`) — first, can short-circuit.
2. **Resource filters** (`IResourceFilter`) — wrap the entire action.
3. **Action filters** (`IActionFilter`) — wrap the action method.
4. **Exception filters** — only fire on action exceptions (see [error-handling](./error-handling.md)).
5. **Result filters** — wrap result execution.

Endpoint filters (minimal API) are the modern equivalent and compose cleaner. For controllers you're stuck with the MVC filter pipeline — see [filters-and-attributes](./filters-and-attributes.md).

## Conventions

`IApplicationModelConvention` lets you mutate the entire controller model at startup — apply `[Authorize]` to every controller in a namespace, change route prefixes, register filters by attribute. Powerful, easy to overuse:

```csharp
public sealed class AddVersionPrefixConvention : IApplicationModelConvention
{
    public void Apply(ApplicationModel application)
    {
        foreach (var controller in application.Controllers)
            controller.Selectors.First().AttributeRouteModel = new(
                new RouteAttribute($"v1/{controller.Selectors.First().AttributeRouteModel?.Template}"));
    }
}

builder.Services.AddControllers(o => o.Conventions.Add(new AddVersionPrefixConvention()));
```

Keep conventions close to where the rule is enforced. Cross-cutting metadata that's invisible from the controller code is the #1 source of "why does this endpoint behave that way" bug reports.

## Controllers vs Minimal APIs

| Concern | Controllers win | Minimal APIs win |
|---|---|---|
| Many endpoints sharing the same filters/policies | ✅ class-level attrs | needs `MapGroup` chains |
| Convention-driven cross-cutting | ✅ `IApplicationModelConvention` | endpoint filters per group |
| Single endpoint or small service | overkill | ✅ |
| AOT / trim | ⚠️ reflection-heavy | ✅ source-generated |
| `[ApiController]` model-state ergonomics | ✅ | manual via filter |
| Polymorphism / inheritance for shared logic | ✅ base controller class | composition only |
| Lowest startup time | ⚠️ | ✅ |
| Testability of a single handler | wrap controller | ✅ pure function |

There's no winner — both are first-class, and a real codebase mixes them: controllers for the resource-rich CRUD surface, minimal APIs for `/health`, `/version`, webhooks, and one-off integrations. See [minimal-apis](./minimal-apis.md).

## Senior-level gotchas

- **`[ApiController]`'s auto-`400` runs *before* your action.** Your handler code never sees an invalid model. If you log "received request X" at the top of every action, you'll never log invalid requests — they short-circuit. Add an `IActionFilter` or middleware for request logging instead.
- **Returning EF entities directly is a footgun.** Navigation properties get serialized — including lazy-loaded ones, which throw inside the JSON serializer (the worst place to throw because the response status is already `200` and headers are sent). Project to a DTO/record at the boundary.
- **`ActionResult<T>` ⇍ `IActionResult`.** They don't implicit-convert between each other. Pick one return shape per action and stick with it. `Results<...>` doesn't suffer this — prefer it.
- **Primary constructors on controllers work but the parameters are scoped per-request.** That's correct — controllers are transient — but `private readonly` field semantics can mislead a reader into thinking they're singleton-stable. They're not.
- **`AddControllers()` vs `AddMvc()` vs `AddControllersWithViews()` vs `AddRazorPages()`** are not interchangeable. Each pulls in more middleware. For an API, `AddControllers()` is the smallest correct choice. `AddMvc()` is rarely right today.
- **`[NonController]` opts a class out of discovery.** Necessary if you have a base class named `*Controller` that you don't want routed (e.g. an internal helper that just inherits `ControllerBase` for `Ok()`/`NotFound()` ergonomics).
- **`[Consumes("application/json")]` and `[Produces("application/json")]` shape OpenAPI but also enforce content negotiation at runtime.** A `[Consumes("application/json")]` action returns `415` for `application/xml` — surprising if you added the attribute "for OpenAPI" only.
- **Controller name collisions across namespaces are silent.** Two `OrdersController` in different namespaces both register; routing picks based on attribute uniqueness. The `[ApiExplorer]` group then has duplicate operationIds and OpenAPI codegen breaks. Add `Name` to `[HttpGet(...)]` to disambiguate, or use `[ApiExplorerSettings(GroupName = "...")]`.
- **Async void on a controller action compiles** (it inherits no constraint) but is fatal — exceptions tear down the process. Always `Task` / `Task<T>` / `ValueTask<T>`.
- **`CancellationToken` parameter is bound automatically** to `HttpContext.RequestAborted`. Always include it on async actions; without it, an aborted client still triggers your full handler — you're doing work no one is waiting for.
- **`[FromBody]` parameter on a `GET`** silently never binds (HTTP spec — GET has no semantic body). The framework doesn't error, you just get `null`. Use `[FromQuery]` + `[AsParameters]` for query objects.
- **Model binding silently swallows `JsonException`** when binding the body and surfaces it as a `ModelState` error. Your custom JSON converter throws? Client gets `400` with no detail. Hook `JsonOptions.SerializerOptions.Converters` and add explicit error messages, or add a custom `IInputFormatter`.
