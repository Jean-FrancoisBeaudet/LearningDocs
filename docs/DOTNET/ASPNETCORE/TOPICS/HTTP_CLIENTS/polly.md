# Polly

_Targets .NET 10 / C# 14. See also: [Microsoft Resilience](./microsoft-resilience.md), [HttpClient & HttpClientFactory](../API_COMMUNICATION/httpclient-and-httpclientfactory.md), [Refit](./refit.md), [RestSharp](./restsharp.md)._

`Polly` (NuGet: `Polly`, v8+) is the .NET resilience library — retry, circuit breaker, timeout, hedging, fallback, rate limiter, and concurrency limiter — composed into a `ResiliencePipeline`. Use it anywhere a transient failure can occur and you don't want callers to deal with it: HTTP, gRPC, message queues, EF Core saves, Azure SDK calls, your own internal services. For outbound `HttpClient` work, prefer the `Microsoft.Extensions.Http.Resilience` wrapper — it bakes Polly v8 into `IHttpClientFactory` with sane defaults. Use raw Polly directly when there's no `HttpClient` involved.

## v7 → v8 break

This is the single most important fact. The old `Policy.Handle<...>().WaitAndRetryAsync(...)` API is **legacy** and lives under separate namespaces. v8 introduces a strategy-options shape that's fundamentally different — and most of the search-engine-indexed tutorials still show v7. New code uses v8.

```csharp
// v7 (LEGACY — do not write new code like this)
var retry = Policy.Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)));

// v8 (CURRENT)
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        BackoffType      = DelayBackoffType.Exponential,
        UseJitter        = true,
        ShouldHandle     = new PredicateBuilder().Handle<HttpRequestException>(),
    })
    .Build();
```

## A complete pipeline

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay            = TimeSpan.FromMilliseconds(200),
        BackoffType      = DelayBackoffType.Exponential,
        UseJitter        = true,
        ShouldHandle     = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>(),
        OnRetry = args =>
        {
            logger.LogWarning("Retry {Attempt} after {Delay} due to {Exception}",
                args.AttemptNumber, args.RetryDelay, args.Outcome.Exception?.Message);
            return ValueTask.CompletedTask;
        },
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio       = 0.5,
        MinimumThroughput  = 20,
        SamplingDuration   = TimeSpan.FromSeconds(30),
        BreakDuration      = TimeSpan.FromSeconds(15),
    })
    .AddTimeout(TimeSpan.FromSeconds(2))   // per-attempt
    .Build();

await pipeline.ExecuteAsync(
    async ct => await DoWorkAsync(ct),
    cancellationToken);
```

**Strategy order is registration order, outer → inner.** The pipeline above does: try → on failure, retry → on too many failures, trip the breaker → each attempt is bounded to 2s. The classic mistake is putting `AddTimeout` **outside** `AddRetry` — that bounds the *entire* loop including all retries to 2s, defeating the retry. Timeout should typically be the *innermost* strategy.

## Typed pipelines (`ResiliencePipeline<T>`)

For result-based decisions (e.g. retry on `HttpResponseMessage.StatusCode == 503`):

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => (int)r.StatusCode >= 500 || r.StatusCode == HttpStatusCode.RequestTimeout),
        MaxRetryAttempts = 3,
        DelayGenerator = args => ValueTask.FromResult<TimeSpan?>(
            args.Outcome.Result?.Headers.RetryAfter?.Delta ?? TimeSpan.FromSeconds(Math.Pow(2, args.AttemptNumber))),
    })
    .Build();

var response = await pipeline.ExecuteAsync(async ct => await http.GetAsync(url, ct), cancellationToken);
```

The `DelayGenerator` lets you honor `Retry-After` from a 429/503 — Polly will wait the server-specified duration instead of your backoff curve.

## DI registration via the registry

Pipelines are **stateful** (the circuit breaker remembers failures, the rate limiter holds permits). Registering a fresh pipeline per call defeats them. Use the registry — it interns pipelines and exposes them through `ResiliencePipelineProvider<TKey>`:

```csharp
builder.Services.AddResiliencePipeline("github", b => b
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5 }));

public sealed class GitHubService(ResiliencePipelineProvider<string> provider, HttpClient http)
{
    public async Task<string> GetZenAsync(CancellationToken ct)
    {
        var pipeline = provider.GetPipeline("github");
        return await pipeline.ExecuteAsync(async token => await http.GetStringAsync("/zen", token), ct);
    }
}
```

The same pipeline instance is reused on every call — breaker state and rate limiter permits accumulate as intended.

## Hedging — tail latency mitigation

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
    {
        MaxHedgedAttempts = 2,
        Delay             = TimeSpan.FromMilliseconds(100),   // start a parallel call after 100ms
        ShouldHandle      = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => !r.IsSuccessStatusCode),
    })
    .Build();
```

If the primary call hasn't returned in 100ms, fire a second one in parallel and take whichever wins. Effective for read-heavy paths against multi-replica services; obviously wrong for non-idempotent operations.

## Telemetry

When registered via `AddResiliencePipeline(...)`, Polly v8 emits structured `ILogger` events and OpenTelemetry metrics under the meter name `Polly` automatically. Counters: `resilience.polly.strategy.events`, gauges for circuit-breaker state. Wire it into your OTel exporter and you get retry/circuit-breaker dashboards for free.

## When to use raw Polly vs Microsoft.Extensions.Http.Resilience

| Context | Use |
|---|---|
| `HttpClient` outbound | `Microsoft.Extensions.Http.Resilience` (`AddStandardResilienceHandler`) |
| gRPC over `GrpcChannel` | Raw Polly pipeline wrapping the call |
| EF Core / Dapper retries | EF Core's built-in execution strategy (`EnableRetryOnFailure`); Polly for non-EF scenarios |
| Azure SDK | Built-in pipeline; rarely need Polly on top |
| Background queue consumer | Raw Polly pipeline |
| Custom internal protocol | Raw Polly pipeline |

## Senior-level gotchas

- **Circuit breaker state is per pipeline instance.** Registering as `Transient` (or building a new pipeline per call) means a fresh breaker per call — it never trips. Always build once and share via the registry or a singleton.
- `ShouldHandle` is `ValueTask<bool>`. The `PredicateBuilder` returns a delegate already; don't wrap with `Task.FromResult` (allocates and breaks `async` context).
- `UseJitter = true` is essential for retry. Without it, every replica that failed at the same time retries at the exact same moment — a thundering herd that finishes the upstream off.
- v7 `Policy.WrapAsync(...)` ordering was the **reverse** of registration (innermost-first). v8 `ResiliencePipelineBuilder` runs in **registration order** (outermost-first). If you ported from v7 mechanically, your pipeline behavior is now inverted.
- `AddTimeout` cancels via the supplied `CancellationToken`. Your downstream code **must** honor it — a `Task.Delay` without a token, or a synchronous CPU loop, will not be interrupted.
- Inside `OnRetry` / `OnCircuitOpened` callbacks: don't do I/O. They run on the calling thread before the retry delay starts. Log and return.
- For typed `ResiliencePipeline<T>` retry: `args.Outcome.Exception` is null when the failure was a *result* (e.g. a 503 response); `args.Outcome.Result` is null when it was an exception. Always branch on which one is set.
- The rate limiter strategy uses `System.Threading.RateLimiting`. `AddConcurrencyLimiter(...)` is a *concurrency cap* (max in-flight), not a per-second rate — different concept.
- Generic `ResiliencePipeline<T>` and non-generic `ResiliencePipeline` are not interchangeable. A non-generic pipeline can execute typed lambdas but cannot react to result content — use the typed builder when you need to decide based on `T`.
- `ResiliencePipelineRegistry` resolves pipelines lazily on first `GetPipeline` call. Misconfigured options surface at first use, not at startup — call `provider.GetPipeline(...)` in a startup task if you want fail-fast behavior.
