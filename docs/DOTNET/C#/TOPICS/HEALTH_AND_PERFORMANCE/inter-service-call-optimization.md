# Inter-service call optimization

_Targets .NET 10 / C# 14. See also: [Connection pooling](./connection-pooling.md), [I/O saturation](./io-saturation.md), [Service mesh latency](./service-mesh-latency.md), [Serialization overhead](./serialization-overhead.md), [Backpressure handling](./backpressure-handling.md), [Streaming patterns](./streaming-patterns.md), [HttpClient and HttpClientFactory](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/httpclient-and-httpclientfactory.md), [gRPC](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/grpc.md), [REST](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/rest.md)._

**Inter-service call optimization** is reducing the wall-clock and resource cost of N synchronous outbound calls between services. The four levers — call count, per-call cost, serialization, and concurrency — apply independently; most teams reach for "more concurrency" first when the win is almost always in "fewer calls" or "smaller calls."

> One in-region HTTP/2 call to a healthy service is ~1–3 ms. Five sequential calls is 5–15 ms — and that's the *p50*. The p99 is where the request actually lives, and tail amplification means each additional dependency makes the p99 worse non-linearly.

## Tail amplification

If one downstream has p99 = 50 ms and p50 = 5 ms, calling it once costs ~5 ms typically. Calling it 5 times in parallel and waiting for *all* responses gives you the worst of the five — and the probability that *at least one* hits the slow tail rises with N:

```
P(at least one slow) = 1 − (1 − p)^N
                    ≈ N × p  for small p
```

5 calls × 1% chance each of being a tail = 5% chance the request is slow. **Adding dependencies is multiplicative on the tail, even when each one is fast.** This is why hedging, request collapsing, and aggregator endpoints exist.

## The four levers

| Lever | Technique | Order-of-magnitude impact |
|---|---|---|
| Call count | BFF / aggregator endpoint, GraphQL, REST `?include=`, server-side joins | 10× (5 calls → 1) |
| Per-call cost | HTTP/2 or HTTP/3 multiplexing, gRPC, payload trimming, server-side projection | 2–5× |
| Serialization | `System.Text.Json` source generators, MemoryPack/MessagePack on internal links | 2–10× CPU |
| Concurrency | `Task.WhenAll`, `Parallel.ForEachAsync`, request hedging | bounded by downstream capacity |

## Lever 1: reduce the call count

A typical request chain looks like this:

```
client → bff → orders ↘
                customers ↘ → 5 sequential calls, 250 ms wall-clock
                pricing ↘
                inventory ↘
                shipping ↘
```

The BFF (backend-for-frontend) pattern collapses this to one logical call. Inside, the BFF fans out — but fans out *in parallel* and to services that are colocated with it:

```csharp
public async Task<OrderViewModel> GetOrderAsync(Guid id, CancellationToken ct)
{
    var orderTask    = _orders.GetAsync(id, ct);
    var customerTask = _customers.GetByOrderAsync(id, ct);
    var pricingTask  = _pricing.GetForOrderAsync(id, ct);
    var inventoryTask = _inventory.GetForOrderAsync(id, ct);
    var shippingTask = _shipping.GetForOrderAsync(id, ct);

    await Task.WhenAll(orderTask, customerTask, pricingTask, inventoryTask, shippingTask)
              .ConfigureAwait(false);

    return OrderViewModel.From(orderTask.Result, customerTask.Result, /* … */);
}
```

5 sequential calls (250 ms) → 1 call to BFF + 5 parallel calls inside it (45 ms p50, bounded by the slowest of the five). If most are cache hits, even faster.

GraphQL solves the same problem differently: one query, server resolves the dependency graph. The trade is non-trivial server-side complexity and per-resolver N+1 risk; use it when the client-side flexibility is worth the operational cost.

REST with `?include=customer,pricing,shipping` is the lightweight version — same idea, less infrastructure.

## Lever 2: reduce per-call cost

**HTTP/2 over HTTP/1.1** is the biggest free win. One TCP connection, multiplexed streams, no head-of-line blocking. .NET enables HTTP/2 automatically when both ends advertise it (TLS ALPN); the client side needs:

```csharp
builder.Services.AddHttpClient<OrdersClient>(c =>
{
    c.BaseAddress = new Uri("https://orders.svc/");
    c.DefaultRequestVersion = HttpVersion.Version20;
    c.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrLower;
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    EnableMultipleHttp2Connections = true,        // beyond per-connection stream limit
    PooledConnectionLifetime = TimeSpan.FromMinutes(5),
});
```

`EnableMultipleHttp2Connections = true` matters under high RPS: HTTP/2 connections are stream-limited by the server (`SETTINGS_MAX_CONCURRENT_STREAMS`, often 100). Once saturated, additional requests queue on the existing connection unless the handler is told it can open a second one.

**gRPC over JSON-REST** when the schema is stable and the cost matters. You get HTTP/2 always, protobuf wire format, code-gen on both ends, and cancellation/deadline semantics built in. Cross-link: [grpc.md](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/grpc.md).

**Payload trimming** is the cheapest of all. If your client uses 3 fields out of 50, expose a projection endpoint or a `?fields=id,name,total` parameter. The wire bytes savings dwarfs anything you'll do with a faster JSON parser.

## Lever 3: serialization

`System.Text.Json` source generators eliminate reflection and per-call closure allocations:

```csharp
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(IReadOnlyList<Order>))]
internal partial class AppJsonContext : JsonSerializerContext;

// Per call
var order = await response.Content.ReadFromJsonAsync(AppJsonContext.Default.Order, ct);
```

Source generation cuts deserialization CPU by 30–50% and removes most of the allocations on the hot path; under NativeAOT it's required. See [serialization-overhead.md](./serialization-overhead.md).

**Stream the body instead of buffering** when the response is large:

```csharp
var resp = await _http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
resp.EnsureSuccessStatusCode();
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Item>(
                   await resp.Content.ReadAsStreamAsync(ct), AppJsonContext.Default.Options, ct))
    Process(item);
```

`HttpCompletionOption.ResponseHeadersRead` returns control as soon as the headers arrive — body bytes flow as they're consumed. Without it, `GetAsync` waits for the full body (default `ResponseContentRead`).

For internal-service-to-internal-service links where both sides are .NET, **MemoryPack** or **MessagePack** beat JSON by 5–10× CPU and 2–4× bytes. The trade is human-illegibility on the wire and tighter coupling between sides; only use it when both sides ship together.

## Lever 4: bounded concurrency

`Task.WhenAll` over 5 known calls is fine. `Task.WhenAll` over 10000 generated tasks is a self-DDoS:

```csharp
// WRONG: 10k inflight, downstream throttles, retries make it worse.
var results = await Task.WhenAll(items.Select(i => _client.LookupAsync(i, ct)));

// RIGHT: 32 inflight, downstream stays healthy, total time often lower.
var results = new ConcurrentBag<Result>();
await Parallel.ForEachAsync(items,
    new ParallelOptions { MaxDegreeOfParallelism = 32, CancellationToken = ct },
    async (item, c) => results.Add(await _client.LookupAsync(item, c)));
```

The right MaxDegreeOfParallelism is roughly the downstream's *capacity / your share* — not "as many cores as you have."

## Resilience without amplification

A retry policy without back-off and jitter is a DoS multiplier. Three retries × 100 callers × downstream blip = 400 calls instead of 100, all within the same window.

```csharp
builder.Services.AddHttpClient<OrdersClient>(c => c.BaseAddress = new("https://orders.svc/"))
.AddStandardResilienceHandler(o =>
{
    o.AttemptTimeout.Timeout    = TimeSpan.FromSeconds(2);
    o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(8);
    o.Retry.MaxRetryAttempts    = 3;
    o.Retry.BackoffType         = DelayBackoffType.Exponential;
    o.Retry.UseJitter           = true;
    o.CircuitBreaker.FailureRatio       = 0.5;
    o.CircuitBreaker.MinimumThroughput  = 20;
    o.CircuitBreaker.SamplingDuration   = TimeSpan.FromSeconds(30);
});
```

`AddStandardResilienceHandler` (Microsoft.Extensions.Http.Resilience) bundles timeout / retry / rate-limiter / circuit-breaker / hedging in the v8+ recommended order. The pipeline order matters: timeout outside retry, retry outside circuit breaker, circuit breaker outside hedging.

The timeout invariant: **per-attempt < per-pipeline < `HttpClient.Timeout` < server's keep-alive**. Inverting any of these creates phantom errors.

## Caching: client-side and on the wire

```csharp
builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),
        LocalCacheExpiration = TimeSpan.FromSeconds(30),
    };
});

public sealed class OrdersClient(HttpClient http, HybridCache cache)
{
    public ValueTask<Order> GetAsync(Guid id, CancellationToken ct) =>
        cache.GetOrCreateAsync($"order:{id}", async (token) =>
            await http.GetFromJsonAsync<Order>($"orders/{id}", AppJsonContext.Default.Order, token)
                ?? throw new InvalidOperationException(), cancellationToken: ct);
}
```

`HybridCache` (.NET 9+) coalesces concurrent requests for the same key — N callers waiting on a cold cache produce **one** downstream call, not N. This single feature usually beats every other client-side optimization combined for read-heavy services.

Wire-level caching: `Cache-Control`, `ETag`/`If-None-Match` for `304 Not Modified`. `HttpClient` doesn't honor these by default — you need `Microsoft.Extensions.Caching.HttpClientFactory` or a custom delegating handler.

## Worked example: 5 sequential REST calls → BFF + cache

Before — 250 ms p50 (5 × 50 ms RTT, sequential):

```csharp
var order = await _orders.GetAsync(id, ct);
var customer = await _customers.GetAsync(order.CustomerId, ct);
var pricing = await _pricing.GetAsync(order.Id, ct);
var inventory = await _inventory.GetAsync(order.Items.Select(i => i.Sku), ct);
var shipping = await _shipping.GetAsync(order.Id, ct);
return OrderViewModel.From(order, customer, pricing, inventory, shipping);
```

After — 45 ms p50 (one fan-out, parallel, cache hits on customer/pricing):

```csharp
var orderTask = _cache.GetOrCreateAsync($"order:{id}", _ => _orders.GetAsync(id, ct), ct);
var customerTask = orderTask.ContinueWithAsync(o =>
    _cache.GetOrCreateAsync($"customer:{o.CustomerId}", _ => _customers.GetAsync(o.CustomerId, ct), ct));
var pricingTask = _cache.GetOrCreateAsync($"pricing:{id}", _ => _pricing.GetAsync(id, ct), ct);
var inventoryTask = orderTask.ContinueWithAsync(o => _inventory.GetAsync(o.Items.Select(i => i.Sku), ct));
var shippingTask = _shipping.GetAsync(id, ct);

await Task.WhenAll(orderTask, customerTask, pricingTask, inventoryTask, shippingTask);
return OrderViewModel.From(orderTask.Result, customerTask.Result, /* … */);
```

(`ContinueWithAsync` is a helper for chaining `Task<T>` continuations; native `Task.WhenAll` doesn't take a dependency graph.)

## Telemetry

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddHttpClientInstrumentation().AddAspNetCoreInstrumentation())
    .WithMetrics(m => m.AddHttpClientInstrumentation().AddRuntimeInstrumentation());
```

`System.Net.Http` exposes `http.client.request.duration` (histogram) and `http.client.active_requests` (UpDown). The histogram broken down by `http.response.status_code` and `server.address` is the entry point for any inter-service performance investigation.

W3C Trace Context (`traceparent`, `tracestate`) propagates automatically through `HttpClient` — the trace shows you which downstream is the slow one without guessing.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| 5 sequential `await`s where 4 are independent | 4× wall-clock latency for no reason |
| `Task.WhenAll` over a generated/unbounded set | Self-DDoS; downstream throttles; retries amplify |
| `new HttpClient()` per call | Socket exhaustion, no DNS rotation — see [io-saturation.md](./io-saturation.md) |
| Retry without jitter | Synchronized retry storms across the fleet |
| Missing `HttpCompletionOption.ResponseHeadersRead` on large bodies | Buffers full body before processing — TTFB ↑ |
| `JsonSerializer.Deserialize<List<T>>(stream)` over a 50 MB body | Materializes everything; use `DeserializeAsyncEnumerable` |
| `CancellationToken` not threaded through every layer | Long-running calls survive request cancellation, holding pool slots |
| HTTP/1.1 to a downstream that supports HTTP/2 | Per-call connection acquisition cost; head-of-line blocking |
| Per-request `HybridCache` lookup with no key bound | Cache fills with one-shot keys; eviction churns; hit rate near zero |
| Client-side timeout > server-side timeout | Server hangs up; client sees `IOException` instead of `TimeoutException` |
| Same logical service called via different `BaseAddress` strings | Two pools, two TLS sessions, doubled handshake cost |

## Senior-level gotchas

- **Tail latency is multiplicative across fan-out.** "Each downstream is fine" doesn't make the aggregate fine. Plot p99 of *the request*, not p99 of each call. Hedging (issuing a second call after the first looks slow) is the standard mitigation; .NET 8+ `AddStandardHedgingHandler` provides it.
- **HTTP/2 stream limits are per-connection.** Server's `SETTINGS_MAX_CONCURRENT_STREAMS` is typically 100. With one connection and 200 concurrent calls, 100 wait on the server side. Set `EnableMultipleHttp2Connections = true` and watch `connections-current-total`.
- **`HttpClient.Timeout` doesn't cover DNS resolution.** A failing DNS server can hang for ~30 seconds before the request even starts. Set `DefaultDnsResolver.QueryTimeout` (in .NET 8+ via `SocketsHttpHandler.ConnectTimeout`).
- **gRPC deadlines and HTTP timeouts are different things.** gRPC uses `CallOptions.Deadline`; `HttpClient.Timeout` is the outer ceiling. The deadline propagates as `grpc-timeout` header to the server; the HTTP timeout doesn't. If you want server-side cancellation, set both.
- **Retry policies on read endpoints are usually safe; on write endpoints they require idempotency.** Ship a retry-safe contract (`Idempotency-Key` header, server-side dedupe table) before turning on retries for POSTs. Otherwise you'll have charges-applied-twice in production.
- **`Polly` circuit breaker state is per process.** A pod opens its breaker; the other 99 pods don't know. For coordinated decisions, use a service-mesh-level breaker (Istio outlier detection) — see [service-mesh-latency.md](./service-mesh-latency.md).
- **`HybridCache` cache-stampede protection is per-process.** Cross-process stampedes still happen on cache expiry; if every pod expires its key at the same wall-clock moment, you get fleet-wide cold-cache fan-out. Add jitter to expiration (`Expiration = base + Random.Shared.Next(0, jitterMs)`).
- **JSON source generators don't reflect type changes at runtime.** Adding a property after publishing requires regenerating the source generator output; otherwise the new field silently isn't serialized. Build pipeline must regenerate on schema change.
- **`MemoryPack`/`MessagePack` evolution rules are strict.** Adding a field requires a default value; reordering breaks compatibility. Treat the schema as a contract, version it explicitly, support N-1.
- **HTTP/3 over QUIC has different failure modes.** UDP rate-limiting middleboxes, NAT timeouts, ECMP path inconsistencies. Worth it for cross-region public traffic; rarely worth it inside a VPC. Test before enabling.
- **`Task.WhenAll` propagates the *first* exception only by default; the others land in `Task.Exception`.** Under partial failure, you lose context. Use `Task.WhenAll(...).WaitAsync(ct)` and inspect each task individually for production diagnostics.
- **Trace propagation across queue hops doesn't happen automatically.** If a request goes HTTP → Service Bus → another service, the W3C trace context lives only on the HTTP side unless your queue producer attaches `traceparent` as a message property. OpenTelemetry instrumentation for Service Bus / RabbitMQ does this; hand-rolled consumers usually don't.
- **`HttpClient` typed clients with conflicting handler config get weird.** Two typed clients pointing to the same host but configured with different `SocketsHttpHandler` instances mean two separate pools. Either share the handler explicitly or make the configs identical.
- **gRPC streaming for "infinite" calls leaks gracefully on the wire but ungracefully in the client.** A client-streaming call held open across deployments holds a connection slot per stream until cancelled. Set sensible `IdleTimeout` and force reconnect on long-running streams.
- **The fastest call is the one you don't make.** Most "slow service" investigations end up being "we call this thing five times per request and three of those are unnecessary." Profile the *call graph*, not the call.
- **`ConfigureAwait(false)` doesn't matter in ASP.NET Core**, but it does in libraries shared with WPF/WinForms or older ASP.NET. If your client library is intended for embedding, add it; if it's only for ASP.NET Core, it's noise.
