# Wolverine

_Targets .NET 10 / C# 14 with Wolverine 3+. See also: [MassTransit](./masstransit.md), [Rebus](./rebus.md), [Dapr](./dapr.md), [OpenTelemetry](../LOGGING_AND_MONITORING/opentelemetry.md)._

Wolverine (JasperFx, by the author of Marten and Lamar) is a **command bus + message bus** in one library. It replaces MediatR for in-process dispatch *and* MassTransit/Rebus for distributed messaging, with a single mental model and a single configuration. The differentiator: heavy use of **runtime code generation** (Lamar) to produce zero-allocation handler invocations, "static handler" conventions (no `IHandleMessages<T>` interface — just a static method), and first-class transactional outbox/inbox via Marten or EF Core. Pick it when you're greenfield on PostgreSQL, want one tool for both in-proc and async, and are willing to buy into the conventions. Don't pick it for a brownfield app deeply wedded to MS DI + MediatR — Lamar takes over the container, and the source-gen magic is harder to debug than vanilla code.

## Wiring

```csharp
builder.Host.UseWolverine(opts =>
{
    opts.Discovery.IncludeAssembly(typeof(Program).Assembly);

    // distributed transport
    opts.UseRabbitMq(builder.Configuration.GetConnectionString("Rabbit")!)
        .AutoProvision();

    opts.PublishMessage<OrderPlaced>().ToRabbitExchange("orders");
    opts.ListenToRabbitQueue("orders-svc");

    // transactional outbox via EF Core
    opts.Services.AddDbContext<AppDbContext>(o => o.UseNpgsql("..."));
    opts.UseEntityFrameworkCoreTransactions();
    opts.Policies.AutoApplyTransactions();          // every handler runs in a tx
    opts.Policies.UseDurableLocalQueues();          // durable in-proc inbox/outbox
});
```

`Host.UseWolverine` swaps the DI container to **Lamar** under the hood. That's the load-bearing decision — `Microsoft.Extensions.DependencyInjection` is still callable, but Lamar's compiled runtime is what runs.

## Static handlers (the convention)

```csharp
public static class OrderHandlers
{
    public static async Task Handle(
        OrderPlaced cmd,
        AppDbContext db,                            // injected by Lamar
        IMessageBus bus,
        CancellationToken ct)
    {
        db.Orders.Add(new Order(cmd.Id, cmd.Total));
        await db.SaveChangesAsync(ct);

        // cascading message — handler return values get dispatched
        await bus.PublishAsync(new OrderConfirmed(cmd.Id));
    }
}
```

No interface, no attribute. Wolverine source-generates a wrapper that resolves `db`, `bus`, `ct`, calls `Handle`, and (with `AutoApplyTransactions`) commits the EF tx and the outbox in one shot. Generated code lives at runtime — dump it for inspection (see gotchas).

## Cascading messages

A handler can `return` (or yield) messages and Wolverine dispatches them:

```csharp
public static OrderConfirmed Handle(OrderPlaced cmd, AppDbContext db)
{
    db.Orders.Add(new Order(cmd.Id, cmd.Total));
    db.SaveChanges();
    return new OrderConfirmed(cmd.Id);              // dispatched after handler returns
}

// Multiple cascades
public static IEnumerable<object> Handle(OrderPlaced cmd) =>
    new object[] { new OrderConfirmed(cmd.Id), new ChargeRequested(cmd.Id) };
```

Inside `AutoApplyTransactions`, cascades enroll in the same transaction's outbox — they don't fly until commit.

## Sagas

```csharp
public sealed class OrderSaga : Saga
{
    public Guid Id { get; set; }
    public bool Charged { get; set; }
    public bool Shipped { get; set; }

    public static (OrderSaga, ChargeRequested) Start(OrderPlaced m) =>
        (new OrderSaga { Id = m.OrderId }, new ChargeRequested(m.OrderId));

    public ShipRequested Handle(PaymentReceived m)
    {
        Charged = true;
        return new ShipRequested(Id);
    }

    public void Handle(Shipped m)
    {
        Shipped = true;
        MarkCompleted();
    }
}
```

Persisted as a Marten document by default; EF Core saga support exists. No state-machine DSL — saga state is the object's fields, transitions are method bodies. Good for ≤5 states; beyond that, you'll wish for MassTransit's DSL.

## In-proc command bus (replaces MediatR)

```csharp
public sealed class OrderController(IMessageBus bus) : ControllerBase
{
    [HttpPost]
    public Task<OrderResult> Place(PlaceOrder cmd, CancellationToken ct) =>
        bus.InvokeAsync<OrderResult>(cmd, ct);      // synchronous in-proc dispatch
}
```

`InvokeAsync<T>` runs the handler in-process, returns the (typed) result. Same handler discovery as for messaging — one programming model, two execution modes.

## Outbox / inbox

`UseEntityFrameworkCoreTransactions` + `AutoApplyTransactions` is the killer feature: every handler is wrapped in a `BeginTransaction` → handler body → `SaveChanges` → publish-to-outbox → commit cycle, automatically. No manual `UseBusOutbox()` calls per consumer (MassTransit) or unit-of-work middleware setup (Rebus). It Just Works — until it doesn't (see gotchas).

`UseDurableLocalQueues` adds durable inbox semantics for in-proc messages too, surviving process crash mid-handler.

## When to use Wolverine

- Greenfield on PostgreSQL + Marten — the outbox/inbox story is best-in-class here.
- Want **one** tool for in-proc command bus AND async messaging.
- Comfortable with Lamar / runtime codegen / opinionated conventions.
- Like the Marten ecosystem (event store, projections, tenants).

## When NOT to use Wolverine

- Brownfield app heavily invested in MS.DI + MediatR + EF — switching the container is high-risk, low-payoff.
- Need a state-machine saga DSL → [MassTransit](./masstransit.md).
- Need a tiny dependency surface → [Rebus](./rebus.md).
- Polyglot fleet → [Dapr](./dapr.md).
- Team unwilling to read source-generated code when something misbehaves.

**Senior-level gotchas:**
- **Lamar replaces the DI container.** Most MS.DI APIs work, but advanced scenarios (some keyed-service shapes, certain `IServiceProviderIsService` checks, third-party libraries that introspect the container) can misbehave. Audit dependencies that touch `IServiceCollection`/`IServiceProvider` deeply.
- **Source-generated handlers are opaque to debuggers.** Set `opts.OptimizeArtifactWorkflow(TypeLoadMode.Static)` in dev to write generated code to disk under `Internal/Generated/` — then you can step through it. In prod use `Dynamic` for fastest startup, but log the generated assembly hash for incident forensics.
- **`AutoApplyTransactions` only enrols handlers that take a `DbContext`/`IDocumentSession` parameter.** A handler that doesn't take one is *not* in a transaction and `bus.PublishAsync` fires immediately. Be explicit; don't assume the policy applied.
- **Cascading messages can hide flow.** When a handler `return`s a message, the dispatch is implicit — code review misses it. Use Wolverine's preview/diagnostic APIs in tests to assert routing, and prefer explicit `await bus.PublishAsync` for non-trivial chains.
- **Cold-start cost from runtime codegen.** First `InvokeAsync` per handler triggers compilation. Pre-compile in CI/CD with `OptimizeArtifactWorkflow` + check generated artifacts into the repo, *or* warm up at startup in a `BackgroundService`.
- **Outbox is tied to Marten or EF Core.** No "broker-only" outbox mode. If you don't have a relational store, you can't use the durable outbox — you fall back to broker semantics + your own idempotency.
- **Saga persistence on Marten ≠ EF.** Switching stores requires re-modeling — Marten sagas are JSON documents with optimistic concurrency; EF sagas are rows. Pick early.
- **Static handlers feel weird until they don't.** `Handle` methods are static for a reason: no per-handler instance allocation, no constructor injection (deps come per-call from Lamar). You can use instance handlers (`public class FooHandler { public Task Handle(...) }`) — but the static version is the fast path the team optimizes for.
- **`InvokeAsync` is synchronous-ish but still runs through Wolverine middleware.** Logging, validation, retry policies all execute. It's not free; for a hot path measure it vs direct method call.
- **Sustainability check.** JasperFx is a small commercial-OSS shop. Wolverine is mature, well-documented, and used in production, but the bus factor is smaller than MassTransit's. Read the support model before betting a 10-year platform on it.
- **OpenTelemetry is built in but you must add the source.** Enable via `opts.Services.AddOpenTelemetry().WithTracing(t => t.AddSource("Wolverine"))` — the activity source name is `Wolverine`.
- **Don't mix MediatR and Wolverine in the same process.** They overlap. If migrating, do it module-by-module — keep one bus per logical bounded context until the cutover is complete.
