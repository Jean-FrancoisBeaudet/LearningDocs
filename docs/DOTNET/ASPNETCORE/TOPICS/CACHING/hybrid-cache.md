# HybridCache

_Targets .NET 10 / C# 14. See also: [StackExchange.Redis](./stackexchange-redis.md), [FusionCache](./fusion-cache.md), [OutputCache](./output-cache.md)._

`HybridCache` (Microsoft, GA in .NET 9) is the first-party answer to a problem every team eventually solves badly: combining `IMemoryCache` (fast, in-process) with `IDistributedCache` (shared across replicas) while also handling cache stampedes, serialization, and tag-based invalidation. Before HybridCache, "memory-with-Redis-fallback" was a hand-rolled wrapper in every codebase. Now it's a single API.

## What it gives you that the lower-level APIs don't

| Feature | `IMemoryCache` | `IDistributedCache` | `HybridCache` |
|---|---|---|---|
| In-process L1 | ✓ | — | ✓ |
| Out-of-process L2 | — | ✓ | ✓ (optional) |
| Stampede protection (single-flight) | — | — | ✓ (per process) |
| Built-in serialization | — | — | ✓ (JSON default, pluggable) |
| Strongly-typed `T` API | — (object) | — (byte[]) | ✓ `<T>` |
| Tag-based invalidation | — | — | ✓ |
| Async factory (`GetOrCreateAsync`) | partial | — | ✓ |

If you've ever written `_memCache.GetOrCreateAsync(key, async _ => { var bytes = await _distCache.GetAsync(key); ... })` — that's HybridCache.

## Wiring

```csharp
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration.GetConnectionString("Redis"));

builder.Services.AddHybridCache(o =>
{
    o.MaximumPayloadBytesPerEntry = 1 << 20;             // 1 MiB cap
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration      = TimeSpan.FromMinutes(10),      // L2 absolute
        LocalCacheExpiration = TimeSpan.FromMinutes(2),  // L1 absolute (≤ Expiration)
    };
});
```

If `IDistributedCache` is registered, HybridCache uses it as L2; if not, it runs as L1-only (which is `IMemoryCache` plus single-flight and a typed API — still useful).

## Usage

```csharp
public sealed class CatalogService(HybridCache cache, IDbContextFactory<AppDb> dbf)
{
    public ValueTask<Product?> GetAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            $"product:{id}",
            async token =>
            {
                await using var db = await dbf.CreateDbContextAsync(token);
                return await db.Products.FindAsync([id], token);
            },
            tags: ["products", $"product:{id}"],
            cancellationToken: ct);

    public ValueTask InvalidateProductAsync(int id, CancellationToken ct) =>
        cache.RemoveByTagAsync($"product:{id}", ct);
}
```

The factory delegate runs **only on miss**, and only **once per key per process** even under concurrent load — single-flight is the whole point.

## `HybridCacheEntryOptions` and flags

```csharp
var opts = new HybridCacheEntryOptions
{
    Expiration           = TimeSpan.FromHours(1),
    LocalCacheExpiration = TimeSpan.FromMinutes(5),
    Flags = HybridCacheEntryFlags.DisableLocalCacheRead
          | HybridCacheEntryFlags.DisableDistributedCacheWrite
};

await cache.GetOrCreateAsync(key, factory, opts, cancellationToken: ct);
```

Flags let you skip L1 or L2 per call, useful for "always re-read from Redis" or "warm Redis but don't pollute L1." Don't reach for them by default.

## Custom serialization

Default is `System.Text.Json`. Override per-type for hot paths:

```csharp
builder.Services
    .AddHybridCache()
    .AddSerializer<Product, MessagePackHybridSerializer>();
```

Implement `IHybridCacheSerializer<T>` — straightforward `Serialize(T)` / `Deserialize(ReadOnlySequence<byte>)`. For polymorphic graphs or self-referencing types, write the serializer; don't fight `System.Text.Json` defaults.

## Tags and invalidation

```csharp
await cache.GetOrCreateAsync(key, factory,
    tags: ["category:electronics", "supplier:42"], cancellationToken: ct);

// Later — bulk evict everything with that tag
await cache.RemoveByTagAsync("supplier:42", ct);
```

Tag implementation depends on the L2 store. Redis: yes (HybridCache maintains tag→key index). Some other `IDistributedCache` implementations don't — verify with your store before relying on it.

## When to use

- You currently hand-roll `IMemoryCache + IDistributedCache` wrappers — replace them with HybridCache.
- You want stampede protection on misses and don't want to write a `SemaphoreSlim` per key.
- You want strongly-typed `<T>` access without writing a serializer wrapper.
- You're starting a greenfield .NET 9+ service that needs caching.

## When NOT to use

- You need **cross-replica** single-flight (HybridCache de-dupes within one process; N replicas can still concurrently regenerate). Either accept the cost, or use [FusionCache](./fusion-cache.md) with backplane + soft timeouts, or front the source with its own coordination.
- You need fail-safe (return stale on factory exception) — HybridCache does not, [FusionCache](./fusion-cache.md) does.
- You need eager refresh / adaptive TTL — same answer, FusionCache.
- L1-only with ad-hoc API surface — `IMemoryCache` is fine, you don't need the ceremony.

## Comparison vs FusionCache

| Capability | HybridCache | [FusionCache](./fusion-cache.md) |
|---|---|---|
| L1 + L2 | ✓ | ✓ |
| Stampede protection | ✓ (per-process) | ✓ (per-process + cross-process via backplane patterns) |
| Tag invalidation | ✓ | ✓ |
| Typed API | ✓ | ✓ |
| **Fail-safe** (return stale on factory error) | — | ✓ |
| **Soft / hard timeouts** (timeout factory, return stale) | — | ✓ |
| **Eager refresh** (background revalidate near expiry) | — | ✓ |
| **Adaptive caching** (factory adjusts TTL per result) | — | ✓ |
| **Backplane** (Redis pub/sub L1 invalidation) | — | ✓ |
| First-party Microsoft support | ✓ | — (open source, very active) |

HybridCache is the **conservative default**. FusionCache is the **power tool**. Many teams adopt HybridCache, hit a fail-safe or eager-refresh requirement six months later, and then add FusionCache.

**Senior-level gotchas:**
- Single-flight is **per process**, not cluster-wide. Three replicas with cold L1 + cold L2 will all execute the factory once each (3 hits to your DB) under a thundering herd. Mitigations: warm L2 ahead of deploys, accept the 3× factor, or use a coordination mechanism upstream of HybridCache (advisory lock in Postgres/Redis).
- `LocalCacheExpiration` must be ≤ `Expiration`. If you set it higher, L1 outlives L2 and you'll serve from L1 longer than you intended — invalidations through L2 won't propagate until L1's own TTL fires (HybridCache has no L2-driven L1 invalidation built in).
- `MaximumPayloadBytesPerEntry` **silently drops** entries that exceed it (the factory result is returned but never cached). Log misses if you suspect this.
- The default JSON serializer cannot handle `IAsyncEnumerable<T>`, `Task<T>`, raw `Stream`, or types without a parameterless constructor (without source-gen). Write a typed `IHybridCacheSerializer<T>` for anything non-trivial.
- `RemoveByTagAsync` is **eventually consistent**: it issues the eviction but readers in flight may briefly see stale data, especially across replicas. Don't pair it with "user just saved, redirect to read page" without a server-side `DisableLocalCacheRead` on the next fetch.
- Factory exceptions are **not cached** — the next request will retry the factory. Without rate limiting upstream, an outage at the source becomes an amplified retry storm. HybridCache does not provide circuit-breaker semantics; pair it with Polly or move to FusionCache for fail-safe.
- The L2 contract is `IDistributedCache`, so payload limits inherit from the store. Redis defaults to 512 MiB per value but cloud variants cap much lower (Azure Cache: 64 MiB but practical limit much smaller for latency reasons). Combined with `MaximumPayloadBytesPerEntry`, you should set explicit caps both places.
- Tag indexes consume keyspace in your L2. A high-cardinality tag (`tag: "user:{id}"` for millions of users) creates a tag→key set per user — design tags for *bulk* invalidation, not per-entity tracking.
- Don't pass mutable reference types as cached values. The L1 returns the **same instance** to all callers — mutating it after retrieval mutates the cached state. Cache `record` types or DTOs you treat as immutable.
- If you don't register an `IDistributedCache`, HybridCache silently runs L1-only. That's not a bug, but it makes "production has no Redis configured" a quiet performance regression rather than a startup failure. Add a startup health check that verifies the L2 is wired if you require it.
