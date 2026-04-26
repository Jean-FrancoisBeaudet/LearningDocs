# HttpClient & HttpClientFactory

_Targets .NET 10 / C# 14. See also: [REST](./rest.md), [gRPC](./grpc.md), [Refit](../HTTP_CLIENTS/refit.md), [RestSharp](../HTTP_CLIENTS/restsharp.md), [Polly](../HTTP_CLIENTS/polly.md), [Microsoft Resilience](../HTTP_CLIENTS/microsoft-resilience.md)._

`HttpClient` is the primary outbound HTTP type in .NET. It looks innocent — until you find out the hard way that wrapping it in `using` exhausts your sockets and that holding a singleton wrecks DNS rotation. **`IHttpClientFactory`** (`Microsoft.Extensions.Http`) fixes both: pooled, periodically-rotated `SocketsHttpHandler` instances behind a thin transient `HttpClient`. In any ASP.NET Core or hosted service, you should never `new HttpClient()` — always go through the factory.

## The two original problems

```csharp
// Problem 1: socket exhaustion
for (int i = 0; i < 1000; i++)
{
    using var client = new HttpClient();           // new connection every loop
    await client.GetAsync(url);
}                                                   // sockets sit in TIME_WAIT for ~240s
// Eventually: SocketException "Only one usage of each socket address..."

// Problem 2: stale DNS
static readonly HttpClient s_client = new();        // singleton — never sees DNS changes
// Pin to one IP, deployment moves the service, requests fail until the process restarts.
```

`IHttpClientFactory` solves both with a single mechanism: **pool the underlying `HttpMessageHandler` and rotate it on a timer.** New `HttpClient` instances reuse pooled handlers, so connections are kept alive; pooled handlers are recycled every `HandlerLifetime` (default 2 min), so DNS gets refreshed without restarting the process.

## Three registration styles

```csharp
// 1. Basic / named
builder.Services.AddHttpClient("github", c =>
{
    c.BaseAddress = new("https://api.github.com/");      // trailing slash matters
    c.DefaultRequestHeaders.UserAgent.ParseAdd("acme/1.0");
});

// consumer
public sealed class GitHubProbe(IHttpClientFactory factory)
{
    public async Task<HttpResponseMessage> PingAsync(CancellationToken ct)
        => await factory.CreateClient("github").GetAsync("zen", ct);
}
```

```csharp
// 2. Typed (preferred — strong-typed wrapper, DI registers the consumer too)
builder.Services.AddHttpClient<GitHubClient>(c =>
{
    c.BaseAddress = new("https://api.github.com/");
    c.DefaultRequestHeaders.UserAgent.ParseAdd("acme/1.0");
});

public sealed class GitHubClient(HttpClient http)
{
    public async Task<string> GetZenAsync(CancellationToken ct)
        => await http.GetStringAsync("zen", ct);
}
```

```csharp
// 3. Refit (declarative — see HTTP_CLIENTS/refit.md)
builder.Services.AddRefitClient<IGitHubApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new("https://api.github.com/"));
```

The typed style binds the lifetime: `GitHubClient` is registered **transient**, but its `HttpClient` is supplied per-call from the factory. The handler stays pooled.

## DelegatingHandlers

Cross-cutting concerns belong in a handler chain, not in every consumer:

```csharp
public sealed class AuthHeaderHandler(ITokenSource ts) : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct)
    {
        req.Headers.Authorization = new("Bearer", await ts.GetAsync(ct));
        return await base.SendAsync(req, ct);
    }
}

builder.Services.AddTransient<AuthHeaderHandler>();
builder.Services.AddHttpClient<GitHubClient>(c => c.BaseAddress = new("https://api.github.com/"))
    .AddHttpMessageHandler<AuthHeaderHandler>()       // outermost first
    .AddHttpMessageHandler<LoggingHandler>()
    .AddHttpMessageHandler<CorrelationIdHandler>();   // innermost — closest to the wire
```

Handler order is **outer → inner**: the first registered runs first on the way out and last on the way in. Authorization (which mutates the request) goes outermost; logging that wants the final headers goes innermost.

Each `DelegatingHandler` must be registered as transient (or scoped) — the factory wires a fresh instance per pooled handler chain. Singletons here re-use pooled state and break correctness.

## Resilience

Don't roll your own retry. Use **`Microsoft.Extensions.Http.Resilience`** (the modern wrapper around Polly v8):

```csharp
builder.Services.AddHttpClient<GitHubClient>(...)
    .AddStandardResilienceHandler(o =>
    {
        o.Retry.MaxRetryAttempts = 3;
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
        o.CircuitBreaker.FailureRatio = 0.5;
    });
```

The standard pipeline gives you **rate limiter → total timeout → retry → circuit breaker → attempt timeout** in the right order. Custom strategies via `AddResilienceHandler("name", b => b.AddRetry(...).AddCircuitBreaker(...))`. See [microsoft-resilience](../HTTP_CLIENTS/microsoft-resilience.md) and [polly](../HTTP_CLIENTS/polly.md).

## Reading responses without buffering

The default behavior buffers the whole response into memory before returning. For large payloads or streaming JSON:

```csharp
using var response = await http.GetAsync("export", HttpCompletionOption.ResponseHeadersRead, ct);
response.EnsureSuccessStatusCode();
await using var stream = await response.Content.ReadAsStreamAsync(ct);

// streaming JSON — does not buffer the whole array
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Order>(stream, cancellationToken: ct))
    handle(item);
```

`ReadFromJsonAsync<T>` and `PostAsJsonAsync` are the idiomatic JSON helpers — they pick up `JsonSerializerOptions` from `ConfigureHttpJsonOptions` and play well with `[JsonSerializable]` source generators.

## Handler tuning

You'll occasionally need to set `SocketsHttpHandler` flags directly:

```csharp
builder.Services.AddHttpClient<GitHubClient>(...)
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime       = TimeSpan.FromMinutes(2),
        PooledConnectionIdleTimeout    = TimeSpan.FromMinutes(1),
        MaxConnectionsPerServer        = 100,
        EnableMultipleHttp2Connections = true,
        AutomaticDecompression         = DecompressionMethods.All,
    })
    .SetHandlerLifetime(TimeSpan.FromMinutes(5));     // must be ≥ PooledConnectionLifetime
```

`MaxConnectionsPerServer` defaults to `int.MaxValue` and the framework happily opens hundreds of sockets to a single host under load — clamp it for any high-fanout service. `EnableMultipleHttp2Connections` matters when one HTTP/2 stream cap (~100 concurrent) becomes a bottleneck.

## Modern note (.NET 10)

- `HttpClient` supports HTTP/3 via QUIC where the OS provides msquic — set `RequestVersion = HttpVersion.Version30` and `VersionPolicy = HttpVersionPolicy.RequestVersionOrLower`.
- `HttpClient.GetFromJsonAsAsyncEnumerable<T>` streams a JSON array as `IAsyncEnumerable<T>` without intermediate buffering.
- The `WebApplicationBuilder` automatically registers `IHttpClientFactory` services if you call `AddHttpClient(...)`; no separate `AddHttpClient()` call needed (older docs sometimes show one).

## Senior-level gotchas

- `using var client = factory.CreateClient(...)` is **fine** — disposing the wrapper is a no-op for the pooled handler. Mid-level devs see `IDisposable`, panic, and either over-dispose (no harm) or skip the factory entirely (harm).
- `BaseAddress` requires a **trailing slash** if you want `new Uri(base, "users")` to resolve as `/users` under the base path. Without the slash, `Uri` strips the last segment of `BaseAddress`. Hours have been lost.
- `DefaultRequestHeaders` is **not** thread-safe and is shared by every client minted from the same registration. Set it at registration time only; per-request headers go on `HttpRequestMessage`.
- `HttpClient.Timeout` cancels the **entire** request including streaming — including consumption of the response body. For streaming downloads, set `Timeout = Timeout.InfiniteTimeSpan` and rely on `CancellationToken` + `AttemptTimeout`.
- Logging produced by `IHttpClientFactory` (category `System.Net.Http.HttpClient.<name>.LogicalHandler` and `.ClientHandler`) is verbose and includes URLs/headers. PII can leak into logs at `Information`. Tighten that category to `Warning` in production unless you're actively debugging.
- A pooled handler that's been recycled keeps living until its existing requests drain. `SetHandlerLifetime(TimeSpan.Zero)` defeats pooling entirely (one handler per call) — use only for tests; absolutely not in prod.
- `HttpClient` instances created from the factory **don't** share `CookieContainer` automatically. If you need cookies, `ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler { CookieContainer = container, UseCookies = true })`.
- Kestrel's HTTP/2 server doesn't support gRPC over plain HTTP without explicit opt-in; `HttpClient`'s default version negotiation will downgrade silently. Set `RequestVersion = HttpVersion.Version20, VersionPolicy = HttpVersionPolicy.RequestVersionExact` when the protocol matters.
- `PostAsJsonAsync` writes the body using `application/json; charset=utf-8` content type. Some legacy servers reject the `charset` parameter — set the `Content-Type` manually if you hit a 415.
