# Dapr (Distributed Application Runtime)

_Targets .NET 10 / C# 14 with Dapr 1.14+. See also: [MassTransit](./masstransit.md), [Rebus](./rebus.md), [Wolverine](./wolverine.md), [Kafka](../../../../ARCHITECTURE/EVENTDRIVEN/KAFKA/00-index.md)._

Dapr is a **language-agnostic sidecar** that exposes pluggable distributed-system "building blocks" — pub/sub, state, secrets, bindings, actors, workflows — over a localhost HTTP/gRPC API. Your app talks to `http://localhost:3500/v1.0/...`; Dapr talks to Redis/Kafka/Service Bus/etc. via component YAML. The pitch: write idiomatic .NET, swap brokers/state stores by changing a YAML file, and hand the polyglot fleet next door the same primitives. Pick it when your platform is K8s + multi-language and you want one infra contract; don't pick it for a single-stack .NET app — you're paying sidecar latency and operational complexity for portability you won't use.

## Architecture in one diagram

```
┌─ Pod ─────────────────────────────────┐
│  ┌─────────────┐    ┌──────────────┐  │
│  │ Your App    │←──→│ daprd        │──┼──→ Redis / Kafka / SB / NATS
│  │ (.NET, Go,  │HTTP│ (sidecar,    │  │   (configured by Component YAML)
│  │  Python...) │ gRPC│ ~5-15MB RSS) │  │
│  └─────────────┘    └──────────────┘  │
└───────────────────────────────────────┘
```

Sidecar is invoked via `Dapr.Client` SDK (`DaprClient`) or the ASP.NET Core integration (`Dapr.AspNetCore`) which sugars subscriptions and state.

## Wiring

```csharp
builder.Services.AddDaprClient();                  // IHttpClientFactory-aware DaprClient
builder.Services.AddControllers().AddDapr();       // [Topic], [FromState] model binders

var app = builder.Build();

app.UseRouting();
app.UseCloudEvents();                              // unwrap CloudEvents envelope
app.MapSubscribeHandler();                         // GET /dapr/subscribe — sidecar reads this
app.MapControllers();
app.Run();
```

## Pub/sub: publish

```csharp
public sealed class OrderService(DaprClient dapr)
{
    public Task PlaceAsync(OrderPlaced evt, CancellationToken ct) =>
        dapr.PublishEventAsync(
            pubsubName : "orders-pubsub",            // matches Component metadata.name
            topicName  : "orders.placed",
            data       : evt,
            cancellationToken: ct);
}
```

## Pub/sub: subscribe

```csharp
[ApiController]
public sealed class OrderHandlers(IProjector projector) : ControllerBase
{
    [Topic("orders-pubsub", "orders.placed")]
    [HttpPost("/internal/orders/placed")]
    public async Task<IActionResult> Handle(OrderPlaced evt, CancellationToken ct)
    {
        // idempotency: dedupe by evt.OrderId — Dapr offers at-least-once
        await projector.ApplyAsync(evt, ct);
        return Ok();
    }
}
```

The sidecar polls `/dapr/subscribe` on startup, gets `[{ pubsubname, topic, route }]`, then forwards messages by HTTP POST to `route`. Return 2xx → ack. Return 4xx → drop (poison). Return 5xx → retry per the component's resiliency policy.

## Component YAML (the actual config)

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: orders-pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka-bootstrap:9092"
    - name: consumerGroup
      value: "orders-svc"
    - name: authType
      value: "mtls"
```

This file lives outside your code (typically a K8s `Component` CR). Swap `pubsub.kafka` for `pubsub.redis` and the C# code is unchanged.

## Other building blocks worth knowing

| Block | Use case | C# entry point |
|---|---|---|
| **State** | KV store fronting Redis/Cosmos/Postgres with optimistic concurrency (ETags) | `dapr.SaveStateAsync`, `GetStateAsync` |
| **Bindings** | Input/output to external systems (cron, queues, S3, SMTP) without bespoke code | Input → HTTP route; output → `dapr.InvokeBindingAsync` |
| **Actors** | Single-threaded, addressable, stateful entities — Erlang/Orleans-flavored | `[ActorRoute]`, `IActor` |
| **Workflows** | Durable orchestrations (sagas) — async/await over weeks, replay-based | `Dapr.Workflow` SDK, `WorkflowRuntime` |
| **Secrets** | Uniform `dapr.GetSecretAsync` over Vault, AKV, K8s secrets | `DaprSecretStoreConfigurationProvider` |
| **Configuration** | Live config with subscribe-on-change | `dapr.GetConfiguration`, `SubscribeConfiguration` |

## Workflows (Dapr's saga answer)

```csharp
public sealed class ShipOrderWorkflow : Workflow<OrderPlaced, ShipResult>
{
    public override async Task<ShipResult> RunAsync(WorkflowContext ctx, OrderPlaced input)
    {
        var reservation = await ctx.CallActivityAsync<ReserveResult>(nameof(ReserveStock), input);
        if (!reservation.Ok) return new ShipResult(false, "OOS");

        try
        {
            await ctx.CallActivityAsync(nameof(ChargeCard), input);
            await ctx.CallActivityAsync(nameof(DispatchWarehouse), input);
            return new ShipResult(true, null);
        }
        catch
        {
            await ctx.CallActivityAsync(nameof(ReleaseStock), input);  // compensation
            throw;
        }
    }
}
```

Workflow state is checkpointed in Dapr's state store; replays survive sidecar/pod restarts. Activities must be **deterministic-replay-safe** — no `DateTime.UtcNow`, use `ctx.CurrentUtcDateTime`.

## When to use Dapr

- Polyglot fleet (Go + .NET + Python services) — one infra contract beats per-language SDKs.
- K8s-first, want broker-agnostic code so you can swap Kafka↔Service Bus by config.
- Need both pub/sub *and* virtual actors *and* durable workflows — getting all three from one runtime is rare.

## When NOT to use Dapr

- Single-language .NET shop → MassTransit/Wolverine is closer to the metal and has richer .NET-specific features (state machines, EF outbox, source-gen handlers).
- Serverless / non-K8s deploys (Lambda, App Service Easy mode) — sidecar model fights the platform.
- Latency-sensitive (<1ms) hot paths — sidecar adds ~1–5ms localhost RTT per call. Fine for messaging, painful for sync RPC.

**Senior-level gotchas:**
- **`State` is not a database.** It's a KV facade with ETags for optimistic concurrency, no joins, no transactions across keys (except `ExecuteStateTransactionAsync` and only on stores that support it). Don't put your domain model in it.
- **CloudEvents double-wrapping.** When Dapr forwards a pub/sub message, it wraps the payload in a CloudEvents envelope. If your producer also emits CloudEvents, `data` ends up nested. `app.UseCloudEvents()` unwraps once — stop there. Don't manually re-wrap.
- **Component config drift across environments.** Components live outside the app. Dev runs `pubsub.in-memory`, staging runs `pubsub.redis`, prod runs `pubsub.kafka` — same code, three failure modes. Keep a contract test per env that publishes + subscribes a known message at startup.
- **Sidecar latency floor.** ~1–5ms localhost RTT *per Dapr call*. A workflow with 10 activities = 10 sidecar hops minimum. Profile before betting hot paths on it.
- **mTLS is sidecar-to-sidecar, not user auth.** Dapr secures the pod-to-pod hop. Your app still owns user identity, claims, RBAC. Don't disable JWT validation because "Dapr handles auth."
- **Actor reentrancy is opt-in.** By default a Dapr actor is single-threaded, and a self-call deadlocks. Enable `reentrancy.enabled: true` in the actor config — and then you have to worry about reentrancy bugs again.
- **Workflow versioning is your problem.** A workflow in flight when you deploy a new version replays against new code. Breaking changes (added activity, renamed step) corrupt replay. Use `WorkflowOptions.Version` and keep both versions running until in-flights drain.
- **At-least-once is the default.** Every consumer must be idempotent (dedupe by message ID or business key). Dapr does not give you exactly-once magic; the broker semantics still apply.
- **The sidecar is a separate process** — its crash takes your app's connectivity with it but not the app itself. Probe `/v1.0/healthz` from your readiness probe so K8s sees the pod as not-ready when daprd is unhealthy.
- **Workflows run on the sidecar**, not in your app. Sidecar restart loses in-memory orchestration state — durability is via state-store checkpointing. Pick a state store that's actually durable (Redis without AOF/RDB persistence is *not*).
