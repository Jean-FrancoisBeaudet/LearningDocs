# gRPC

_Targets .NET 10 / C# 14. See also: [REST](./rest.md), [GraphQL HotChocolate](./graphql-hotchocolate.md), [HttpClient & HttpClientFactory](./httpclient-and-httpclientfactory.md)._

gRPC is a **contract-first, HTTP/2, binary RPC** framework. You write `.proto` files (Protocol Buffers), the toolchain generates strongly-typed server stubs and client classes for every supported language, and the wire format is a length-prefixed Protobuf payload over HTTP/2. Pick gRPC for service-to-service traffic in a polyglot mesh, for streaming workloads, and anywhere you'd rather have a compiler enforce the contract than rely on Postman discipline. Don't pick it for browser clients without grpc-Web (browsers can't speak raw HTTP/2 trailers), and don't pick it as a public API for unknown third parties — REST + OpenAPI has more reach.

## Project shape

```xml
<!-- Server.csproj -->
<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" Version="2.66.*" />
  <Protobuf Include="Protos/orders.proto" GrpcServices="Server" />
</ItemGroup>

<!-- Client.csproj -->
<ItemGroup>
  <PackageReference Include="Grpc.Net.Client" Version="2.66.*" />
  <PackageReference Include="Grpc.Tools" Version="2.66.*" PrivateAssets="All" />
  <Protobuf Include="Protos/orders.proto" GrpcServices="Client" />
</ItemGroup>
```

`GrpcServices="Both"` for monorepos that share the proto. The build emits `OrdersBase` (server abstract class) and `OrdersClient` (client) into the assembly — no checked-in generated code.

```protobuf
// Protos/orders.proto
syntax = "proto3";
option csharp_namespace = "Acme.Orders.V1";
package acme.orders.v1;

import "google/protobuf/timestamp.proto";

service Orders {
  rpc Get          (OrderRequest)         returns (Order);
  rpc Stream       (StreamRequest)        returns (stream Order);
  rpc Bulk         (stream Order)         returns (BulkResult);
  rpc Chat         (stream ClientMsg)     returns (stream ServerMsg);
}

message OrderRequest { int64 id = 1; }
message Order        { int64 id = 1; string sku = 2; google.protobuf.Timestamp createdAt = 3; }
```

## Server

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc(o =>
{
    o.MaxReceiveMessageSize = 4 * 1024 * 1024;     // 4 MB default
    o.EnableDetailedErrors   = builder.Environment.IsDevelopment();
});
builder.Services.AddGrpcReflection();              // optional: lets grpcurl discover services

var app = builder.Build();
app.MapGrpcService<OrdersService>();
if (app.Environment.IsDevelopment()) app.MapGrpcReflectionService();
app.Run();

public sealed class OrdersService(IOrderRepo repo) : Orders.OrdersBase
{
    public override async Task<Order> Get(OrderRequest request, ServerCallContext context)
    {
        var ct = context.CancellationToken;        // ALWAYS use this, never default
        return await repo.FindAsync(request.Id, ct)
            ?? throw new RpcException(new Status(StatusCode.NotFound, $"Order {request.Id} not found"));
    }
}
```

`ServerCallContext.CancellationToken` is wired to the client deadline + connection drop. Pass it to every downstream call.

## Client

```csharp
// Program.cs of the consumer
builder.Services.AddGrpcClient<Orders.OrdersClient>(o =>
{
    o.Address = new Uri("https://orders.internal");
})
.ConfigureChannel(c =>
{
    c.MaxReceiveMessageSize = 16 * 1024 * 1024;
    c.HttpHandler = new SocketsHttpHandler
    {
        EnableMultipleHttp2Connections = true,
        PooledConnectionIdleTimeout    = TimeSpan.FromMinutes(2),
        KeepAlivePingDelay             = TimeSpan.FromSeconds(60),
        KeepAlivePingTimeout           = TimeSpan.FromSeconds(30),
    };
})
.AddStandardResilienceHandler();                   // Microsoft.Extensions.Http.Resilience
```

`AddGrpcClient<T>` integrates with `IHttpClientFactory` — see [httpclient-and-httpclientfactory.md](./httpclient-and-httpclientfactory.md). The factory hands you a fresh `OrdersClient` per request scope, but the underlying `HttpMessageHandler` (and HTTP/2 connection) is pooled.

## The four call styles

```csharp
// 1. Unary
var order = await client.GetAsync(new() { Id = 42 }, deadline: DateTime.UtcNow.AddSeconds(2), cancellationToken: ct);

// 2. Server streaming
using var call = client.Stream(new StreamRequest { Since = ... }, cancellationToken: ct);
await foreach (var item in call.ResponseStream.ReadAllAsync(ct))
    process(item);

// 3. Client streaming
using var call = client.Bulk(cancellationToken: ct);
foreach (var o in batch)
    await call.RequestStream.WriteAsync(o, ct);
await call.RequestStream.CompleteAsync();
var result = await call.ResponseAsync;

// 4. Bidirectional
using var call = client.Chat(cancellationToken: ct);
var receive = Task.Run(async () =>
{
    await foreach (var msg in call.ResponseStream.ReadAllAsync(ct))
        handle(msg);
}, ct);
await call.RequestStream.WriteAsync(new ClientMsg { ... }, ct);
await call.RequestStream.CompleteAsync();
await receive;
```

## Errors, deadlines, cancellation

- Server throws `RpcException(new Status(StatusCode.X, "msg"))`. Don't let raw exceptions leak — without `EnableDetailedErrors`, clients only see `StatusCode.Unknown`.
- **Deadlines** are absolute timestamps; **cancellation tokens** are local to your process. The deadline propagates via `grpc-timeout` header — chained services should re-derive deadlines: `context.Deadline` on the server is what the upstream caller demanded.
- `CancellationToken.None` + no deadline = a stuck HTTP/2 stream that survives forever. Always set one or the other.

## Metadata (headers / trailers)

```csharp
// Client → server header
var headers = new Metadata { { "x-tenant", tenantId } };
var order = await client.GetAsync(req, headers, deadline, ct);

// Server reads, sets a response trailer
var tenant = context.RequestHeaders.GetValue("x-tenant");
context.ResponseTrailers.Add("x-cache-hit", "true");
```

Binary header names must end with `-bin` and the value must be base64. Headers are sent before the first message; trailers after the last.

## Interceptors

Cross-cutting concerns live here, not in the service:

```csharp
public sealed class AuthInterceptor(ITokenSource ts) : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request, ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var headers = context.Options.Headers ?? new Metadata();
        headers.Add("authorization", $"Bearer {ts.Get()}");
        return continuation(request, context.WithOptions(context.Options.WithHeaders(headers)));
    }
}

builder.Services.AddGrpcClient<Orders.OrdersClient>(...).AddInterceptor<AuthInterceptor>();
```

Server-side: `AddGrpc(o => o.Interceptors.Add<LoggingInterceptor>())`.

## Browser clients

Two transcoding bridges:

| Option | Use when |
|---|---|
| **gRPC-Web** (`AddGrpcWeb`, `app.UseGrpcWeb()`) | The client is a browser SPA. Adds a thin protocol shim — server stays gRPC. |
| **gRPC JSON transcoding** (`AddJsonTranscoding`, `Microsoft.AspNetCore.Grpc.JsonTranscoding`) | You want one definition serving both gRPC and REST + OpenAPI. Annotate methods with `google.api.http` rules in the proto. |

## Health and observability

- `Grpc.HealthCheck` exposes the standard `grpc.health.v1.Health` service for k8s-native probes.
- gRPC clients and servers emit OpenTelemetry traces and metrics out of the box when you wire `AddOpenTelemetry().WithTracing(t => t.AddGrpcClientInstrumentation().AddAspNetCoreInstrumentation())`.

## Modern note (.NET 10)

- The Kestrel HTTPS dev cert is required for HTTP/2 (no h2c on most clients). Trust the dev cert: `dotnet dev-certs https --trust`.
- Native AOT is supported for gRPC services with the `Grpc.AspNetCore.Server` toolchain — generated code is annotated for trimming.
- `proto3` *finally* has explicit field presence via the `optional` keyword (no more "is it default value or unset?" ambiguity for scalars).

## Senior-level gotchas

- **HTTP/2 over plain HTTP**: Kestrel serves h2c only when you explicitly opt in (`ListenLocalhost(5001, lo => lo.Protocols = HttpProtocols.Http2)`). Behind a TLS-terminating proxy that can't speak HTTP/2 backend, you need this.
- **Deadline propagation is manual**: a service that calls another service downstream must forward `context.Deadline` (or `context.CancellationToken`) — the framework does not do it for you.
- **`DateTime` ↔ `Timestamp`**: `Timestamp.FromDateTime(dt)` requires `dt.Kind == DateTimeKind.Utc`; mid-level devs hit `ArgumentException` here constantly. Prefer `DateTimeOffset` everywhere and convert at the edge.
- **Null vs default for proto3 scalars**: a `string` field defaults to `""`, an `int32` to `0`. There is no "unset" distinction unless you use `optional`. Mid-level devs ship "delete this field by sending null" semantics that don't exist on the wire.
- **`MaxReceiveMessageSize` defaults to 4 MB** on both client and server. Bulk endpoints fail with `ResourceExhausted` and no obvious cause — set both ends.
- **Channel reuse**: reuse one `GrpcChannel` per target — it owns the HTTP/2 connection. `IHttpClientFactory` integration handles this; building channels by hand and `using` them defeats it.
- **Reflection in production**: `MapGrpcReflectionService` exposes your full schema. Gate behind dev only or auth.
- **Generated code is regenerated on every build** when the proto changes. Don't hand-edit the `obj/` partial classes — extend via `partial class` in your own file.
