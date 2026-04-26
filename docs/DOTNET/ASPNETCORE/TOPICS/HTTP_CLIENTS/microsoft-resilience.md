# Microsoft.Extensions.Http.Resilience

_Targets .NET 10 / C# 14. See also: [Polly](./polly.md), [HttpClient & HttpClientFactory](../API_COMMUNICATION/httpclient-and-httpclientfactory.md), [Refit](./refit.md), [RestSharp](./restsharp.md)._

`Microsoft.Extensions.Http.Resilience` (NuGet of the same name) is the **official** way to add resilience to `HttpClient`. It wraps Polly v8 with two opinionated, production-ready pipelines — `AddStandardResilienceHandler` and `AddStandardHedgingHandler` — and exposes the full Polly strategy surface for custom pipelines via `AddResilienceHandler`. It supersedes the v7-era `Microsoft.Extensions.Http.Polly` package (`AddPolicyHandler`), which is now legacy. Default to `AddStandardResilienceHandler` for every outbound `HttpClient` and only drop down to a custom pipeline when you have a specific reason.

## The standard pipeline

```csharp
builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new("https://api.github.com/"))
    .AddStandardResilienceHandler();
```

That single call adds five strategies in this fixed order:

1. **Rate limiter** — caps in-flight requests per pipeline (default `1000` permits, queue `0`).
2. **Total request timeout** — ceiling for the entire operation including retries (default `30s`).
3. **Retry** — exponential backoff with jitter (default 3 attempts, base delay `2s`).
4. **Circuit breaker** — opens when failure ratio crosses 10% across a 30s window with ≥100 throughput (default).
5. **Per-attempt timeout** — bounds each individual try (default `10s`).

The defaults are tuned for "well-behaved internal microservice." For public APIs or flaky upstreams, override them:

```csharp
builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new("https://api.github.com/"))
    .AddStandardResilienceHandler(o =>
    {
        o.Retry.MaxRetryAttempts             = 3;
        o.Retry.Delay                        = TimeSpan.FromMilliseconds(500);
        o.Retry.BackoffType                  = DelayBackoffType.Exponential;
        o.Retry.UseJitter                    = true;
        o.AttemptTimeout.Timeout             = TimeSpan.FromSeconds(2);
        o.TotalRequestTimeout.Timeout        = TimeSpan.FromSeconds(15);   // must be ≥ AttemptTimeout * (MaxRetryAttempts + 1) + backoff
        o.CircuitBreaker.FailureRatio        = 0.5;
        o.CircuitBreaker.MinimumThroughput   = 50;
        o.CircuitBreaker.SamplingDuration    = TimeSpan.FromSeconds(30);
        o.CircuitBreaker.BreakDuration       = TimeSpan.FromSeconds(15);
        o.RateLimiter.DefaultRateLimiterOptions.PermitLimit = 200;
    });
```

These options are **validated at startup**. If `TotalRequestTimeout` < `AttemptTimeout`, the host fails to start with a clear `OptionsValidationException` — no silent misconfiguration.

## Default `ShouldHandle`

The standard pipeline already classifies the right things as transient:

- `HttpRequestException` (DNS, socket, TLS)
- `IOException` (mid-stream interruption)
- `TimeoutRejectedException` (the per-attempt timeout firing)
- HTTP `408`, `429`, `5xx`

To extend (e.g. retry on a 401 once after refreshing a token — usually a bad idea, but as an example):

```csharp
.AddStandardResilienceHandler(o =>
{
    o.Retry.ShouldHandle = args =>
    {
        if (args.Outcome.Result is { StatusCode: HttpStatusCode.Unauthorized })
            return ValueTask.FromResult(true);
        return HttpClientResiliencePredicates.IsTransient(args.Outcome);   // keep defaults
    };
});
```

`HttpClientResiliencePredicates.IsTransient` is the public predicate the standard pipeline uses internally — call it to preserve default behavior while extending it.

## The hedging pipeline

For latency-sensitive read paths against a multi-replica service:

```csharp
builder.Services.AddHttpClient<SearchClient>(c => c.BaseAddress = new("https://search.internal/"))
    .AddStandardHedgingHandler(o =>
    {
        o.Hedging.MaxHedgedAttempts = 2;
        o.Hedging.Delay             = TimeSpan.FromMilliseconds(100);
    });
```

If the primary call hasn't returned in 100ms, fire a parallel attempt. Whichever finishes first wins, the other is cancelled. Use only for **idempotent** reads — hedging a non-idempotent `POST` will create duplicates.

## Custom pipelines

Drop `Standard` for full control:

```csharp
builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new("https://api.github.com/"))
    .AddResilienceHandler("github", builder =>
    {
        builder
            .AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 5,
                BackoffType      = DelayBackoffType.Exponential,
                UseJitter        = true,
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                FailureRatio       = 0.3,
                MinimumThroughput  = 20,
            })
            .AddTimeout(TimeSpan.FromSeconds(2));
    });
```

The `Http*StrategyOptions` types come pre-configured with `HttpClient`-aware `ShouldHandle` predicates — easier than writing them from raw `Polly.Core` types.

## Per-call overrides

To tune resilience for a specific call (e.g. a known-flaky endpoint), attach a `ResilienceContext`:

```csharp
public async Task<Quote> GetQuoteAsync(string symbol, CancellationToken ct)
{
    var context = ResilienceContextPool.Shared.Get(ct);
    context.Properties.Set(ResilienceKeys.MaxRetries, 5);   // a custom key your pipeline reads
    try
    {
        var req = new HttpRequestMessage(HttpMethod.Get, $"/quotes/{symbol}");
        req.SetResilienceContext(context);
        var resp = await http.SendAsync(req, ct);
        return (await resp.Content.ReadFromJsonAsync<Quote>(ct))!;
    }
    finally { ResilienceContextPool.Shared.Return(context); }
}
```

The `SetResilienceContext` extension flows the context to whichever resilience handler runs in the chain — your `OnRetry` callbacks and `ShouldHandle` predicates can read it via `args.Context.Properties`.

## Telemetry

The resilience handler emits:

- `ILogger` events under category `Polly` with structured properties (strategy name, attempt number, outcome).
- OpenTelemetry meter `Polly` with `resilience.polly.strategy.events` (counter) and circuit-state gauges.
- Activity tags for distributed tracing — retries appear as child spans of the originating request.

Enable via the standard OTel registration; no extra config:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m.AddMeter("Polly").AddPrometheusExporter())
    .WithTracing(t => t.AddSource("Polly"));
```

## Standard vs custom — when to choose

| Need | Use |
|---|---|
| Sane defaults, one-line wiring | `AddStandardResilienceHandler` |
| Tail latency on idempotent reads | `AddStandardHedgingHandler` |
| Different strategy *order* (e.g. retry inside breaker) | `AddResilienceHandler` |
| Per-endpoint tuning within one client | `AddResilienceHandler` keyed by `HttpRequestMessage` properties |
| No `HttpClient` involved (queue, gRPC) | Drop down to raw [Polly](./polly.md) |

## Senior-level gotchas

- `AddStandardResilienceHandler` and the legacy `AddPolicyHandler` (Polly v7) are **mutually incompatible** — never mix them on the same client. Migrate fully.
- `TotalRequestTimeout` includes retry waits. Set it generously: `AttemptTimeout × (MaxRetryAttempts + 1) + sum-of-backoffs + jitter`. Otherwise the first retry storm trips the total timeout instead of the per-attempt one and your retries never complete.
- The rate limiter is **per pipeline instance**, which means **per pooled handler**. `IHttpClientFactory` rotates pooled handlers every `HandlerLifetime` (default 2 min) — newly-created handlers get fresh limiter state. For strict global rate limiting, you need an external coordinator (Redis, sidecar).
- Circuit breaker state is shared across all calls through that named client *in this process*, but **not across replicas**. For fleet-wide breaker coordination you need a service mesh, a feature flag, or a shared signal.
- The standard handler's strategy order is fixed (rate limiter → total timeout → retry → breaker → attempt timeout). If you need a different order, you must use `AddResilienceHandler` and assemble manually — `o.Retry`, `o.CircuitBreaker` only let you tune the *parameters*, not the position.
- `o.Retry.OnRetry` (and other callbacks) run **synchronously on the request thread** before the retry delay starts. Don't perform I/O or block — log and return `ValueTask.CompletedTask`.
- Hedging and non-idempotent verbs is a footgun. The default `ShouldHedge` predicate hedges everything. For mixed clients, override it to skip POST/PUT/PATCH/DELETE explicitly.
- `AddStandardResilienceHandler().Configure(...)` (the older overload) and the inline lambda `AddStandardResilienceHandler(o => ...)` use different validation passes. Prefer the inline form — validation runs at `Build()` and surfaces misconfigurations at startup.
- The handler runs **outside** any `DelegatingHandler` you add via `AddHttpMessageHandler`. So an auth handler that refreshes a token sees only the *original* request, not the retried one. If you need to rebuild the bearer token per attempt, do it inside the auth handler unconditionally (it's already invoked once per attempt).
- For Refit clients, the resilience handler wraps the entire Refit-generated call. Refit doesn't see retries — it just sees a successful or failed `HttpResponseMessage` after the pipeline finishes. That's the desired behavior; just be aware that `OnRetry` callbacks fire below the Refit interface.
