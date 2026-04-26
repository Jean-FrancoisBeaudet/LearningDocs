# OutputCache

_Targets .NET 10 / C# 14. See also: [StackExchange.Redis](./stackexchange-redis.md), [HybridCache](./hybrid-cache.md)._

`OutputCache` (ASP.NET Core 7+) is server-side caching of **whole HTTP responses** — status code, headers, and body. The middleware sits in front of the endpoint pipeline; on a hit, the request short-circuits and the cached response is replayed without the endpoint ever running. It replaces the older `ResponseCaching` middleware for any case where you actually want the server to do the caching.

## OutputCache vs ResponseCaching — they solve different problems

| | `OutputCache` | `ResponseCaching` |
|---|---|---|
| **What it caches** | Server stores the response | Honors `Cache-Control` from upstream/your endpoint, sets headers for downstream caches |
| **Layer** | Server-side (in this app) | HTTP-cache semantics (browsers, CDNs, proxies) |
| **Control** | Programmatic policies | Headers (`Cache-Control: public, max-age=...`) |
| **Per-user variation** | Custom `VaryByValue` | Tied to `Vary` header |
| **Tag invalidation** | Yes (`EvictByTagAsync`) | No |
| **Stampede protection** | Yes (per-key locking, opt-in) | No |

If you want a CDN to cache it, set `Cache-Control` (use `ResponseCaching`). If you want **this app** to skip work on repeat requests, use `OutputCache`.

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromMinutes(1)));   // default for all
    options.AddPolicy("Hourly", b => b.Expire(TimeSpan.FromHours(1)));
    options.AddPolicy("PerCountry", b => b
        .Expire(TimeSpan.FromMinutes(10))
        .SetVaryByHeader("CF-IPCountry")
        .Tag("country-data"));
});

var app = builder.Build();

app.UseOutputCache();   // before endpoints

app.MapGet("/products",        GetProducts).CacheOutput("Hourly");
app.MapGet("/news/{country}",  GetNews).CacheOutput("PerCountry");
app.MapPost("/admin/reload",   async (IOutputCacheStore store, CancellationToken ct) =>
{
    await store.EvictByTagAsync("country-data", ct);
    return Results.NoContent();
});

app.Run();
```

On controllers, use `[OutputCache(PolicyName = "Hourly")]` or `[OutputCache(Duration = 60, VaryByQueryKeys = new[] { "page", "sort" })]`.

## Vary dimensions

`OutputCache` keys are derived from the request path + the policy's vary configuration:

```csharp
options.AddPolicy("PerUser", b => b
    .Expire(TimeSpan.FromMinutes(5))
    .SetVaryByQuery("page", "size")
    .SetVaryByHeader("Accept-Language")
    .VaryByValue(httpContext =>
    {
        var userId = httpContext.User.FindFirst("sub")?.Value ?? "anon";
        return new KeyValuePair<string, string>("uid", userId);
    }));
```

`VaryByValue` is the escape hatch — anything you can compute from `HttpContext` becomes part of the key. Use sparingly; **every distinct value is a separate cache entry**.

## Tag-based invalidation

Cached entries can be tagged at insert time and evicted in bulk:

```csharp
options.AddPolicy("ByCategory", b => b
    .Expire(TimeSpan.FromHours(2))
    .Tag("products")
    .Tag(ctx => $"category:{ctx.Request.RouteValues["categoryId"]}"));

// On product update:
await outputCacheStore.EvictByTagAsync($"category:{categoryId}", ct);
```

This is the right pattern for cache invalidation: tag at write time, evict by tag on the mutation. Don't try to compute keys retroactively.

## Stores

Default store is in-memory (`MemoryOutputCacheStore`). For multi-replica apps you need a shared store — `Microsoft.AspNetCore.OutputCaching.StackExchangeRedis` provides one:

```csharp
builder.Services.AddStackExchangeRedisOutputCache(opt =>
{
    opt.Configuration = builder.Configuration.GetConnectionString("Redis");
    opt.InstanceName  = "oc:";
});
```

With a distributed store, `EvictByTagAsync` correctly invalidates across all replicas.

## When to use

- Idempotent GETs whose body is **stable for a known TTL** (product catalog, daily report, public news).
- High-fanout endpoints where the same response goes to many clients.
- Endpoints with cheap-to-compute keys but expensive bodies (DB roll-ups, multi-service aggregations).
- Anywhere you used to write `[ResponseCache(Duration=...)]` and were actually trying to cache server-side.

## When NOT to use

- Per-user data unless you've thought through the key explosion (`VaryByValue` on user id → one entry per user, possibly fine, possibly not).
- Anything that mutates state (POST/PUT/DELETE).
- Responses with `Set-Cookie`, fresh CSRF tokens, or anti-forgery tokens — replaying these is a security bug.
- Endpoints with `Authorization` header — by default `OutputCache` **skips** these requests entirely (you must opt in explicitly via policy).
- Streaming or chunked responses where you'd be capturing megabytes of body per cache entry.

**Senior-level gotchas:**
- Requests with `Authorization` are bypassed by default. To cache an authenticated endpoint, the policy must call `b.AllowLocking()` *and* explicitly opt in (`b.SetCacheManager(...)` or write a custom `IOutputCachePolicy` that doesn't short-circuit on `Authorization`). Forgetting this means "the cache mysteriously never hits in prod" because every request carries a bearer token.
- Locking is **on per default for the base policy** — concurrent misses for the same key wait for the first to populate. If you disable locking (`b.AllowLocking(false)`), N concurrent misses execute the endpoint N times. Good for non-idempotent diagnostics, bad for everything else.
- `VaryByValue` keys grow unbounded. Per-user variation × any other vary axis × URL = combinatorial explosion. The in-memory store has size caps (`SizeLimit`); blow past them and entries get evicted on every request — cache hit rate goes to zero and your CPU climbs.
- `EvictByTagAsync` is **best-effort and async** — it returns when the tag→key index has been purged, but readers may briefly observe stale entries depending on store semantics. Don't treat it as strongly consistent for "the user just clicked save and must see fresh data."
- In-memory store is per-replica. Behind a load balancer with N pods, an `EvictByTagAsync` only clears one pod's cache. Use the distributed store, or accept that "fresh after invalidate" requires sticky routing.
- The middleware order matters: `UseOutputCache()` must be **after** `UseRouting()` and **before** the endpoint execution. If you add it after auth, `Authorization`-header requests still skip; if before routing, policies that depend on route values won't have them.
- `OutputCache` captures the body before `Response.Body` is replaced for compression. If you want compressed cached bodies, `UseResponseCompression()` must come **before** `UseOutputCache()` in the pipeline — otherwise you cache uncompressed and re-compress per hit.
- A cached response includes the `Set-Cookie` header from the original. If your endpoint sets cookies (session, anti-forgery), you'll **leak the first user's cookie** to every subsequent hit. The middleware does not strip these — you must.
- Policy ordering: endpoint-level `CacheOutput("Foo")` overrides the base policy entirely, it does not compose. If you want layered behavior, build a policy that explicitly chains via `IOutputCachePolicy.CacheRequestAsync`.
- `ResponseCaching` and `OutputCache` can coexist but it's almost always a configuration smell. Pick one mental model per endpoint.
