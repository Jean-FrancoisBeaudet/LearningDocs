# Service mesh latency

_Targets .NET 10 / C# 14. See also: [Inter-service call optimization](./inter-service-call-optimization.md), [Connection pooling](./connection-pooling.md), [I/O saturation](./io-saturation.md), [Serialization overhead](./serialization-overhead.md), [Backpressure handling](./backpressure-handling.md), [HttpClient and HttpClientFactory](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/httpclient-and-httpclientfactory.md), [gRPC](../../../ASPNETCORE/TOPICS/API_COMMUNICATION/grpc.md)._

A **service mesh** (Istio/Envoy, Linkerd, Consul Connect, OpenServiceMesh) intercepts every inbound and outbound TCP/HTTP connection through a sidecar proxy that lives next to your .NET service in the same pod. You get mTLS, retries, traffic shifting, and trace propagation for free — and you pay for them in **per-hop latency** that the .NET process can neither see nor control directly.

> The .NET application has no idea the mesh is there. That's the design point. It also means most "why is this service slow?" investigations need to look outside the process — at the sidecar, at the mTLS handshake, at the cross-AZ topology. The application is fast; the path between applications has grown.

## The shape of a hop

```
service-A pod                           service-B pod
┌─────────────┐                        ┌─────────────┐
│ .NET app    │                        │ .NET app    │
│   ↓ HTTP    │                        │   ↑ HTTP    │
│ envoy (out) │ ── mTLS network ───►   │ envoy (in)  │
└─────────────┘                        └─────────────┘
```

Per call: 4 syscalls (loopback in/out on both sides) + 2 userspace proxies + mTLS framing. p50 cost is typically 0.5–2 ms in-cluster; p99 is where it bites — sidecar GC pause, config push, Envoy worker thread saturation.

The first call after a sidecar's TLS session expires also pays a full handshake (~2 RTT + key derivation). Sustained high RPS keeps sessions warm; bursty traffic does not.

## What the .NET service controls (and doesn't)

| Concern | .NET controls? | Notes |
|---|---|---|
| TLS handshake to peer | No | Sidecar handles mTLS; app talks plaintext HTTP to localhost sidecar |
| Retries | Sometimes | If the mesh retries *and* `Polly` retries, you get N×M attempts — pick one tier |
| Timeouts | Yes (must be ≤ mesh timeout) | Inverted timeouts produce confusing 503s |
| Connection pool to peer | No | App pool is to the *sidecar*, not the destination |
| Load balancing | No | Mesh does it; app sees one virtual endpoint |
| Trace propagation | Yes — must propagate `traceparent` | Mesh adds spans; app must continue them |
| Cancellation propagation | Mostly yes | gRPC deadlines and `CancellationToken` over HTTP/2 propagate through Envoy |
| mTLS cert rotation | No | Sidecar handles via SDS; app sees zero downtime if configured right |

## Pool semantics shift with a sidecar

Without a mesh, your `SocketsHttpHandler.MaxConnectionsPerServer = 64` is "to `orders.svc`." With a mesh, it's "to `localhost:15001` (the outbound Envoy)." The mesh is the actual destination of every TCP connection your service opens for outbound traffic. The pool to *each real backend* is owned by Envoy.

Consequences:

- `MaxConnectionsPerServer` is now sized against your sidecar's worker capacity, not the upstream. A reasonable default is 32–128 — small enough that the sidecar can multiplex efficiently, large enough that it's not the bottleneck.
- Envoy's upstream connection pool is configured at the mesh level (`circuitBreakers.connectionPool.http.http1MaxPendingRequests`, etc). If your service is rejected with `503 UO` (Upstream Overflow), the mesh-level circuit breaker tripped — not yours.
- HTTP/2 multiplexing through the sidecar is bidirectional but with separate stream limits per leg (app↔sidecar, sidecar↔upstream). The slower leg is the bottleneck.

For HTTP/2 to the sidecar:

```csharp
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    EnableMultipleHttp2Connections = true,
    PooledConnectionLifetime = TimeSpan.FromMinutes(5),
    MaxConnectionsPerServer = 64,
})
```

`PooledConnectionLifetime` matters even on localhost: when Envoy reloads config (live update of routes / weights / endpoints), it tears down upstream connections. The .NET app's pool to localhost can survive a reload, but DNS-cached pod IPs don't apply because everything goes through localhost. Set `PooledConnectionLifetime` for symmetry with non-mesh paths and predictability — *not* for DNS rotation, which doesn't apply here.

## Timeouts: the inversion trap

Mesh and app timeouts must be ordered:

```
per-attempt timeout (.NET) <  mesh per-attempt timeout < ingress timeout
       2 s                          5 s                       30 s
```

If the .NET service has `HttpClient.Timeout = 30s` and Envoy has `route.timeout = 5s`, the mesh closes the connection at 5s. The app sees an `IOException: The response ended prematurely` at 5.001s — not a `TimeoutException`. The error message is misleading, the trace says "upstream sent half a response," and you spend an afternoon blaming the wrong layer.

Rule: app timeout is always **strictly less** than the mesh's. The mesh is the safety net, not the contract.

## Envoy 503 response flags decoder

When the mesh replies 503, the response flag in Envoy access logs tells you why. The `RESPONSE_FLAGS` in the standard format:

| Flag | Meaning | What to do |
|---|---|---|
| `UH` | No healthy upstream | Mesh's endpoint discovery has zero healthy pods. Check readiness probes. |
| `UO` | Upstream overflow (circuit breaker) | `connectionPool` capacity exceeded. Raise mesh limits or scale down concurrency. |
| `UR` | Upstream remote reset | Backend `Connection: close` or RST. Backend is choking; check it. |
| `URX` | Upstream retry limit | Mesh exhausted retries. Often pairs with backend slowness. |
| `UF` | Upstream connection failure | TCP connect failed. DNS, network, MTU, or backend down. |
| `UC` | Upstream connection termination | Backend closed mid-response. Look at backend timeouts. |
| `LH` | Local health check failed | The mesh's own probe to the upstream is failing. |
| `NR` | No route configured | VirtualService / DestinationRule misconfig — usually a deploy bug. |
| `DC` | Downstream connection termination | Caller hung up. Often a client cancellation. |

**Decode the flag first**; then look at app logs. 90% of "the mesh broke our service" investigations are answered by the flag.

## Retries: pick one tier

Mesh retries (`route.retryPolicy.attempts: 3`) and Polly retries (`Retry.MaxRetryAttempts = 3`) compose multiplicatively. Three at each layer = 9 attempts per failed call, and they fan out across pods. Under a downstream brownout, this is how a single-pod failure becomes a fleet-wide retry storm.

Default: **let the mesh retry**, disable client-side retries (or set `MaxRetryAttempts = 0`). The mesh sees more system state (which pod is healthy, which subset is preferred, whether the breaker is open) and its retries don't compound.

Exception: when the *target* is the mesh-managed service but the *failure* is in the .NET serialization layer, the mesh can't help. For that, layer a single client-side retry inside the mesh-level retry budget (e.g. mesh = 3, client = 1 with strict retry budget).

## Locality and topology routing

Cross-AZ adds ~1 ms RTT; cross-region adds 30–80 ms. The mesh decides this:

```yaml
# Istio DestinationRule
trafficPolicy:
  loadBalancer:
    localityLbSetting:
      enabled: true
      distribute:
      - from: "us-east-1/*"
        to:
          "us-east-1/*": 80
          "us-east-2/*": 20
```

The .NET service has no awareness. But the *symptom* of bad locality routing is "p50 fine in test, p50 doubled in prod with the same RPS" — your test environment is single-AZ, prod is three-AZ, and the mesh distributes across them.

Diagnose by looking at trace span attributes: `peer.service` and `network.peer.address`. If the IP belongs to a different AZ than the caller's pod, the mesh is routing cross-AZ.

## Worked example: app is 8 ms, wall-clock is 60 ms

A `/orders/{id}` endpoint shows:

```
client                                  │ p50: 60 ms │
│
└─ ingress envoy ──────► sidecar A      │ 8 ms in app trace
   │                  ┌─ sidecar B ─►   │
   └─ DNS resolve     │  app B          │
   │                  └─ database       │
```

App-side spans show 8 ms total. The missing 52 ms is:

- 2× sidecar hop (out + in): ~3 ms
- mTLS handshake on a cold connection: ~6 ms
- Cross-AZ RTT × 2 (client-AZ → app-AZ → DB-AZ): ~4 ms
- Ingress controller TLS termination + L7 routing: ~5 ms
- DNS lookup for first call after pod start: ~30 ms (cold cache)

Fixes, in order of impact:

1. Pre-warm DNS at pod startup (resolve all known peers once).
2. Locality-pinning DestinationRule so client and app share an AZ.
3. Connection pool warming so the first request after deploy doesn't pay handshake cost.
4. Ingress controller HTTP/2 to upstream (often defaults to HTTP/1).

The .NET app didn't change. The wall-clock dropped from 60 ms to 12 ms.

## Diagnosing: where the time actually goes

| Question | Tool |
|---|---|
| "Is it the mesh or my app?" | Compare app-side OpenTelemetry span duration to the request total. The gap is the mesh. |
| "Why 503?" | Envoy access log `%RESPONSE_FLAGS%` |
| "Cold connection or steady state?" | Compare p50 vs p99; if p99 ≫ p50 with low RPS, cold connections / handshake. |
| "Cross-AZ?" | Trace span `network.peer.address` — resolve to AZ via cloud provider metadata |
| "Sidecar starved?" | Envoy admin endpoint `/stats` → `cluster.<name>.upstream_rq_pending_overflow` |
| "Config push pause?" | `pilot_xds_pushes` metric; correlate timing with tail-latency spikes |
| "mTLS handshake cost?" | Compare same call inside and outside the mesh (port-forward bypass) |

## Bypassing the mesh: when to do it

Sometimes the mesh is the wrong tool. Common bypasses:

- **Localhost-only calls** between containers in the same pod (sidecar pattern within the app — e.g. a metrics scraper). Use UDS or `127.0.0.1` directly; configure mesh to exclude these.
- **Liveness/readiness probes** — never go through mesh; use `httpGet` on app port directly. Otherwise the mesh's outlier detection can mark all pods unhealthy when *itself* is misconfigured.
- **High-volume internal traffic** where mesh overhead exceeds the value (e.g. cache replication). Evaluate a non-mesh path; mesh is not always the right cost trade-off.

In Istio, exclusion is via `traffic.sidecar.istio.io/excludeOutboundPorts` annotation on the pod, or `Sidecar` resource scoping. In Linkerd, `linkerd.io/inject: disabled` on a per-pod basis.

## Anti-patterns

| Anti-pattern | Failure mode |
|---|---|
| App timeout > mesh timeout | App sees `IOException` instead of `TimeoutException`; misleading errors |
| Polly retries on top of mesh retries | Multiplicative retry storms on backend brownout |
| Liveness probe through mesh | Mesh outage flags all pods unready; cascades to outage |
| Health endpoint that calls downstream services | Cascading unhealthiness when the mesh-managed downstream blips |
| `MaxConnectionsPerServer = int.MaxValue` to localhost sidecar | Sidecar worker saturation; UO 503s |
| Trusting `peer.address` in app code (it's always the sidecar) | Auditing fails; use mesh-injected `X-Forwarded-For` / mTLS SAN |
| Custom retry of `503 UH` | UH means no healthy upstream — retrying doesn't make one appear |
| Static cert pinning in the .NET app | Mesh rotates certs constantly; pinning breaks within the cert TTL |
| Disabling mTLS for "performance" without measuring | Handshake amortizes well at steady state; the 1–2% CPU cost rarely justifies the risk |
| App ignores `traceparent` from inbound | Trace tree breaks at the app; you can't follow the request through the mesh |

## Senior-level gotchas

- **Sidecar config-push is fleet-wide and instantaneous.** Istiod pushes new config to every Envoy in the mesh. During the push, each Envoy briefly pauses (~50–200 ms) to apply it. The result is a synchronized fleet-wide tail latency spike on every config change. Frequent config changes (canary deploys, weight shifts) compound this — alert on `pilot_xds_pushes` rate.
- **mTLS handshake cost is per-connection, not per-request.** With a warm pool, mTLS is free at steady state. With pool churn (low traffic, long idle, frequent restarts), every cold connection pays it. `PooledConnectionIdleTimeout` interacts here — too short and you handshake constantly.
- **Envoy's connection pool to upstream uses HTTP/2 by default *between Envoys*, regardless of what your app sent.** App talks HTTP/1.1 to its sidecar; sidecar talks HTTP/2 to peer sidecar; peer sidecar talks whatever-the-target-prefers to peer app. You don't directly control the inter-Envoy protocol — that's a `DestinationRule` decision.
- **gRPC deadlines propagate as `grpc-timeout` headers.** Envoy honors them and aborts upstream calls when the deadline passes. `HttpClient.Timeout` does not propagate this way; only the gRPC-level deadline does. For deadline propagation across mesh hops, use gRPC, not HTTP+JSON.
- **The mesh adds an Envoy span between app spans.** A trace from .NET shows app→app, but the actual hops are app→envoyOut→envoyIn→app. Without mesh-side OpenTelemetry, you see a "missing 4 ms" gap; with it, you see the gap explained.
- **`kubectl exec ... curl` bypasses the mesh.** That's why curl-from-inside-the-pod works while the app's HTTP call to the same URL fails. Useful for diagnosis, dangerous for "it works in test."
- **mTLS and HTTP probe headers don't compose well.** A `httpGet` probe on the app port works because it bypasses mesh; a probe through the mesh requires permissive mTLS or a probe identity. Misconfig here is a common source of 503 LH (local health check failed).
- **`X-Envoy-Original-Path` and friends are mesh-injected and trusted by Envoy.** A .NET app that reads them must be sure the request actually came through the mesh — otherwise an external client can spoof. The standard pattern: only trust `X-Forwarded-*` from the mesh, not from arbitrary callers.
- **Outlier detection is non-deterministic.** Envoy ejects pods that fail too often; the threshold is a moving window. Pods can flap in/out of the load balancer pool during a partial outage, making errors transient and hard to reproduce.
- **The mesh can't help with serialization-cost tail latency.** If your app's p99 is JSON deserialization on a 5 MB response, the sidecar didn't cause it and can't fix it. Disambiguate by comparing app-span time to total time — if app time ≈ total time, mesh is innocent.
- **Connection draining on pod shutdown is mesh-managed.** Envoy stops accepting new requests, drains existing ones, then exits. The .NET app needs `IHostApplicationLifetime` shutdown hooks that finish in-flight work — if it exits before drain completes, in-flight requests get `UC` (upstream connection termination).
- **`localhost:15001` is the outbound interception port** in Istio; everything goes through it via iptables redirection. If your .NET app makes a UDP call (DNS aside), it's *not* intercepted — UDP-based protocols (DNS, QUIC/HTTP3) bypass the mesh. HTTP/3 *to* mesh-managed services therefore degrades to HTTP/2 transparently.
- **Sidecar resource limits are part of your latency budget.** A sidecar capped at 100m CPU starves under burst — its event loop falls behind, and you see tail-latency spikes that look like network issues. Right-size the sidecar (typical: 200m–500m CPU) before optimizing the app.
- **Cross-region calls through the mesh add the mesh hop *plus* the WAN hop.** Locality routing keeps calls in-region; without it, you can pay 30–80 ms WAN RTT in addition to mesh overhead. Always check `localityLbSetting` on cross-region deployments.
- **The mesh trace context survives, the .NET cancellation token doesn't — across HTTP/JSON.** A cancelled `CancellationToken` aborts the local `HttpClient` send; the sidecar gets a stream reset and propagates that as a TCP RST or HTTP/2 `RST_STREAM` to upstream. Most upstreams treat that as cancellation correctly, but custom servers may not. gRPC handles this cleanly.
- **mTLS doesn't replace authn/authz at the application layer.** It proves the *connection* came from a workload identity; it doesn't say anything about the *user*. Don't swap your app-level auth for mesh mTLS; layer them.
