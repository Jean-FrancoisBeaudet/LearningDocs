# I/O saturation

_Targets .NET 10 / C# 14. See also: [Connection pooling](./connection-pooling.md), [System.IO.Pipelines](./system-io-pipelines.md), [Backpressure handling](./backpressure-handling.md), [Streaming patterns](./streaming-patterns.md), [Inter-service call optimization](./inter-service-call-optimization.md), [Serialization overhead](./serialization-overhead.md), [dotnet-counters](./dotnet-counters.md)._

**I/O saturation** is a *capacity* problem, not a *code* problem.

> The bottleneck is the bytes-per-second the OS, NIC, disk, or remote service can deliver — the application's threads are parked in I/O waits, and adding code optimization or more workers will not help. Either the underlying capacity has to increase, or the application has to ask for less of it.

The trap is that I/O saturation looks identical to thread-pool starvation from the symptom side: requests queue, latency climbs, the service becomes unresponsive. The diagnosis is what differentiates them — and the fix lists do not overlap.

| Problem | CPU | Workers | Queue | Root |
|---|---|---|---|---|
| **I/O saturation** | Low | Low / steady | Outbound queue rises (`current-requests`) | Downstream / disk / NIC at limit |
| **Thread-pool starvation** | Low | Climbing fast | `threadpool-queue-length` rises | Sync-over-async — see [thread contention](./thread-contention.md) |
| **CPU saturation** | ~100% | Steady | Falls when CPU pinned | Algorithm |
| **GC bound** | Bursts | Steady | Latency tracks GC pauses | See [GC pressure](./gc-pressure.md) |

## Quantitative signals

You're I/O saturated when, sustained at peak load:

- **`System.Net.Http` counter `current-requests` rising** while `requests-failed` stays low — outbound calls are queued behind a connection pool.
- **`requests-queue-duration`** > a few hundred milliseconds — connections are not available immediately.
- **OS `Disk Queue Length` > 2** sustained on a service that does disk I/O, or NIC bytes/sec at link cap.
- **TCP `RetransSegs` rising** — packet loss / saturation on the network path.
- **`SocketException: Only one usage of each socket address` / `address already in use`** — ephemeral port exhaustion from `new HttpClient()` per call.
- **CPU < 50%, throughput flat or falling** as concurrency rises — adding callers makes things worse.

## Measurement recipe

1. **App-level counters**:
   ```bash
   dotnet-counters monitor System.Net.Http System.Net.NameResolution System.Net.Sockets --process-id <pid>
   ```
   `connections-current-total` (HttpClient pool size), `requests-queue-duration` (waiting for a slot), and `dns-lookups-duration` together describe outbound health.
2. **Kestrel-side counters**:
   ```bash
   dotnet-counters monitor Microsoft.AspNetCore.Hosting Microsoft.AspNetCore.Server.Kestrel --process-id <pid>
   ```
   If `current-requests` plateaus while CPU is low and the *outbound* queue is also rising, the saturation is downstream of you, not in your service.
3. **OS counters**:
   - Linux: `iostat -xz 1`, `ss -tan | wc -l`, `sar -n DEV 1`.
   - Windows: `Perfmon → \PhysicalDisk(*)\Avg. Disk Queue Length`, `\Network Interface(*)\Bytes Total/sec`.
4. **Trace the call path**:
   ```bash
   dotnet-trace collect --process-id <pid> --providers Microsoft-DotNETCore-SampleProfiler,Microsoft-Diagnostics-DiagnosticSource
   ```
   With `Microsoft.Extensions.Http.Logging` enabled, you'll see per-request `Start`/`Stop` and the time-on-the-wire vs time-waiting-for-a-slot.
5. **Distributed tracing** — OpenTelemetry spans on outbound calls are the only sane way to diagnose multi-hop saturation. The `http.client.request.duration` histogram from `System.Net.Http` is OTel-native in .NET 8+.

## The usual suspects

In rough descending order:

- **`new HttpClient()` per call.** Each instance owns a `SocketsHttpHandler`, each handler opens its own connection pool, and the underlying TCP sockets sit in `TIME_WAIT` for ~60 seconds after disposal. A few hundred requests per second exhausts ephemeral ports. **Use `IHttpClientFactory`** (typed clients).
- **Long-lived `HttpClient` without `PooledConnectionLifetime`.** The opposite mistake: a singleton handler caches the resolved IP forever. When the downstream rolls (Kubernetes, ASG, blue/green), you keep hammering the dead one. **Set `PooledConnectionLifetime` to 5–15 minutes.**
- **Sync I/O on async paths.** `File.ReadAllText`, `Stream.Read`, `WebClient.DownloadString`. The thread blocks in the kernel — same effect as sync-over-async on managed code, plus you don't get the IOCP fast path.
- **`HttpClient.GetAsync` followed by `ReadAsStringAsync` on a multi-MB body.** Buffers the whole response into memory before you process a byte. Use `HttpCompletionOption.ResponseHeadersRead` plus stream the body, or use `System.Net.Http.Json` deserializers that read incrementally.
- **No timeout.** Default `HttpClient.Timeout` is 100 seconds. If the downstream hangs, your pool fills with dead waits. Set per-request timeouts with `CancellationTokenSource(TimeSpan.FromSeconds(2))`.
- **No connection pooling for SQL / Redis / Mongo / RabbitMQ.** See [connection pooling](./connection-pooling.md).
- **TLS handshake under burst.** Without keep-alive, every request renegotiates TLS — 2–4 RTT plus crypto. Keep-alive is on by default with `HttpClient`; check that it's not being killed by an upstream proxy with `Connection: close`.
- **Buffer copies.** Reading a stream into a `byte[]`, copying to a `MemoryStream`, then to a `Span<byte>`. Each copy is bandwidth you can't get back. Use `System.IO.Pipelines` or `PipeReader.AsStream()` and parse from `ReadOnlySequence<byte>`.
- **Unbounded outbound concurrency.** Fan-out with `Task.WhenAll` over thousands of items. The downstream gets DDoS'd by your own service; you saturate its capacity, not yours. Apply `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`, or a `SemaphoreSlim` gate.

## The fix toolkit

| Symptom | Tool | Notes |
|---|---|---|
| `HttpClient` per call | `IHttpClientFactory` typed clients | One named/typed client = one pool |
| Stale DNS on long-lived handler | `PooledConnectionLifetime` | 5–15 min is typical; default is infinite |
| Slow downstream pinning your pool | `HttpClient.Timeout` + per-request `CancellationToken` + Polly timeout policy | Layered timeouts: per-call < per-pipeline < HttpClient ceiling |
| Retry storms after a downstream blip | Polly retry with **jittered exponential** backoff + circuit breaker | Never retry without jitter — herd effects |
| Unbounded fan-out | `Parallel.ForEachAsync` with `MaxDegreeOfParallelism` | Tune to ~connection-pool size |
| Buffering large bodies | `HttpCompletionOption.ResponseHeadersRead` + `JsonSerializer.DeserializeAsyncEnumerable` | Stream, never `ReadAsStringAsync` for big payloads |
| Parsing throughput | `System.IO.Pipelines` | Decoupled producer/consumer over `ReadOnlySequence<byte>` |
| Retry-causing port exhaustion | `SocketsHttpHandler.PooledConnectionIdleTimeout` | Reuse idle sockets longer; reduce TCP churn |
| Per-tenant rate isolation | Bulkhead per upstream key | Per-tenant `SemaphoreSlim` or per-host Polly bulkhead |

## Worked example — typed client done right

```csharp
// Before: socket exhaustion + stale DNS + no timeout.
public class OrdersClient
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        using var http = new HttpClient(); // disaster: new socket pool per call
        return await http.GetFromJsonAsync<Order>($"https://orders.svc/{id}", ct);
    }
}

// After: pooled, DNS-aware, timeout-bounded.
// Program.cs
builder.Services.AddHttpClient<OrdersClient>(c =>
{
    c.BaseAddress = new Uri("https://orders.svc/");
    c.Timeout = TimeSpan.FromSeconds(5);                 // hard ceiling
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime  = TimeSpan.FromMinutes(5), // DNS rotation
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
    MaxConnectionsPerServer   = 64,                      // bulkhead
})
.AddStandardResilienceHandler();                          // .NET 8+ Polly bundle

public sealed class OrdersClient(HttpClient http)
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        using var perCall = CancellationTokenSource.CreateLinkedTokenSource(ct);
        perCall.CancelAfter(TimeSpan.FromSeconds(2));    // tighter than HttpClient ceiling
        return await http.GetFromJsonAsync<Order>($"{id}", perCall.Token);
    }
}
```

A real service moved from 3% socket-exhaustion errors and 800 ms p99 to zero errors and 35 ms p99 with this template — same hardware, same downstream.

## Worked example — bounded fan-out

```csharp
// Before: 10,000 inflight calls; downstream throttles you, all calls time out together.
var results = await Task.WhenAll(items.Select(i => _client.LookupAsync(i, ct)));

// After: bounded to 32 inflight; downstream stays healthy; total time often lower
// (queueing-theory: high utilization with low concurrency beats high concurrency
//  with throttling-induced retries).
await Parallel.ForEachAsync(items,
    new ParallelOptions { MaxDegreeOfParallelism = 32, CancellationToken = ct },
    async (item, c) =>
    {
        var result = await _client.LookupAsync(item, c);
        // accumulate to a thread-safe sink, e.g. ConcurrentBag or Channel
    });
```

## Is it actually I/O?

Cross-check before optimizing:

- **Idle CPU + low queue + low outbound** = service genuinely has slack; problem is upstream of the service (caller).
- **Idle CPU + low outbound queue + climbing thread count** = thread-pool starvation, not I/O — see [thread contention](./thread-contention.md).
- **High CPU + saturated outbound** = both bound; the costlier one wins. Profile.
- **Latency tracks GC pauses** = GC bound — see [GC pressure](./gc-pressure.md).

**Senior-level gotchas:**
- **`HttpClient` is *not* an `IDisposable` thing you close after each call.** It is a thin wrapper around a handler that owns the socket pool. Disposing it disposes the pool. Use `IHttpClientFactory`.
- **Default outbound concurrency is `int.MaxValue`.** `MaxConnectionsPerServer` is unbounded by default — your service is its own DDoS until you set it.
- **`Connection: close` upstream defeats keep-alive.** Inspect proxy behavior: a misconfigured load balancer can force a fresh TLS handshake per request even though both endpoints support keep-alive.
- **TLS session resumption matters.** If you're terminating TLS in-process, ensure the `SslClientAuthenticationOptions` reuse session tickets. Otherwise burst load = full handshake per call.
- **Timeouts are layered, not nested.** `CancellationToken.CancelAfter` ≤ Polly per-attempt timeout ≤ Polly pipeline timeout ≤ `HttpClient.Timeout` ≤ Kestrel server timeout. Inverting any of these creates phantom failures with confusing error messages.
- **`Polly` retries can amplify saturation.** Three retries × 100 callers × downstream blip = 400 calls instead of 100, all within the same window. Combine with circuit breakers and jitter.
- **`HttpClient` does not refresh DNS** under any default settings prior to setting `PooledConnectionLifetime`. After a deploy of the downstream, your service can keep calling the dead pod for hours.
- **`File.ReadAllBytesAsync` exists**, but the OS file cache often makes the synchronous `File.ReadAllBytes` indistinguishable in practice — *unless* the file is on a slow disk under contention, in which case the sync version blocks a thread-pool worker. Prefer the async overload regardless.
- **Streaming JSON does not always win.** `JsonSerializer.DeserializeAsyncEnumerable<T>` shines on large arrays of records; on a small object response, it's a wash and adds complexity.
- **NIC bandwidth is full duplex.** A 10 Gbps link is 10 Gbps in each direction; saturating both sides is rare but possible — diagnose direction by direction.
- **`SocketsHttpHandler` keeps connections per-host, not per-URL.** All calls to `https://api.svc/...` share the same pool, regardless of path; configure pools by host expectations, not by endpoint.
- **`HTTP/2` multiplexing changes the math.** A single connection can carry many concurrent streams — `MaxConnectionsPerServer` is barely relevant. Look at `http.client.active_requests` instead of pool size.
