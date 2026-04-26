# Rebus

_Targets .NET 10 / C# 14 with Rebus 8+. See also: [MassTransit](./masstransit.md), [Wolverine](./wolverine.md), [Dapr](./dapr.md)._

Rebus (Mogens Heller Grabe, MIT) is the **lean** .NET service bus — a small, composable core with one NuGet per transport, persistence, and serializer. If MassTransit is "service bus + framework," Rebus is "service bus, period." Same building blocks (commands, events, sagas, deferred delivery), much smaller surface, no DSLs. Pick it when you want messaging without buying into a framework, when team size is small, or when you'd rather read 5,000 lines of source than 50,000. Don't pick it if you want a state-machine DSL, the largest community, or contract-evolution tooling — MassTransit beats it there.

## Wiring

```csharp
builder.Services.AddRebus((configure, sp) => configure
    .Logging(l => l.Serilog())
    .Transport(t => t.UseRabbitMq(
        builder.Configuration.GetConnectionString("Rabbit"), inputQueueName: "orders-svc"))
    .Routing(r => r.TypeBased()
        .Map<ChargeCard>("payments-svc")            // command → owning queue
        .Map<ShipOrder>("shipping-svc"))
    .Subscriptions(s => s.StoreInPostgres(
        builder.Configuration.GetConnectionString("Db"), "rebus_subscriptions"))
    .Sagas(s => s.StoreInPostgres(
        builder.Configuration.GetConnectionString("Db"), "rebus_sagas", "rebus_saga_index"))
    .Outbox(o => o.StoreInPostgres(
        builder.Configuration.GetConnectionString("Db"), "rebus_outbox"))
    .Options(o =>
    {
        o.SetMaxParallelism(20);                     // concurrent in-flight messages
        o.SetNumberOfWorkers(4);                     // worker tasks
        o.RetryStrategy(maxDeliveryAttempts: 5);
    }));

builder.Services.AutoRegisterHandlersFromAssemblyOf<Program>();
```

The `Configure.With` chain reads as a recipe — every concern (logging, transport, routing, subs, sagas, outbox) is opt-in via its own NuGet (`Rebus.RabbitMq`, `Rebus.PostgreSql`, `Rebus.SerilogLogging`, ...).

## Three communication patterns

| Pattern | API | Routing |
|---|---|---|
| **Send** (command) | `bus.Send(new ChargeCard(...))` | Type-based map (`Routing(r => r.TypeBased().Map<T>(queue))`) |
| **Publish** (event) | `bus.Publish(new OrderPlaced(...))` | Subscriptions store (`Subscriptions(s => s.StoreInPostgres(...))`) |
| **Reply** | `await bus.Reply(response)` | Reads `ReturnAddress` header from incoming message |

```csharp
public sealed class OrderHandler(IBus bus, IOrders orders)
    : IHandleMessages<PlaceOrder>, IHandleMessages<OrderPlaced>
{
    public async Task Handle(PlaceOrder cmd)
    {
        var order = await orders.PlaceAsync(cmd, MessageContext.Current!.IncomingStepContext.Load<CancellationToken>());
        await bus.Publish(new OrderPlaced(order.Id, order.Total));
    }

    public Task Handle(OrderPlaced evt) =>
        // separate subscriber concern — projection, audit, notifications
        Task.CompletedTask;
}
```

`MessageContext.Current` exposes headers, message ID, and the transaction context. Cancellation tokens flow through `IncomingStepContext` items, or wire a `CancellationToken` parameter via a custom incoming step.

## Sagas

```csharp
public sealed class OrderSagaData : ISagaData
{
    public Guid Id { get; set; }
    public int Revision { get; set; }
    public Guid OrderId { get; set; }
    public bool Charged { get; set; }
    public bool Shipped { get; set; }
}

public sealed class OrderSaga(IBus bus)
    : Saga<OrderSagaData>,
      IAmInitiatedBy<OrderPlaced>,
      IHandleMessages<PaymentReceived>,
      IHandleMessages<Shipped>
{
    protected override void CorrelateMessages(ICorrelationConfig<OrderSagaData> c)
    {
        c.Correlate<OrderPlaced>     (m => m.OrderId, d => d.OrderId);
        c.Correlate<PaymentReceived> (m => m.OrderId, d => d.OrderId);
        c.Correlate<Shipped>         (m => m.OrderId, d => d.OrderId);
    }

    public async Task Handle(OrderPlaced m)
    {
        Data.OrderId = m.OrderId;
        await bus.Send(new ChargeCard(m.OrderId, m.Total));
    }

    public async Task Handle(PaymentReceived m)
    {
        Data.Charged = true;
        await bus.Send(new ShipOrder(m.OrderId));
    }

    public Task Handle(Shipped m)
    {
        Data.Shipped = true;
        MarkAsComplete();
        return Task.CompletedTask;
    }
}
```

No state-machine DSL — just handler methods plus `MarkAsComplete()`. Simpler than MassTransit; rots faster as the saga grows. Limit yourself to ~5 events; beyond that, the conditional branching is hard to reason about.

## Deferred delivery

```csharp
await bus.Defer(TimeSpan.FromMinutes(15), new RemindUser(userId));
```

Built-in across all transports — Rebus uses transport-native scheduling where available (Azure Service Bus, RabbitMQ delayed exchange) or a SQL-backed timeout manager for transports without it. No extra config beyond `Timeouts(t => t.StoreInPostgres(...))` for SQL-based transports.

## Outbox

`Outbox(o => o.StoreInPostgres(...))` writes outgoing messages to a SQL table inside your `IDbConnection`/`DbContext` transaction; a worker drains them. Pair with the unit-of-work middleware to enroll the same connection in the message handler's transaction. This gets you transactional consistency without ceremony — same pattern as MassTransit's outbox, less wiring.

## When to use Rebus

- Small/medium app where MassTransit's surface feels heavy.
- You want to read the source when something breaks.
- Multi-transport with simple needs (Send/Publish/Reply, sagas, scheduling).
- MIT-only license requirement (no commercial v9 anxiety).

## When NOT to use Rebus

- State-machine sagas with many events → [MassTransit](./masstransit.md)'s DSL pays off.
- Want one tool for in-proc + distributed → [Wolverine](./wolverine.md).
- Polyglot fleet → [Dapr](./dapr.md).
- You need a giant integration ecosystem (every Quartz scheduler, every observability adapter) — MassTransit's community is bigger.

**Senior-level gotchas:**
- **`IBus` is a singleton, threadsafe; handlers run with a per-message DI scope.** Don't store request-scoped state on the bus. Resolve scoped services inside the handler via DI — `Rebus.ServiceProvider` wires this for you.
- **Type-based routing requires shared message-contract assemblies.** Producer and consumer must agree on the CLR type's full name. For independent service evolution, use topic-based pub/sub or a shared `Contracts` assembly with strict semver. Renaming a record namespace silently breaks routing.
- **Saga conflicts are revision-checked.** `ISagaData.Revision` is the optimistic-concurrency marker. Two simultaneous deliveries for the same correlation ID → one wins, one retries. If retries exhaust (`maxDeliveryAttempts`), the message lands in the error queue. Tune `maxDeliveryAttempts` to absorb expected contention; alert on the error queue.
- **Second-level retries.** Standard retry pipeline retries within the worker. After exhaustion, the message goes to `error` (a dead-letter). Configure a custom retry pipeline (`o.Decorate<IPipeline>...`) for delayed retry or send to a `manual_review` queue instead of dropping into `error`.
- **No built-in saga DSL = handler `if/else` rot.** A 7-state saga in Rebus is a maintenance trap. If you find yourself writing `if (Data.A && !Data.B && Data.C)` patterns, you've outgrown Rebus's saga model — either model the state explicitly with an enum + switch, or migrate to MassTransit's state machine.
- **Subscription store is durable, transport may not be.** With RabbitMQ, subscriptions live in your SQL table (durable across restarts). With Azure Service Bus, subscriptions are native topic subscriptions and must be created on the broker — Rebus auto-creates on first publish. Don't mix the two mental models.
- **`Defer` precision is bounded by the timeout-manager poll interval** (default 1 second). It's "deliver no earlier than X," not a millisecond timer.
- **Transactional outbox needs the same connection.** If your app code uses `DbContext` and your outbox uses `IDbConnection` from a different pool, the writes are atomic but only by accident. Use a single connection enrolled in the unit-of-work.
- **Polymorphic message handling**: a handler implementing `IHandleMessages<BaseEvent>` does NOT receive `DerivedEvent` unless you configure polymorphic dispatch or register both. Be explicit.
- **`AutoRegisterHandlersFromAssemblyOf<>` registers transient.** Heavy handlers re-instantiated per message. For pooled deps, register them yourself with the lifetime you need.
- **One-way clients are a real thing.** `Configure.OneWayClient()` builds a publish/send-only bus with no input queue — use it from short-lived processes (a CLI, a one-shot job) so you don't pollute the broker with throwaway queues.
