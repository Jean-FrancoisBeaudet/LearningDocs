# Rate Limiting

_Targets .NET 10 / C# 14. See also: [Middlewares](./middlewares.md), [Authentication and Authorization](./authentication-and-authorization.md), [Problem Details](./problem-details.md)._

ASP.NET Core ships built-in rate limiting (`Microsoft.AspNetCore.RateLimiting`, GA-quality from .NET 8) — four algorithms, partitioned across keys, plumbed via endpoint metadata. **In-process by default**; cross-replica state is your problem.

## The four algorithms

| Algorithm | Use it for |
|---|---|
| **Fixed window** | Simple per-window quotas (e.g. 100 req / minute). Bursty at window edges. |
| **Sliding window** | Smoother quota by splitting the window into segments. More memory, fairer rate. |
| **Token bucket** | Burst-tolerant rate limiting (refill rate + bucket capacity). Best for "average X but allow brief spikes". |
| **Concurrency** | Caps *in-flight* requests, not requests per second. Use to protect a slow downstream (e.g. only 10 concurrent calls allowed). |

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(opts =>
{
    opts.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    opts.OnRejected = async (ctx, ct) =>
    {
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retry))
            ctx.HttpContext.Response.Headers.RetryAfter = ((int)retry.TotalSeconds).ToString();

        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            type   = "https://example.com/probs/rate-limited",
            title  = "Too Many Requests",
            status = 429,
        }, cancellationToken: ct);
    };

    opts.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window      = TimeSpan.FromMinutes(1),
                QueueLimit  = 0,
            }));

    opts.AddPolicy("write", ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            ctx.User.Identity?.Name ?? "anon",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit           = 20,
                TokensPerPeriod      = 10,
                ReplenishmentPeriod  = TimeSpan.FromSeconds(1),
                AutoReplenishment    = true,
                QueueLimit           = 5,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            }));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();          // after auth so we can partition by user

app.MapPost("/orders", CreateOrder).RequireRateLimiting("write");
app.MapGet("/orders",  ListOrders);    // hits GlobalLimiter only
```

`UseRateLimiter` ordering matters: place it **after** authentication if you partition by `User.Identity.Name` (otherwise everyone is "anon"), but **before** the endpoint. If anonymous abuse is a real concern *and* token validation is expensive, run a cheaper IP-based limiter ahead of auth and a per-user limiter after.

## Partitioning

Every limiter is partitioned by a key derived from the request:

```csharp
opts.AddPolicy("per-tenant", ctx =>
{
    var tenant = ctx.Request.Headers["X-Tenant"].ToString();
    return string.IsNullOrEmpty(tenant)
        ? RateLimitPartition.GetNoLimiter("anon")     // unbounded for missing tenant
        : RateLimitPartition.GetSlidingWindowLimiter(
            tenant,
            _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit       = 10_000,
                Window            = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6,
                QueueLimit        = 0,
            });
});
```

Every distinct partition key allocates its own limiter. **Cardinality matters** — see gotchas.

## Applying

```csharp
// Endpoint
app.MapDelete("/admin/users/{id:int}", DeleteUser).RequireRateLimiting("admin-mutations");

// Group
var api = app.MapGroup("/api").RequireRateLimiting("api-default");

// Controllers
[EnableRateLimiting("write")]
public class OrdersController : ControllerBase { /* ... */ }

[DisableRateLimiting]   // exempt one action
public IActionResult Health() => Ok();
```

## Concurrency limiter

Different shape — caps simultaneous in-flight requests, not rate:

```csharp
opts.AddPolicy("downstream-bff", ctx =>
    RateLimitPartition.GetConcurrencyLimiter("bff",
        _ => new ConcurrencyLimiterOptions
        {
            PermitLimit          = 50,        // only 50 active at a time
            QueueLimit           = 200,       // queue up to 200 more
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        }));
```

`PermitLimit` here is *active permits*, **not** requests per second. A request that takes 10 s holds its permit for 10 s — slow downstream = saturation.

## Distributed rate limiting (what's NOT in the box)

Built-in limiters are per-process. Behind a load balancer with N replicas, `PermitLimit = 100, Window = 1 minute` allows N × 100 cluster-wide. Options:

- **Sticky sessions** — partitions stick to one replica. Simplest if your LB supports it; falls apart on replica scale-out.
- **Push to the gateway** — APIM, Azure Front Door, NGINX, Envoy, Cloudflare. Usually the right answer; the gateway has cluster-wide state and can offload the work.
- **Redis-backed `PartitionedRateLimiter`** — community packages or roll your own with Redis `INCR` + `EXPIRE`. Adds a network hop per request; viable for low-latency Redis.
- **Accept N× drift** and provision limits accordingly — fine for fairness, not fine for hard-cap quotas.

## Senior-level gotchas

- Counters are **per-process**. A 3-replica deployment with `PermitLimit = 100, Window = 1 minute` allows 300 req/min cluster-wide. Decide early whether to push limits to a gateway or live with the drift.
- `OnRejected` fires *before* the response is written. Async work there sits in the rejection path — keep it tight (one log line + headers + small JSON body). Don't `await` a database call in `OnRejected`.
- `PermitLimit` semantics differ across limiter types:
  - Fixed/sliding window/token bucket: requests *per window* / *bucket size*.
  - Concurrency limiter: requests in-flight at any instant.
  Mixing the mental models is the #1 source of "why doesn't my limit work" bugs.
- Partition keys are stored in a `ConcurrentDictionary` keyed by string. Unbounded keys (per-request UUID, full path with query string, raw user-agent) cause unbounded memory growth. Always cap cardinality — bucket IPs to /24, hash long keys, normalize paths.
- `QueueProcessingOrder.OldestFirst` (FIFO) and `NewestFirst` (LIFO) materially change p99 under sustained overload. FIFO is fairer; LIFO drops old requests faster (better when clients retry on timeout). Pick based on whether your clients tolerate variable latency or aggressive timeouts.
- `RequireRateLimiting("policy")` overrides the global limiter — it does **not** compose. If you want hierarchical "global + per-endpoint" limiting, write a custom `PartitionedRateLimiter` that composes both, or chain two middleware instances.
- `aspnetcore.rate_limiting.request_lease.duration` and `aspnetcore.rate_limiting.queued_requests` are exported via `System.Diagnostics.Metrics` — wire them to OpenTelemetry. Without metrics, you're flying blind on whether limits are biting or just queueing.
- Apply rate limits *before* authentication only when the limit is by IP/connection (cheap signal). Apply *after* auth when partitioning by user. The right answer for many APIs is two layers — cheap per-IP outer, per-user inner. Don't combine them into one mega-policy; readability collapses.
- `AutoReplenishment = true` on the token bucket runs a background timer on the thread pool. Heavy thread-pool starvation can make replenishment lag and effectively reduce your limit. If you have aggressive workloads, set `AutoReplenishment = false` and call `TryReplenish()` from a dedicated timer.
- The chained limiter API (`PartitionedRateLimiter.CreateChained(...)`) lets you require *all* of multiple partitioned limiters to grant a permit before the request proceeds. Useful for "global + per-tenant" without writing custom code, but every layer is a potential queue point.
