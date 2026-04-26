# Problem Details

_Targets .NET 10 / C# 14. See also: [Error Handling](./error-handling.md), [Controllers](./controllers.md), [Minimal APIs](./minimal-apis.md), [REST](../API_COMMUNICATION/rest.md)._

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) (which obsoletes RFC 7807) defines a canonical, machine-readable shape for HTTP error responses: `application/problem+json`. ASP.NET Core ships first-class support — stop inventing your own `{"error": "...", "code": "..."}` envelope.

## The shape

```json
{
  "type":     "https://example.com/probs/out-of-stock",
  "title":    "Out of stock",
  "status":   409,
  "detail":   "Item SKU-42 has 0 units available.",
  "instance": "/orders/123",
  "traceId":  "00-c0c0...-01",
  "errors":   { "quantity": ["must be > 0"] }
}
```

Standard members: `type` (URI identifying the *class* of problem), `title` (short human summary), `status` (HTTP status code), `detail` (specific to *this* occurrence), `instance` (URI of the failing resource). Anything else is an *extension* member — `traceId`, `errors`, your own `errorCode`, etc.

Clients should branch on `type` (or `status`), never on `title` (which can be localized).

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails();    // canonical responses + IProblemDetailsService

var app = builder.Build();

app.UseExceptionHandler();    // unhandled exception → ProblemDetails
app.UseStatusCodePages();     // bare 4xx/5xx with no body → ProblemDetails

app.MapGet("/oops", () => { throw new InvalidOperationException("nope"); });
app.Run();
```

What each piece does:

- `AddProblemDetails()` — registers `IProblemDetailsService` and the default factory. **Without it, status-code-only responses (404, 415) have empty bodies.**
- `UseExceptionHandler()` — catches unhandled exceptions and uses the service to write a 500 ProblemDetails. **Pair with `AddProblemDetails()` — the handler alone won't produce JSON.**
- `UseStatusCodePages()` — materializes a body for responses that left the pipeline with a status code but no body (404 from routing, 405 method-not-allowed, etc.).

Order: exception handler outermost (catches everything downstream), then status code pages, then auth, routing, endpoints.

## Producing problem responses explicitly

Minimal APIs:

```csharp
app.MapGet("/orders/{id:int}", async Task<Results<Ok<Order>, ProblemHttpResult>>
    (int id, IOrderRepo repo, CancellationToken ct) =>
{
    var order = await repo.FindAsync(id, ct);
    if (order is null)
        return TypedResults.Problem(
            type:       "https://example.com/probs/order-not-found",
            title:      "Order not found",
            detail:     $"No order with id {id}",
            statusCode: 404);

    return TypedResults.Ok(order);
});

app.MapPost("/orders", (CreateOrderRequest req) =>
{
    var errors = new Dictionary<string, string[]>();
    if (req.Total <= 0) errors["total"] = ["must be > 0"];
    if (errors.Count > 0) return Results.ValidationProblem(errors);
    /* ... */
});
```

Controllers inherit `Problem(...)` and `ValidationProblem(...)` from `ControllerBase`:

```csharp
return Problem(
    detail:     "Item out of stock",
    statusCode: 409,
    type:       "https://example.com/probs/out-of-stock",
    title:      "Out of stock");
```

## Validation problem details

`ValidationProblemDetails` extends `ProblemDetails` with an `errors` dictionary. `[ApiController]` controllers emit it automatically when `ModelState` is invalid (status 400). Minimal APIs with `AddValidation()` (.NET 10) do the same.

```json
{
  "type":   "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title":  "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": ["The Email field is required."],
    "Total": ["The field Total must be between 0.01 and 1000000."]
  }
}
```

## Customizing globally

Add fields to every problem response:

```csharp
builder.Services.AddProblemDetails(opts =>
{
    opts.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"]  = Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["instance"] = ctx.HttpContext.Request.Path.Value;
        ctx.ProblemDetails.Extensions["machine"]  = Environment.MachineName;
    };
});
```

## Mapping domain exceptions

Map specific exception types to specific status codes via the exception handler:

```csharp
app.UseExceptionHandler(eh => eh.Run(async ctx =>
{
    var ex = ctx.Features.Get<IExceptionHandlerPathFeature>()!.Error;
    var (status, type, title) = ex switch
    {
        OutOfStockException         => (409, "https://example.com/probs/out-of-stock",   "Out of stock"),
        ConcurrencyException        => (409, "https://example.com/probs/concurrency",    "Concurrent update"),
        UnauthorizedAccessException => (403, (string?)null,                              "Forbidden"),
        _                           => (500, (string?)null,                              "Internal Server Error"),
    };
    await Results.Problem(detail: ex.Message, statusCode: status, title: title, type: type)
                 .ExecuteAsync(ctx);
}));
```

Or use `IExceptionHandler` (.NET 8+), preferred for type safety and DI:

```csharp
public sealed class OutOfStockHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not OutOfStockException oos) return false;
        await Results.Problem(detail: oos.Message, statusCode: 409,
            type: "https://example.com/probs/out-of-stock").ExecuteAsync(ctx);
        return true;
    }
}

builder.Services.AddExceptionHandler<OutOfStockHandler>();
builder.Services.AddProblemDetails();
```

Multiple handlers run in registration order; the first to return `true` wins. Register specific handlers before generic ones.

## Hiding sensitive details in production

In `Development`, the default exception handler includes the exception type and a stack trace; in `Production` it strips them. Verify by setting `ASPNETCORE_ENVIRONMENT=Production` and re-throwing — the response body should contain *no* internal information beyond `traceId`. Don't put PII, SQL fragments, or connection-string details in `Detail`.

## Senior-level gotchas

- `AddProblemDetails()` alone changes nothing about exceptions — it just registers `IProblemDetailsService`. You must pair it with `UseExceptionHandler()` (or `UseDeveloperExceptionPage()` in dev) to actually convert thrown exceptions to ProblemDetails.
- `[ApiController]` triggers automatic 400 ProblemDetails when `ModelState` is invalid. Disable only via `SuppressModelStateInvalidFilter = true`, and only when you fully own the validation flow — the default is correct for ~99% of services.
- `UseExceptionHandler` runs *after* the response has started writing if any byte has been flushed. Past the first `Response.Body.WriteAsync`, the connection just terminates — there's nothing left to convert. Don't write partial responses before validation completes.
- `traceId` should be `Activity.Current?.Id` (W3C TraceContext format), not a fresh GUID. The W3C `traceparent` header lets log aggregators correlate the request across services. Generating a new id on each problem response breaks distributed tracing.
- Clients pattern-matching on `title` is a bug waiting to happen — `title` is meant to be human-readable and may be localized. Branch on `type` (a stable URI) or `status`. Document the `type` URIs as part of your API contract.
- A `type` of `"about:blank"` (the spec default) means "no specific type" — the client falls back to `status`. Don't omit `type` (the field is technically valid empty) when you have a specific class of error; mint a real URI.
- `ValidationProblemDetails.errors` keys default to **PascalCase property names** but JSON serialization converts to camelCase by default. The mismatch confuses client-side form binders — set `ApiBehaviorOptions.ClientErrorMapping` and a consistent naming policy across both.
- Calling `Problem(...)` from a controller without `[ApiController]` produces an `ObjectResult` that bypasses `IProblemDetailsService` — your global `CustomizeProblemDetails` won't run. Either add `[ApiController]` or use `Results.Problem(...)` (which always flows through the service).
- Don't expose `Exception.Message` directly in `Detail` for unmapped 500s — message text often leaks file paths, connection strings, or row counts. Map known domain exceptions explicitly; for unmapped, return a generic "An error occurred" with the `traceId` so support can correlate against logs.
- `ProblemDetails` (the type) is mutable and serialized via `System.Text.Json` like any other class. If you publish your own derived type with extra properties, register it with the JSON source generator for AOT — otherwise reflection-based serialization triggers trim warnings.
