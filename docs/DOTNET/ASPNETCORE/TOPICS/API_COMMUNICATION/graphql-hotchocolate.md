# GraphQL with HotChocolate

_Targets .NET 10 / C# 14 with HotChocolate v14+. See also: [REST](./rest.md), [gRPC](./grpc.md), [HotChocolate Subscriptions](../REAL_TIME_COMMUNICATION/hotchocolate-subscriptions.md)._

GraphQL is a query language and runtime where the **client** picks fields and the **server** advertises a typed schema. HotChocolate (`HotChocolate.AspNetCore`) is the de-facto .NET implementation — code-first by default (your C# types ARE the schema), with first-class EF Core integration, subscriptions, and DataLoader baked in. Pick GraphQL when one client (especially a mobile app) aggregates data from many services and you want to ship one schema instead of 14 REST endpoints. Don't pick it for simple CRUD — the operational complexity (depth limits, persisted queries, N+1 traps) doesn't pay off.

## Wiring it up

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>()
    .AddProjections()                        // [UseProjection]
    .AddFiltering()                          // [UseFiltering]
    .AddSorting()                            // [UseSorting]
    .AddInMemorySubscriptions()              // dev — swap for Redis in prod
    .ModifyRequestOptions(o =>
    {
        o.IncludeExceptionDetails = builder.Environment.IsDevelopment();
        o.ExecutionTimeout        = TimeSpan.FromSeconds(30);
    })
    .AddMaxExecutionDepthRule(8)             // ⚠ opt-in — defaults are unbounded
    .AddCostAnalyzer(o => o.MaxAllowedCost = 1000);

var app = builder.Build();
app.MapGraphQL();                            // POST /graphql + WS /graphql
app.Run();
```

## Resolvers

A resolver is just a method on the `Query` (or any object) type. Parameters are dependency-injected: services via `[Service]`, the cancellation token via type, the parent object implicitly:

```csharp
public sealed class Query
{
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Order> GetOrders([Service] AppDbContext db) => db.Orders;

    public async Task<Order?> GetOrderAsync(
        long id,
        [Service] IOrderRepo repo,
        CancellationToken ct)
        => await repo.FindAsync(id, ct);
}

[ExtendObjectType<Order>]                    // adds field resolvers to Order without touching the entity
public sealed class OrderExtensions
{
    public async Task<Customer?> GetCustomerAsync(
        [Parent] Order order,
        CustomerByIdDataLoader loader,
        CancellationToken ct)
        => await loader.LoadAsync(order.CustomerId, ct);
}
```

## The N+1 problem and DataLoader

A query for 100 orders, each requesting `customer.name`, will issue **101 SQL queries** if the resolver reads the customer per-order. DataLoader batches and caches per-request:

```csharp
public sealed class CustomerByIdDataLoader(
    IBatchScheduler scheduler,
    DataLoaderOptions options,
    IDbContextFactory<AppDbContext> dbFactory)
    : BatchDataLoader<long, Customer>(scheduler, options)
{
    protected override async Task<IReadOnlyDictionary<long, Customer>> LoadBatchAsync(
        IReadOnlyList<long> keys, CancellationToken ct)
    {
        await using var db = await dbFactory.CreateDbContextAsync(ct);
        return await db.Customers
            .Where(c => keys.Contains(c.Id))
            .ToDictionaryAsync(c => c.Id, ct);
    }
}

builder.Services.AddGraphQLServer().AddDataLoader<CustomerByIdDataLoader>();
```

The execution engine collects all `LoadAsync` calls within a single execution tick, dispatches one batch query, and resolves every field with the cached lookup.

**Always** use a `IDbContextFactory<T>` for DataLoaders — parallel field resolution will use the loader from multiple tasks, and a single `DbContext` is not thread-safe.

## Mutations

```csharp
public sealed class Mutation
{
    public async Task<Order> CreateOrderAsync(
        CreateOrderInput input,
        [Service] IOrderService svc,
        CancellationToken ct)
        => await svc.CreateAsync(input.CustomerId, input.Items, ct);
}

public sealed record CreateOrderInput(long CustomerId, IReadOnlyList<OrderLine> Items);
```

The naming convention `*Input` for inputs and a `*Payload` wrapper for outputs is the Relay convention — useful but optional. HotChocolate generates `CreateOrderInput` as a GraphQL `input` type automatically.

## Subscriptions

```csharp
public sealed class Subscription
{
    [Subscribe]
    [Topic("orders.created")]
    public Order OrderCreated([EventMessage] Order order) => order;
}

// publish from anywhere
await sender.SendAsync("orders.created", newOrder, ct);    // ITopicEventSender
```

`AddInMemorySubscriptions()` works for single-instance dev. For production, swap in `AddRedisSubscriptions(...)` so each pod doesn't see only its own publishes. See [hotchocolate-subscriptions](../REAL_TIME_COMMUNICATION/hotchocolate-subscriptions.md).

## Filtering / sorting / projection over EF Core

```csharp
[UseProjection]                              // selects only requested fields → SELECT specific columns
[UseFiltering]                               // exposes a `where` argument
[UseSorting]                                 // exposes an `order` argument
public IQueryable<Order> GetOrders([Service] AppDbContext db) => db.Orders;
```

Client query:

```graphql
query {
  orders(where: { total: { gt: 100 } }, order: { createdAt: DESC }) {
    id
    total
    customer { name }
  }
}
```

The translated SQL only selects the columns asked for and pushes the filter into a `WHERE` clause. Powerful — and dangerous (see gotchas).

## Persisted queries

Public-facing GraphQL endpoints are easy to attack: any client can send an arbitrary expensive query. **Persisted queries** flip the trust model — the server only accepts a known set of query hashes:

```csharp
.AddGraphQLServer()
    .UsePersistedOperationPipeline()
    .AddFileSystemOperationDocumentStorage("./queries");
```

Build-time, the client toolchain extracts every `.graphql` it uses, hashes them, and ships only `{ "id": "abc123", "variables": {...} }` at runtime. Combined with `MaxAllowedCost`, this is the right shape for production.

## Authorization

```csharp
.AddAuthorization()                          // wires HotChocolate to ASP.NET Core auth

[Authorize(Policy = "Admin")]
public sealed class Mutation { /* ... */ }

[Authorize(Roles = new[] { "Reader" })]
public IQueryable<Order> GetOrders(...) => ...;
```

Authorize at the field level (HotChocolate's `[Authorize]` attribute, *not* the ASP.NET one of the same name — they share the namespace name but apply differently). Operation-level authz is too coarse for GraphQL.

## Tooling

- **Nitro** (formerly Banana Cake Pop) — HotChocolate's own IDE, served at `/graphql` in dev. UI for queries, schema explorer, history.
- `dotnet graphql` CLI — schema export, code generation for client side (Strawberry Shake).

## Modern note (.NET 10)

- HotChocolate v14+ uses `Microsoft.Extensions.DependencyInjection` keyed services for resolver-scoped injection — no more custom scope plumbing.
- `[Authorize]` and `[Cost]` attributes are now first-class metadata picked up by the schema builder; no separate `.AddAuthorization()` boilerplate beyond the registration.
- The `Microsoft.AspNetCore.OpenApi` REST generator and HotChocolate can coexist on the same app — the OpenAPI doc covers your REST surface, the GraphQL schema covers the rest.

## REST vs gRPC vs GraphQL — when to pick GraphQL

| Need | Pick |
|---|---|
| Public API for many third parties | REST — broader tooling, CDN-friendly |
| Mobile client doing many roundtrips | **GraphQL** — one query, pick exactly the fields |
| Internal microservices, low latency, streaming | gRPC |
| Aggregating across multiple backends behind a BFF | **GraphQL** with field resolvers calling each backend |
| File uploads, simple CRUD | REST — GraphQL multipart is more friction than it's worth |

## Senior-level gotchas

- **`[UseProjection]` only fires when the resolver returns `IQueryable<T>`**. Returning `Task<List<T>>` materializes everything before projection — schema looks the same, performance does not.
- Depth and complexity limits are **opt-in**. `AddMaxExecutionDepthRule` and `AddCostAnalyzer` default off — a query of `{ user { friends { friends { friends ... 30 levels } } } }` will happily melt your DB. Set them before going to prod.
- A single shared `DbContext` across DataLoader batches and field resolvers will throw `InvalidOperationException: A second operation was started on this context...`. Use `IDbContextFactory<T>` and create per-loader / per-resolver scopes.
- `[UseFiltering]` exposes **every** scalar property on the entity by default. PII columns, internal flags — all filterable. Restrict with `[UseFiltering(typeof(OrderFilterType))]` and a hand-written filter type for any entity that crosses a trust boundary.
- Errors via thrown exceptions become opaque `Unexpected error`. Use `IError` / `ErrorBuilder` to surface domain errors with codes; reserve exceptions for genuinely unexpected faults.
- Subscriptions over `MapGraphQL` use the WebSocket protocol — make sure `app.UseWebSockets()` is in the pipeline **before** `MapGraphQL`.
- Per-field authorization runs **after** the field is resolved if you put it on the return type. To deny before computing, attribute the field method, not the type.
- The HTTP `GET` form of GraphQL (queries only) is cacheable; `POST` is not. Reverse proxies that don't know GraphQL will not cache `POST /graphql` even for a pure read — that's why persisted queries with `GET` matter.
- `IBatchScheduler` is request-scoped; capturing a DataLoader in a static or singleton silently breaks batching across requests.
