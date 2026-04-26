# HotChocolate Subscriptions

_Targets .NET 10 / C# 14 with HotChocolate v14+. See also: [GraphQL HotChocolate](../API_COMMUNICATION/graphql-hotchocolate.md), [WebSockets](./websockets.md). For raw RPC fan-out, the future `signalr.md`._

A GraphQL **subscription** is a long-running operation that pushes events to the client over a WebSocket. HotChocolate implements it as a regular schema field whose return type is a stream — under the hood, the engine subscribes to a topic, awaits messages, and streams each through the resolver back to the client. Pick HotChocolate Subscriptions when you already have a HotChocolate schema and want subscriptions in the same query language as your queries and mutations (one client library, one auth flow, typed payloads). Skip it for raw RPC fan-out (SignalR is more direct), or when consumers want broker-native delivery semantics (Kafka/RabbitMQ → consume the broker directly, don't proxy through GraphQL).

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddInMemorySubscriptions();                          // dev / single instance only

// production: swap to Redis backplane
// .AddRedisSubscriptions(_ => ConnectionMultiplexer.Connect(redisCs));

var app = builder.Build();

app.UseWebSockets();                                       // ⚠ MUST come before MapGraphQL
app.MapGraphQL();                                          // POST /graphql + WS /graphql
app.Run();
```

The HotChocolate WS endpoint speaks both `graphql-transport-ws` (preferred, Apollo's modern protocol) and the legacy `graphql-ws` (Apollo's older one) — clients negotiate via `Sec-WebSocket-Protocol`. Modern Apollo Client and urql default to `graphql-transport-ws`; double-check older clients.

## Defining subscription fields

```csharp
public sealed class Subscription
{
    // Static topic — all subscribers share the same channel
    [Subscribe]
    [Topic("orders.created")]
    public Order OrderCreated([EventMessage] Order order) => order;

    // Topic templated from arguments — one channel per customer
    [Subscribe]
    [Topic($"{nameof(OnOrderForCustomer)}.{{customerId}}")]
    public Order OnOrderForCustomer(long customerId, [EventMessage] Order order) => order;
}
```

`[Subscribe]` marks the resolver as a subscription field. `[Topic]` declares the channel the engine subscribes to. `[EventMessage]` injects the published payload.

## Publishing events

```csharp
public sealed class OrderService(ITopicEventSender sender)
{
    public async Task<Order> PlaceAsync(NewOrder input, CancellationToken ct)
    {
        var order = /* ... persist ... */;

        await sender.SendAsync("orders.created", order, ct);
        await sender.SendAsync($"OnOrderForCustomer.{order.CustomerId}", order, ct);

        return order;
    }
}
```

`ITopicEventSender` is a singleton — capture it anywhere (controllers, message handlers, background services). The corresponding `ITopicEventReceiver` is for **dynamic** subscription topics (next section).

## Dynamic subscription topics

When the topic depends on per-subscriber state (e.g., the authenticated user id), define the subscription as an `IAsyncEnumerable<T>` resolver that builds the topic at subscribe-time:

```csharp
public sealed class Subscription
{
    public async IAsyncEnumerable<Notification> OnNotification(
        [Service] ITopicEventReceiver receiver,
        ClaimsPrincipal user,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var topic = $"notifications.{user.GetUserId()}";
        var stream = await receiver.SubscribeAsync<Notification>(topic, ct);

        await foreach (var msg in stream.ReadEventsAsync().WithCancellation(ct))
            yield return msg;
    }
}
```

The engine treats each `yield return` as a subscription delivery. Cancellation flows from the WS disconnect into `ct` and unwinds the iterator.

## Server-side filtering

Don't push events the subscriber would ignore. Filter in the resolver before yielding:

```csharp
[Subscribe]
[Topic("orders.created")]
public Order? OrderCreatedFiltered(
    [EventMessage] Order order,
    long? minTotal)
    => minTotal is null || order.Total >= minTotal ? order : null;
```

Returning `null` skips the event without canceling the subscription. Filter heavy logic out at the **publisher** when possible — running it per subscriber on every event is the GraphQL-subscriptions equivalent of N+1.

## Authorization

```csharp
builder.Services
    .AddGraphQLServer()
    .AddAuthorization();                                   // wires AspNetCore auth into HC

public sealed class Subscription
{
    [Authorize(Policy = "ReadOrders")]
    [Subscribe]
    [Topic("orders.created")]
    public Order OrderCreated([EventMessage] Order order) => order;
}
```

Subscription auth runs on the **`connection_init`** message, not on the HTTP upgrade. Browsers can't send an `Authorization` header on the WS handshake — the token rides in the init payload, validated by an `ISocketSessionInterceptor`:

```csharp
public sealed class JwtConnectInterceptor(IJwtValidator jwt) : DefaultSocketSessionInterceptor
{
    public override async ValueTask<ConnectionStatus> OnConnectAsync(
        ISocketSession session,
        IOperationMessagePayload connectionInitMessage,
        CancellationToken ct)
    {
        var payload = connectionInitMessage.As<Dictionary<string, object?>>();
        if (payload?.TryGetValue("authToken", out var token) is not true || token is not string s)
            return ConnectionStatus.Reject("missing token");

        var principal = await jwt.ValidateAsync(s, ct);
        if (principal is null) return ConnectionStatus.Reject("invalid token");

        session.Connection.HttpContext.User = principal;
        return ConnectionStatus.Accept();
    }
}

builder.Services
    .AddGraphQLServer()
    .AddSocketSessionInterceptor<JwtConnectInterceptor>();
```

Token refresh **mid-connection** isn't supported by the protocol — when the token expires, the client must reconnect with a new one.

## Operating

```csharp
builder.Services
    .AddGraphQLServer()
    .AddMaxExecutionDepthRule(8)                           // prevents abusive subscription queries
    .ModifyRequestOptions(o => o.IncludeExceptionDetails = builder.Environment.IsDevelopment())
    .AddInstrumentation();                                 // OpenTelemetry — emits subscription spans

builder.Services.AddHealthChecks()
    .AddRedis(redisCs, name: "redis-subscriptions");      // when using Redis backplane
```

The Nitro IDE (formerly Banana Cake Pop, served at `/graphql` in dev) speaks subscriptions over WS — the easiest way to verify a topic is firing without standing up a real client.

## Scale-out semantics

| Backplane | Cross-instance fan-out | Late-joiner backfill | Delivery |
|---|---|---|---|
| `AddInMemorySubscriptions` | ❌ — only the publishing pod's subscribers receive | ❌ | At-most-once |
| `AddRedisSubscriptions` | ✅ — Redis pub/sub | ❌ — pub/sub is fire-and-forget | At-most-once |
| Custom `ITopicEventSender` over Kafka / Service Bus | ✅ | Depends on broker (Kafka: yes; SB topic w/ subscription: yes) | Up to broker |

**Redis pub/sub does not buffer.** A subscriber that connects 10 ms after a publish gets nothing. If "missed events while disconnected" matters, pair the subscription with a `eventsSince(timestamp: ...)` query the client calls on reconnect — subscriptions for the live stream, queries for the catch-up window.

## Comparison — Subscriptions vs SignalR vs raw WS

| Need | Pick |
|---|---|
| Already on HotChocolate, typed payloads, schema-discoverable streams | **HotChocolate Subscriptions** |
| Server-push to many clients with reconnect, broadcast, fallback transport | **SignalR** |
| Custom wire protocol, no GraphQL/RPC abstraction | **[Raw WebSockets](./websockets.md)** |
| GraphQL clients (Apollo / urql / Relay) already in the stack | **HotChocolate Subscriptions** |
| Heavy fan-out (millions of clients), broker-native semantics | Direct broker (Kafka, MQTT, Service Bus) |

## Senior-level gotchas

- **`app.UseWebSockets()` MUST come before `MapGraphQL()`.** Wrong order: the upgrade silently 404s and the client retries forever. The query/mutation HTTP path keeps working, which makes the bug invisible until someone tries a `subscription` in dev.
- **In-memory subscriptions don't fan out across replicas.** Publish on pod A, subscribe on pod B — silence. The bug only shows up after horizontal scale and is the single most common production trap. Switch to `AddRedisSubscriptions` before going multi-instance.
- **`graphql-transport-ws` vs `graphql-ws`**: HotChocolate accepts both, but Apollo Client v3.7+ defaults to `graphql-transport-ws` while older clients use the legacy one. If subscriptions "don't connect," check the `Sec-WebSocket-Protocol` the client is sending.
- **Subscription auth fires on `connection_init`, not on every operation.** Token refresh mid-connection requires reconnecting; design for that on the client.
- **Filter resolvers run per subscriber, per event.** Heavy filtering belongs at the publisher (one cheap topic per shape) — otherwise you do the same work N times on every publish.
- **Capturing `ITopicEventReceiver` in a singleton breaks subscription scoping.** It's request-scoped because each subscription owns its own subscription handle. Inject via `[Service]` on the resolver.
- **Redis pub/sub is fire-and-forget.** Pair subscriptions with a "since" query for replay if missed-while-disconnected matters.
- **`MapGraphQL()` mounts at the same path for HTTP and WS.** Reverse proxies that strip `Connection: Upgrade` (Cloudflare WS not enabled, App Gateway probe misconfigured) will let queries through and break subscriptions silently.
- **A long subscription lifetime holds the resolver's DI scope open.** Avoid scoped `DbContext` capture inside the resolver — use `IDbContextFactory<T>` and create per-event scopes, like with [DataLoader](../API_COMMUNICATION/graphql-hotchocolate.md#the-n1-problem-and-dataloader).
- **`OnNotification` style `IAsyncEnumerable<T>` resolvers**: forgetting `[EnumeratorCancellation]` on the `CancellationToken` parameter means the iterator never observes WS disconnects — connections stack up until the pod runs out of file descriptors.
