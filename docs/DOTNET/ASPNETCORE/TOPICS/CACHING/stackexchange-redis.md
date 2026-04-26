# StackExchange.Redis

_Targets .NET 10 / C# 14. See also: [HybridCache](./hybrid-cache.md), [FusionCache](./fusion-cache.md), [OutputCache](./output-cache.md)._

`StackExchange.Redis` is the de-facto .NET Redis client (built by Stack Overflow). Most ASP.NET Core apps interact with it indirectly through `IDistributedCache` (via `Microsoft.Extensions.Caching.StackExchangeRedis`), but the raw client surface (`IConnectionMultiplexer`, `IDatabase`) is what you reach for when you need pub/sub, transactions, scripting, or anything beyond key/value GET/SET.

## The two layers

```
Your code
   │
   ├── IDistributedCache  ◄── byte[] in/out, sliding/absolute TTL, no Redis-specific features
   │
   └── IConnectionMultiplexer / IDatabase  ◄── full Redis API: pub/sub, scripts, transactions, streams
```

Pick the lowest layer that does what you need. `IDistributedCache` is sufficient for "fetch JSON, cache for 10 min." Drop to `IDatabase` when you need lists, sorted sets, Lua scripts, atomic increments, or pub/sub.

## Wiring `IDistributedCache`

```csharp
builder.Services.AddStackExchangeRedisCache(opt =>
{
    opt.Configuration = builder.Configuration.GetConnectionString("Redis");
    opt.InstanceName  = "myapp:";   // prefix prepended to every key
});
```

Then inject and use:

```csharp
public sealed class OrderReader(IDistributedCache cache)
{
    public async Task<Order?> GetAsync(int id, CancellationToken ct)
    {
        var key   = $"order:{id}";
        var bytes = await cache.GetAsync(key, ct);
        if (bytes is not null) return JsonSerializer.Deserialize<Order>(bytes);

        var order = await LoadFromDbAsync(id, ct);
        await cache.SetAsync(
            key,
            JsonSerializer.SerializeToUtf8Bytes(order),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) },
            ct);
        return order;
    }
}
```

`IDistributedCache` is **bytes in / bytes out**. Serialization is your problem — `System.Text.Json` for ergonomics, MessagePack/Protobuf when payload size and CPU matter. Per-app, pick one and stick with it.

## Wiring the multiplexer directly

```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
    ConnectionMultiplexer.Connect(new ConfigurationOptions
    {
        EndPoints = { "redis-prod:6380" },
        Ssl = true,
        Password = secret,
        AbortOnConnectFail = false,   // critical for cloud Redis (Azure, ElastiCache)
        ConnectRetry = 5,
        ConnectTimeout = 5_000,
        SyncTimeout = 5_000,
        AsyncTimeout = 5_000,
    }));
```

`ConnectionMultiplexer` is **already** multiplexed — one instance per app, registered as a singleton. Creating a new one per request is a classic anti-pattern that exhausts sockets and starves the thread pool. From the singleton, get an `IDatabase` per call (cheap, returns a stateless façade):

```csharp
public sealed class LeaderboardService(IConnectionMultiplexer mux)
{
    public Task<long> IncrementScoreAsync(string user, long delta) =>
        mux.GetDatabase().SortedSetIncrementAsync("leaderboard", user, delta);
}
```

## Pub/Sub, transactions, scripts

Beyond key/value, the multiplexer exposes:

- `mux.GetSubscriber()` — pub/sub for backplane invalidation, lightweight events.
- `db.CreateTransaction()` — `MULTI`/`EXEC` with optimistic conditions (`AddCondition(...)`).
- `db.ScriptEvaluateAsync(...)` — Lua for atomic compound operations (rate limiting, conditional set with TTL refresh).
- `db.HashSet`, `db.ListLeftPush`, `db.SortedSetAdd` — full datatype API.
- `IServer.Keys(pattern: ...)` — uses `SCAN` cursor (NOT `KEYS`), still expensive on large keyspaces.

```csharp
// Atomic "set if missing, otherwise return existing TTL" via Lua
const string Lua = """
    if redis.call('SET', KEYS[1], ARGV[1], 'NX', 'EX', ARGV[2]) then return -1
    else return redis.call('TTL', KEYS[1]) end
    """;

var ttl = (long)await db.ScriptEvaluateAsync(Lua, [key], [token, 60]);
```

## Connection topology

| Topology | What changes for the client |
|---|---|
| **Standalone** | One endpoint. 16 logical DBs available (`db.GetDatabase(3)`). |
| **Sentinel** | Configure with sentinel endpoints; client discovers master via Sentinel API. Failover handled transparently. |
| **Cluster** | Multiple endpoints; multiplexer slot-routes commands. **Only DB 0** exists. Multi-key ops require all keys in the same slot — use `{tag}` curly-brace hash tags: `cart:{user42}:items`. |

Cloud Redis (Azure Cache, ElastiCache, MemoryDB) is usually cluster-mode-enabled — code must not assume `db.GetDatabase(N)` for `N > 0` and must hash-tag co-located keys.

## Key design (non-negotiable)

- **Namespace prefix per service** — `orders:`, `auth:`, `feature-flags:`. Avoid collisions across teams sharing a Redis.
- **Schema version suffix** when serialization can change — `order:v2:42`. Lets you deploy serializer changes without flushing.
- **TTLs on everything** unless you have a deletion policy. A cache without expiry is a memory leak with extra steps.
- **No `KEYS *` ever.** Blocks the server. Use `SCAN`-based `IServer.Keys()` for admin work, or maintain index sets yourself.

## When to use what

- Just need byte-level cache with TTL? → `IDistributedCache`.
- Need pub/sub, atomic counters, lists, sorted sets, scripts? → `IDatabase` directly.
- Need an in-memory L1 in front of Redis with stampede protection? → [HybridCache](./hybrid-cache.md) or [FusionCache](./fusion-cache.md), both built on top of `IDistributedCache`.
- Caching full HTTP responses? → [OutputCache](./output-cache.md) with the Redis store.

**Senior-level gotchas:**
- `KEYS *` blocks the entire Redis server (single-threaded command loop). Use `SCAN` (`IServer.Keys()` does this); even then, large keyspaces hurt latency. Prefer maintaining your own index sets for "list all keys with prefix X."
- `IDistributedCache.RefreshAsync` only resets the **sliding** expiration window — it does NOT return the value and it does nothing for absolute TTLs. Frequent confusion.
- `AsyncTimeout` and `SyncTimeout` default to 5 seconds. Under thread-pool starvation, async continuations queue and trip `RedisTimeoutException` even when Redis is healthy. The exception message includes a `THREADS:` block — read it; if `IOCP` or `WORKER` queue length is non-zero, the bug is your app, not Redis.
- `ConnectionMultiplexer` is **singleton per app**. Creating it per request is a top-3 reason teams blame Redis for outages they caused.
- `AbortOnConnectFail = true` (the default) means a startup-time connectivity blip permanently breaks the multiplexer. For cloud Redis, **always** set `false` — the multiplexer will reconnect in the background.
- In cluster mode, multi-key operations (`MGET`, `MSET`, transactions, Lua with multiple `KEYS`) require all keys to hash to the same slot. Use `{...}` hash tags: `user:{42}:profile` and `user:{42}:cart` co-locate. Without tags you get `MOVED`/`CROSSSLOT` errors.
- `db.StringSetAsync(key, value, expiry, when: When.NotExists)` is the idiomatic SET-NX-with-TTL primitive (atomic). Don't roll your own with `SETNX` + `EXPIRE` — that pair is non-atomic.
- `IConnectionMultiplexer.GetDatabase()` is cheap (returns a struct façade). Don't cache the `IDatabase` reference — caching it ties you to a specific DB number and obscures connection-state checks.
- Pipelining is automatic for `async` calls — issuing 100 awaited operations sequentially is **not** 100 round-trips if you `Task.WhenAll` them. Sequential `await` IS 100 RTTs. Batch with `WhenAll` for bulk ops.
- Pub/Sub messages are **not persisted** and are at-most-once. If you need guaranteed delivery for cache invalidation across replicas, use Redis Streams (`XADD`/`XREADGROUP`) or an actual broker — not pub/sub.
- The Redis OSS license changed in 2024 (RSALv2/SSPL). Cloud-managed Redis is unaffected; self-hosting at scale may push you toward Valkey (a Linux Foundation fork). The .NET client speaks the same protocol — `StackExchange.Redis` works against Valkey unchanged.
