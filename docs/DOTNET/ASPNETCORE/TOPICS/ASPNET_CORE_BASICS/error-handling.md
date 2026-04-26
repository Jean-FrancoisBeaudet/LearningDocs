# Error Handling

_Targets .NET 10 / C# 14. See also: [Problem Details](./problem-details.md), [Middlewares](./middlewares.md), [Filters and Attributes](./filters-and-attributes.md), [Logging](./logging.md), [Controllers](./controllers.md)._

ASP.NET Core gives you four places to catch an exception, and the right answer is *almost always the same one*: a global exception middleware. Try/catch in handlers is for *expected* failure modes you can recover from; everything else should bubble to the global handler so you have a single place that owns the response shape, the log line, and the trace correlation. Since .NET 8, the idiomatic implementation uses `IExceptionHandler` services — pluggable, ordered, DI-friendly — paired with `IProblemDetailsService` for an [RFC 9457](https://datatracker.ietf.org/doc/html/rfc9457) response envelope.

## Where exceptions can be caught

| Mechanism | Scope | Use it for |
|---|---|---|
| `try/catch` in the handler | One call site | Expected, recoverable failures (`HttpRequestException` you'll retry, validation you'll surface) |
| MVC exception filters (`IExceptionFilter`, `IAsyncExceptionFilter`) | MVC pipeline only | Legacy MVC apps; **does not catch middleware exceptions** |
| Endpoint filters | One endpoint or group | Per-endpoint cross-cutting (auth probes, idempotency) |
| `UseExceptionHandler` + `IExceptionHandler` | Global, every request | Default. Owns the response envelope. |

`IExceptionFilter` is a trap on modern code: it only fires when MVC's action invoker is on the stack, so an exception thrown from a middleware, an endpoint filter, or model binding skips it entirely. Use the global middleware.

## Wire-up: the modern shape

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails(o =>                       // RFC 9457 envelope
{
    o.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Instance = null;                   // don't echo the route — info disclosure
    };
});

builder.Services.AddExceptionHandler<DomainExceptionHandler>();   // first match wins
builder.Services.AddExceptionHandler<FallbackExceptionHandler>(); // catch-all

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler();        // before everything else
    app.UseHsts();
}
else
{
    app.UseDeveloperExceptionPage();  // dev only — leaks stacks
}

app.UseStatusCodePages();             // converts bare 4xx with no body into ProblemDetails
```

`UseExceptionHandler()` (no path argument) walks the registered `IExceptionHandler` chain and falls back to writing a generic `500` ProblemDetails if none handle the exception.

## `IExceptionHandler` (.NET 8+)

```csharp
internal sealed class DomainExceptionHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (status, title) = exception switch
        {
            ValidationException        => (StatusCodes.Status400BadRequest, "Validation failed"),
            NotFoundException          => (StatusCodes.Status404NotFound,   "Resource not found"),
            ConflictException          => (StatusCodes.Status409Conflict,   "Conflict"),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Forbidden"),
            _ => (0, string.Empty),
        };

        if (status == 0) return false;                       // not ours — let the next handler try

        httpContext.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new()
        {
            HttpContext = httpContext,
            ProblemDetails = { Status = status, Title = title, Detail = exception.Message },
            Exception = exception,
        });
    }
}
```

Return `true` to short-circuit the chain. The fallback handler at the end of the chain catches everything else and writes a generic `500` — never let an exception escape with an unredacted detail.

```csharp
internal sealed class FallbackExceptionHandler(
    IProblemDetailsService problemDetails,
    ILogger<FallbackExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is OperationCanceledException && ctx.RequestAborted.IsCancellationRequested)
            return true;                                     // client disconnected — silent, no log

        logger.LogError(ex, "Unhandled exception for {Method} {Path}", ctx.Request.Method, ctx.Request.Path);

        ctx.Response.StatusCode = StatusCodes.Status500InternalServerError;
        return await problemDetails.TryWriteAsync(new()
        {
            HttpContext = ctx,
            ProblemDetails = { Status = 500, Title = "An unexpected error occurred." },
        });
    }
}
```

## `UseStatusCodePages` — the missing-body case

`UseExceptionHandler` only fires when an exception is thrown. A controller returning `NotFound()` writes the status but no body. `UseStatusCodePages()` converts those into ProblemDetails responses too:

```csharp
app.UseStatusCodePages();                    // simple
app.UseStatusCodePagesWithReExecute("/error/{0}");  // re-runs pipeline for branded HTML pages
```

`WithReExecute` is the right pick for a hybrid app that serves both JSON and Razor — it lets `/error/404` render a styled page for browsers while ProblemDetails wins for `Accept: application/json`.

## Cancellation isn't an error

```csharp
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // intentional — don't log as ERROR, don't return 500
    throw;  // let the host translate to 499 / connection close
}
```

A client that disconnects mid-request triggers `RequestAborted`. The framework throws `OperationCanceledException` deep in your handler. Logging that as ERROR turns dashboards red for normal traffic. The fallback handler above filters this case explicitly.

## Logging exceptions correctly

```csharp
// Right
logger.LogError(ex, "Failed to process order {OrderId}", orderId);

// Wrong — loses the structured property
logger.LogError($"Failed to process order {orderId}: {ex}");

// Wrong — resets the stack trace
catch (Exception ex) { logger.LogError(ex, "..."); throw ex; }

// Right
catch (Exception ex) { logger.LogError(ex, "..."); throw; }
```

Log **once**, at the boundary where you decide to handle vs propagate. Logging in three layers gives you the same exception three times in your log aggregator with three different correlation IDs.

For non-catchable rethrows across `await` boundaries, use `ExceptionDispatchInfo.Capture(ex).Throw()` to preserve the original stack.

## Custom exception → status mapping

Don't let your handlers `catch (Exception)` and translate inline — that scatters mapping across the codebase. Define a small base type and centralize it in the handler:

```csharp
public abstract class DomainException(string message) : Exception(message)
{
    public abstract int StatusCode { get; }
}

public sealed class NotFoundException(string what) : DomainException($"{what} was not found")
{
    public override int StatusCode => StatusCodes.Status404NotFound;
}
```

Then the handler is `(ex as DomainException)?.StatusCode ?? 0` instead of a long `switch`. Trade-off: if your domain is a separate assembly that shouldn't reference HTTP status codes, keep the `switch` in the handler instead.

## Pipeline order matters

```
UseExceptionHandler()         ← MUST be first (or close to it)
UseHsts()
UseHttpsRedirection()
UseStaticFiles()
UseRouting()
UseAuthentication()
UseAuthorization()
UseEndpoints(...)
```

An exception thrown in a middleware *before* `UseExceptionHandler` is not caught by it — the order is left-to-right execution, right-to-left exception bubbling. Put it first.

## Senior-level gotchas

- **`UseExceptionHandler` re-runs the pipeline on the original `HttpContext`.** If your middleware already wrote bytes to the response (e.g. compression set headers), the handler can't change the status code — `Response.HasStarted == true` and the write fails silently. Add `app.Use(async (ctx, next) => { ctx.Response.OnStarting(...); await next(ctx); })` carefully.
- **`UseStatusCodePagesWithReExecute` re-runs middleware including auth.** If your `/error/{0}` route requires auth, anonymous 401s loop. Mark error routes `[AllowAnonymous]`.
- **`IExceptionFilter` (MVC) does not run for endpoint exceptions.** It's strictly inside the action invoker. Don't use it as a global handler — use `IExceptionHandler` instead.
- **`ProblemDetails.Instance` defaults to the request path.** That leaks routing topology to clients (and shows your `/internal/admin/...` paths in error responses). Override it to `null` or a stable error code URI in `CustomizeProblemDetails`.
- **`OperationCanceledException` from `RequestAborted` is not an error.** Treat it as a 499 (or just close the connection — no body). Log it at `LogDebug` if at all. The fallback handler must filter this case before `LogError`, or your error rate dashboard tracks "users who navigated away."
- **Bare `throw ex;` resets the stack trace.** Use `throw;` to preserve, or `ExceptionDispatchInfo.Capture(ex).Throw()` when you must re-raise from a different stack frame (e.g. inside a continuation).
- **Don't serialize `Exception.ToString()` to clients.** It includes file paths, types, internal namespaces — info disclosure. ProblemDetails should carry `traceId` only; the full exception lives in the log.
- **`AddProblemDetails()` only kicks in if you also use the integrating middleware.** `[ApiController]` auto-emits ProblemDetails for `400`/`415`. `UseExceptionHandler()` and `UseStatusCodePages()` for everything else. Without them, `AddProblemDetails` does nothing visible.
- **The `Activity.Current?.Id` you put in `traceId` is W3C trace context (`00-traceid-spanid-flags`).** Tools like Application Insights, Honeycomb, and Datadog correlate on this — surface it consistently, or your traceId in the response doesn't match the traceId in the logs.
- **Throwing from `ExecuteAsync` of a `BackgroundService` doesn't go through the HTTP exception handler** — different host, different pipeline. See [background-service](./background-service.md).
- **Exception filters and `IExceptionHandler` can both fire.** A filter handles the action exception, the handler handles whatever the filter rethrows. Don't have both for the same exception type — pick one layer.
