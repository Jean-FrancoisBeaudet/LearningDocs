# MassTransit

_Targets .NET 10 / C# 14 with MassTransit 8+. See also: [Rebus](./rebus.md), [Wolverine](./wolverine.md), [Dapr](./dapr.md), [Kafka](../../../../ARCHITECTURE/EVENTDRIVEN/KAFKA/00-index.md), [gRPC](../API_COMMUNICATION/grpc.md)._

MassTransit (Loosely Coupled, Inc.) is the de-facto .NET service-bus library — broker abstraction over RabbitMQ, Azure Service Bus, Amazon SQS/SNS, and (via the Rider sidecar) Kafka. Its core value isn't "send a message"; it's the **operational scaffolding** around it: state-machine sagas, retry/redelivery middleware pipeline, request-response, scheduling, transactional outbox, automatic broker topology, and dead-letter handling. Pick it for any non-trivial distributed .NET system that needs sagas or contract-evolution discipline. Don't pick it as "MediatR for the network" — the broker topology, contract types, and saga storage are real operational surface.

> **License note:** MassTransit v9 (announced 2025) moves to a commercial license model; v8 remains MIT. Pin to v8.x and read the v9 license terms before upgrading. This is the single biggest architectural decision in this folder right now.

## Wiring (RabbitMQ)

```csharp
builder.Services.AddMassTransit(x =>
{
    x.SetKebabCaseEndpointNameFormatter();
    x.AddConsumers(typeof(Program).Assembly);

    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UseSqlServer();
        o.UseBusOutbox();                           // transactional publish inside SaveChangesAsync
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("Rabbit"));
        cfg.ConfigureEndpoints(ctx);                // auto endpoint per consumer

        cfg.UseMessageRetry(r => r.Exponential(
            retryLimit   : 5,
            minInterval  : TimeSpan.FromSeconds(1),
            maxInterval  : TimeSpan.FromMinutes(1),
            intervalDelta: TimeSpan.FromSeconds(2)));
    });
});
```

`ConfigureEndpoints` creates one queue per consumer plus the exchange topology for each declared message type. **Lock topology in prod** (see gotchas).

## Three communication patterns

| Pattern | API | Topology |
|---|---|---|
| **Publish** (event, fan-out) | `IPublishEndpoint.Publish<OrderPlaced>(...)` | Topic exchange → N consumers' queues |
| **Send** (command, point-to-point) | `ISendEndpointProvider.GetSendEndpoint(uri)` | Direct queue |
| **Request-Response** | `IRequestClient<TReq>.GetResponse<TResp>` | Temporary reply queue, correlation by `RequestId` |

```csharp
public sealed class OrderController(IPublishEndpoint bus) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place(OrderDto dto, CancellationToken ct)
    {
        await bus.Publish(new OrderPlaced(dto.Id, dto.Total), ct);
        return Accepted();
    }
}

public sealed class StockReserver(IStock stock) : IConsumer<OrderPlaced>
{
    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        await stock.ReserveAsync(ctx.Message.OrderId, ctx.CancellationToken);
        // ConsumeContext IS an IPublishEndpoint — published messages flow through the outbox
        await ctx.Publish(new StockReserved(ctx.Message.OrderId));
    }
}
```

## Sagas (state-machine)

`MassTransitStateMachine<TState>` (Automatonymous, merged into core in v8) defines a state machine; `TState : SagaStateMachineInstance` is the per-correlation-id record persisted in EF/Mongo/Redis/Marten.

```csharp
public sealed class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = default!;
    public DateTime Placed { get; set; }
}

public sealed class OrderSaga : MassTransitStateMachine<OrderState>
{
    public State AwaitingPayment { get; private set; } = default!;
    public State Shipped         { get; private set; } = default!;

    public Event<OrderPlaced>     Placed       { get; private set; } = default!;
    public Event<PaymentReceived> PaymentRecvd { get; private set; } = default!;

    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);
        Event(() => Placed,       e => e.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentRecvd, e => e.CorrelateById(c => c.Message.OrderId));

        Initially(
            When(Placed)
                .Then(c => c.Saga.Placed = DateTime.UtcNow)
                .Publish(c => new ChargeRequested(c.Saga.CorrelationId))
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentRecvd)
                .Publish(c => new ShipRequested(c.Saga.CorrelationId))
                .TransitionTo(Shipped));
    }
}
```

Register with `x.AddSagaStateMachine<OrderSaga, OrderState>().EntityFrameworkRepository(...)`.

## Outbox

Without an outbox, `Publish` outside a `ConsumeContext` hits the broker immediately — your DB rollback won't undo it. `UseBusOutbox()` (above) intercepts publishes inside an EF `SaveChangesAsync` and stores them in `OutboxMessage`; a hosted service drains the outbox to the broker. Atomic with the DB transaction.

For incoming messages, `UseInbox` deduplicates by `MessageId` so retries don't re-execute side effects.

## Retries and redelivery

`UseMessageRetry` is **in-process** (same delivery, paused with `Task.Delay`). For long retry windows or infra outages, use **redelivery** (`UseScheduledRedelivery`) — the message is re-enqueued by the broker scheduler, freeing the consumer thread. Combine: short in-proc exponential retry → scheduled redelivery → `_error` queue.

## When to use MassTransit

- Saga-driven workflows over multiple services.
- Multi-broker portability (start RabbitMQ, move to Service Bus on Azure).
- Need contract-evolution tooling (interfaces vs concrete types, versioned topics).
- Existing team familiarity — it's the safest "pick a service bus" answer.

## When NOT to use MassTransit

- Greenfield on Marten/PostgreSQL + want one tool for in-proc + distributed → [Wolverine](./wolverine.md).
- Want minimal surface area, no state-machine DSL → [Rebus](./rebus.md).
- Polyglot fleet, K8s-first → [Dapr](./dapr.md).
- v9 commercial license is unacceptable to your org → pin v8 or pick alternatives now.

**Senior-level gotchas:**
- **v9 license switch is the elephant.** Decide deliberately: stay on v8 (MIT, mature, no new transports), pay for v9, or migrate. Don't drift onto v9 by accident — pin the major version in `Directory.Packages.props`.
- **`IPublishEndpoint` outside a consumer scope publishes immediately.** Without the EF outbox wired and a `DbContext` in scope, `Publish` flies even if your DB transaction later rolls back. The classic data-loss bug. Always: outbox + `SaveChangesAsync` then publish drains.
- **Message contracts as concrete types couple producers and consumers.** Use **interface contracts** (`public interface IOrderPlaced { Guid OrderId { get; } }`) — MassTransit generates proxies on each side. Producers and consumers can evolve independently.
- **Topology auto-creation in prod is a footgun.** First boot creates exchanges/queues with whatever settings the code declared. Deploy a typo, get a permanent ghost queue. In prod, set `ConfigureEndpoints` to only configure pre-declared topology, or declare via IaC and review.
- **Saga concurrency**: EF repository defaults to **optimistic** (rowversion). Under contention, conflicts surface as `DbUpdateConcurrencyException` — MassTransit retries automatically *if* `UseInMemoryOutbox` or `UseEntityFrameworkOutbox` is enabled. Without retries, you drop messages. For high-contention sagas use **pessimistic** (`SqlServerLockStatements`).
- **Prefetch vs concurrency.** RabbitMQ prefetch (`PrefetchCount`) ≠ MassTransit `ConcurrentMessageLimit`. Prefetch is per-channel buffering; concurrency is the consumer's `Task.Run` width. Set prefetch ≈ 1.5–2× concurrency, not orders of magnitude bigger.
- **Request-response uses temporary auto-delete reply queues.** Each instance creates one. If you scale to 1000 pods, that's 1000 queues. Idle reply queues are cheap on RabbitMQ but visible on Service Bus (entity quota). Reuse `IRequestClient` from DI; don't create a fresh one per call.
- **`InMemoryTransport` is for tests, not "we'll add a broker later."** No durability, no cross-process, no scaling. Wire RabbitMQ in dev too — Docker Compose makes it free.
- **Don't put domain logic in consumer constructors.** Heavy ctor work runs per message (per-scope DI). Move expensive init to `IConfigureReceiveEndpoint` or a hosted service.
- **`ConsumeContext.Publish` honors the outbox; `IPublishEndpoint.Publish` from a singleton does not.** Always prefer `ctx.Publish` inside a consumer.
- **Kafka via Rider is a separate package** (`MassTransit.Kafka`) and a different mental model — Kafka's partition/offset semantics don't map cleanly onto MassTransit's queue abstraction. Read the Kafka notes before assuming feature parity with RabbitMQ.
