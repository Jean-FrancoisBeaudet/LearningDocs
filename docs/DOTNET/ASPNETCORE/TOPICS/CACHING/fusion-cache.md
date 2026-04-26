# FusionCache

_Targets .NET 10 / C# 14. See also: [HybridCache](./hybrid-cache.md), [StackExchange.Redis](./stackexchange-redis.md), [OutputCache](./output-cache.md)._

`FusionCache` (NuGet: `ZiggyCreatures.Caching.Fusion`) is a third-party hybrid cache that predates Microsoft's [HybridCache](./hybrid-cache.md) and remains the more feature-rich option. Same two-tier (L1 + L2) shape, but adds the things production caching layers actually need: **fail-safe** (return stale on factory error), **soft/hard timeouts**, **eager refresh** (background revalidate near expiry), **adaptive caching** (factory adjusts TTL per result), a **backplane** (Redis pub/sub) to invalidate L1 across replicas, and **auto-recovery** for failed write-throughs.

## What FusionCache adds over HybridCache

| Capability | What it does | Why it matters |
|---|---|---|
| **Fail-safe** | If the factory throws, return the last-known-good value past its expiry | Upstream outage doesn't take you down |
| **Soft timeout** | Cap factory time; on timeout, return stale (if available) and let the factory finish in background | Tail latency stops at the cap, not at the slow dependency |
| **Hard timeout** | Cap factory time absolutely; cancel and surface error | Bound worst-case latency for SLA-bound endpoints |
| **Eager refresh** | When entry is past X% of its TTL, refresh in background on next read | No "everyone waits for the regen" cliff at expiry |
| **Adaptive caching** | Factory mutates `FusionCacheFactoryExecutionContext` to set per-call TTL/options | Long TTL for "popular" results, short for "rare" — based on the value itself |
| **Backplane** | Redis pub/sub channel notifies replicas to evict L1 on writes | L1 stays coherent across pods without sticky routing |
| **Auto-recovery** | Failed L2 writes re-attempted in background once L2 is healthy | Brief Redis blip doesn't permanently desync L1/L2 |

If you don't need any of these, [HybridCache](./hybrid-cache.md) is the simpler choice. If you need any one of them, FusionCache is usually less work than building it yourself.

## Wiring

```csharp
builder.Services
    .AddFusionCache()
    .WithDefaultEntryOptions(o =>
    {
        o.Duration                 = TimeSpan.FromMinutes(10);
        o.IsFailSafeEnabled        = true;
        o.FailSafeMaxDuration      = TimeSpan.FromHours(1);   // how stale we'll go
        o.FailSafeThrottleDuration = TimeSpan.FromSeconds(30); // don't retry factory more often than this on failure
        o.FactorySoftTimeout       = TimeSpan.FromMilliseconds(200);
        o.FactoryHardTimeout       = TimeSpan.FromSeconds(2);
        o.EagerRefreshThreshold    = 0.8f;   // refresh in background after 80% of Duration
    })
    .WithSerializer(new FusionCacheSystemTextJsonSerializer())
    .WithDistributedCache(new RedisCache(new RedisCacheOptions
    {
        Configuration = builder.Configuration.GetConnectionString("Redis")
    }))
    .WithBackplane(new RedisBackplane(new RedisBackplaneOptions
    {
        Configuration = builder.Configuration.GetConnectionString("Redis")
    }));
```

L1 only? Drop `WithDistributedCache` and `WithBackplane`. L1 + L2 only? Drop `WithBackplane` (you lose cross-replica L1 coherence). The pieces compose.

## Usage

```csharp
public sealed class WeatherService(IFusionCache cache, HttpClient http)
{
    public ValueTask<Forecast> GetAsync(string city, CancellationToken ct) =>
        cache.GetOrSetAsync<Forecast>(
            $"weather:{city}",
            async (ctx, token) =>
            {
                var fresh = await http.GetFromJsonAsync<Forecast>($"/forecast/{city}", token)
                            ?? throw new InvalidOperationException("null forecast");

                // Adaptive: cache popular cities longer
                if (fresh.IsHighTraffic)
                    ctx.Options.Duration = TimeSpan.FromHours(1);

                return fresh;
            },
            options: opts => opts.SetDuration(TimeSpan.FromMinutes(10)),
            tags: ["weather", $"city:{city}"],
            token: ct);
}
```

The `ctx` parameter is `FusionCacheFactoryExecutionContext<T>` — it exposes the previous (stale) value, lets you mutate per-call options (adaptive caching), and signals fail-safe activation.

## Fail-safe in action

```csharp
// Endpoint depends on a flaky upstream
app.MapGet("/quote/{symbol}", async (string symbol, IFusionCache cache, IQuotes upstream, CancellationToken ct) =>
{
    var quote = await cache.GetOrSetAsync(
        $"quote:{symbol}",
        async (ctx, t) => await upstream.GetQuoteAsync(symbol, t),  // may throw
        options: o => o
            .SetDuration(TimeSpan.FromSeconds(5))
            .SetFailSafe(true, maxDuration: TimeSpan.FromMinutes(10)),
        token: ct);

    return Results.Ok(quote);
});
```

If `upstream.GetQuoteAsync` throws and a previous value exists in cache (even past its 5-second TTL), FusionCache returns that stale value. The next call after `FailSafeThrottleDuration` retries the factory. Result: an upstream outage degrades to "slightly stale data" instead of "503 to every user."

**Always log fail-safe activations** (`OnFailSafeActivate` event) — silent stale-serving is how you mask outages from yourself.

## Backplane semantics

Important: the backplane is **invalidation, not replication**. When pod A writes/evicts a key, it publishes a notification on the Redis channel. Pods B, C, D evict their L1 entry for that key. They do **not** receive the new value — they re-read from L2 (or re-run the factory) on next access.

This means:
1. L1 across replicas stays coherent (no stale reads after a write — well, eventually consistent).
2. A backplane invalidation can trigger a stampede on L2 / source if N replicas all miss simultaneously. The `FactorySoftTimeout` + single-flight combination contains this — first replica regenerates, others return stale (fail-safe) until L2 is repopulated.

## When to use FusionCache (over HybridCache)

- Upstream dependencies that can flake (third-party APIs, slow databases) — fail-safe + soft timeout are killer features.
- Multi-replica deployments where L1 incoherence is visible to users.
- High-traffic endpoints where the "expiry cliff" causes regeneration storms — eager refresh smooths this.
- Per-result TTL logic (cache popular things longer) — adaptive caching.

## When to use HybridCache instead

- You need first-party Microsoft support and your org dislikes third-party deps.
- Simple two-tier with single-flight is enough.
- You're on .NET 9+ greenfield and don't (yet) need fail-safe / eager refresh.

## When to use neither

- Single-process, no distribution, no fail-safe needs → `IMemoryCache` directly.
- Caching whole HTTP responses → [OutputCache](./output-cache.md).
- Just need byte-level distributed cache → [`IDistributedCache`](./stackexchange-redis.md) directly.

**Senior-level gotchas:**
- **Fail-safe silently masks upstream outages.** A monitoring dashboard showing "200s, low latency" while customers see week-old data is the failure mode. Subscribe to `Events.FailSafeActivate` and emit a metric — alert when it spikes.
- The backplane is **invalidation, not replication**. Backplane + bare options (no soft timeout, no fail-safe) → invalidation triggers a thundering herd on the source. Always pair backplane with `FactorySoftTimeout` + fail-safe so concurrent misses across replicas degrade to stale instead of stampede.
- Eager refresh executes the factory **on the thread pool** in the background. A heavy factory triggered eagerly across many keys can saturate threads — bound parallelism upstream (semaphore in the factory, or rely on per-key single-flight which serializes by key but not globally).
- `FactorySoftTimeout` requires fail-safe to be useful — without a stored stale value, "soft timeout" with nothing to fall back to just returns null/throws. The two are paired by design.
- Adaptive caching mutates `ctx.Options` — those changes apply only to **this** call's entry, not to the policy. Don't try to "learn" TTLs by mutating the policy itself.
- Tag invalidation requires the backplane (for cross-replica) **and** a tag store. In recent versions FusionCache supports tag-based invalidation natively; in older versions it was via plugins. Check the version you're on.
- Serializer choice dominates L2 cost. `System.Text.Json` is ergonomic; MessagePack and Protobuf are 2–5× smaller and faster on hot paths. Switch via `WithSerializer(...)`. Don't mix serializers across versions — old entries become unreadable.
- Auto-recovery retries failed L2 writes in the background. If your L2 is "down because credentials rotated," it'll never recover until you restart — auto-recovery handles transient blips, not config bugs.
- `IFusionCache` is registered as singleton. Don't store per-request state in it (e.g., `HttpContext.User`); compute keys from request data and pass them in.
- Open-source, MIT-licensed, very actively maintained — but check your org's policy on non-Microsoft deps. The library is mature (5+ years, used in production at scale) but the support model is GitHub issues, not a Microsoft SLA.
- HybridCache and FusionCache **can coexist** — use HybridCache for simple stuff, FusionCache for the endpoints that need fail-safe. Same Redis works as L2 for both. Don't see them as either/or.
