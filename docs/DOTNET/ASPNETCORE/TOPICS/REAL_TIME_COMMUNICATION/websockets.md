# WebSockets

_Targets .NET 10 / C# 14. See also: [HotChocolate Subscriptions](./hotchocolate-subscriptions.md), [gRPC](../API_COMMUNICATION/grpc.md), [BackgroundService](../ASPNET_CORE_BASICS/background-service.md). SignalR is the higher-level abstraction over this — see the upcoming `signalr.md`._

A WebSocket is the **RFC 6455** protocol: HTTP `Upgrade` handshake, then full-duplex framed messages over a single TCP connection. ASP.NET Core exposes the raw socket via `System.Net.WebSockets.WebSocket` — no RPC, no reconnect, no transport fallback, no broadcast. Pick raw WS when you control both ends, the wire format is yours, and SignalR's overhead (its own framing, hubs, JS client) is in the way. Skip it for typical real-time UI work — SignalR is what most apps actually want.

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);
var app     = builder.Build();

app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromSeconds(30),       // PING frames keep proxies open
});

app.MapGet("/ws", async (HttpContext ctx, ConnectionRegistry registry, CancellationToken ct) =>
{
    if (!ctx.WebSockets.IsWebSocketRequest)
    {
        ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
        return;
    }

    using var socket = await ctx.WebSockets.AcceptWebSocketAsync();
    var connectionId = Guid.NewGuid().ToString("N");
    await registry.RunAsync(connectionId, socket, ct);
});

app.Run();
```

The `AcceptWebSocketAsync` call sends the 101 Switching Protocols response and gives you the duplex `WebSocket`. Anything you do with `HttpContext` after this point is unsafe — the request pipeline is done; your `IServiceScope` is the only durable thing.

## The receive loop

A single message can span many frames. Always loop until `EndOfMessage`:

```csharp
public async Task RunAsync(string id, WebSocket socket, CancellationToken ct)
{
    var buffer = new byte[8 * 1024];
    using var ms = new MemoryStream();

    while (socket.State == WebSocketState.Open && !ct.IsCancellationRequested)
    {
        ms.SetLength(0);
        WebSocketReceiveResult result;
        do
        {
            result = await socket.ReceiveAsync(buffer, ct);
            if (result.MessageType == WebSocketMessageType.Close)
            {
                await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "client closed", ct);
                return;
            }
            ms.Write(buffer, 0, result.Count);
        }
        while (!result.EndOfMessage);

        if (result.MessageType == WebSocketMessageType.Text)
            await HandleTextAsync(id, ms.GetBuffer().AsMemory(0, (int)ms.Length), ct);
        else
            await HandleBinaryAsync(id, ms.GetBuffer().AsMemory(0, (int)ms.Length), ct);
    }
}
```

Don't assume one `ReceiveAsync` returns one logical message. A 64 KB JSON payload can land as a dozen frames; binary uploads commonly do.

## Sending — and why it's not thread-safe

`SendAsync` is **not** safe for concurrent callers. Two threads writing at once will interleave frame headers and corrupt the stream. The simplest correct pattern is one writer task fed by a `Channel<T>`:

```csharp
public sealed class WsConnection : IAsyncDisposable
{
    private readonly WebSocket _socket;
    private readonly Channel<ReadOnlyMemory<byte>> _outbox =
        Channel.CreateBounded<ReadOnlyMemory<byte>>(new BoundedChannelOptions(256)
        {
            FullMode = BoundedChannelFullMode.DropOldest,           // backpressure policy
            SingleReader = true,
        });

    public WsConnection(WebSocket socket) => _socket = socket;

    public ValueTask SendAsync(ReadOnlyMemory<byte> payload, CancellationToken ct)
        => _outbox.Writer.WriteAsync(payload, ct);

    public async Task RunSendLoopAsync(CancellationToken ct)
    {
        await foreach (var msg in _outbox.Reader.ReadAllAsync(ct))
        {
            await _socket.SendAsync(msg, WebSocketMessageType.Text, endOfMessage: true, ct);
        }
    }

    public async ValueTask DisposeAsync()
    {
        _outbox.Writer.TryComplete();
        if (_socket.State == WebSocketState.Open)
            await _socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "shutdown", default);
        _socket.Dispose();
    }
}
```

`Channel<T>` gives you backpressure (`BoundedChannelOptions`), a single-writer rule by construction, and clean shutdown via `TryComplete`. A `SemaphoreSlim` works too, but channels separate the producer rate from the socket rate — slow client doesn't block business code.

## Graceful close

The WebSocket close handshake is two-sided: each side sends a Close frame, then the underlying TCP closes.

```csharp
// Server initiates
await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "shutting down", ct);

// Server received Close from client
if (result.MessageType == WebSocketMessageType.Close)
    await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "ack", ct);
```

`CloseAsync` sends and waits for the peer; `CloseOutputAsync` sends and returns immediately (use it when you don't trust the peer to ack — e.g., shutdown timeouts).

## Auth on the upgrade

The browser `WebSocket` constructor doesn't let you set arbitrary headers — including `Authorization`. Three options:

```csharp
// 1. Cookie auth — the upgrade carries cookies automatically (same-origin or CORS-credentialed).
app.UseAuthentication();
app.UseAuthorization();
app.MapGet("/ws", [Authorize] async (HttpContext ctx, /* ... */) => { /* ... */ });

// 2. Subprotocol negotiation — bearer token shipped as a subprotocol value (Kubernetes pattern).
//    Client: new WebSocket(url, ["bearer.<token>"])
var sub = ctx.WebSockets.WebSocketRequestedProtocols
    .FirstOrDefault(p => p.StartsWith("bearer.", StringComparison.Ordinal));
if (sub is null) { ctx.Response.StatusCode = 401; return; }
var token = sub["bearer.".Length..];
// validate, then:
using var socket = await ctx.WebSockets.AcceptWebSocketAsync(sub);

// 3. Short-lived signed URL (?ticket=...) — simplest for browsers, watch out for token in access logs.
```

Never put the bearer token in the URL of a long-lived WS — it lands in proxy logs and browser history.

## Behind reverse proxies

WebSockets need `Connection: Upgrade` to flow end-to-end. The footguns:

- **Nginx**: explicit `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade"; proxy_read_timeout 3600s;`
- **Azure App Service**: enable WebSockets in the General settings; set `WEBSITES_HTTPLOGGING_RETENTION_DAYS=0` if you don't want connection lines in logs.
- **Azure Application Gateway / Front Door**: WS works but **the listener probe path must respond on the same backend** — a 404'd probe will mark the backend unhealthy and drop your sockets every 30 s.
- **ARR Affinity / sticky sessions**: required if your app holds in-memory connection state — without it, broadcasts only reach the connections on the current pod. (This is the original reason SignalR has a backplane.)

## Comparison — when raw WS is the right call

| Need | Pick |
|---|---|
| Server-push notifications, presence, chat, typing indicators | **SignalR** — reconnect, fallback, broadcast for free |
| Browser → server real-time, custom protocol, control of every byte | **Raw WebSockets** |
| Strongly-typed bidi RPC for service-to-service | **gRPC bidi streaming** |
| Server → browser only, simple, behind plain HTTP | **Server-Sent Events** (`text/event-stream`) — no upgrade, just a long GET |
| GraphQL clients with `subscription` operations | **[HotChocolate Subscriptions](./hotchocolate-subscriptions.md)** |

## Senior-level gotchas

- **Sends are not thread-safe.** Two `SendAsync` callers will interleave frames and corrupt the stream. Always serialize through a `Channel<T>` or `SemaphoreSlim`.
- **One `ReceiveAsync` ≠ one message.** Loop on `EndOfMessage`. Forgetting this is the most common WebSocket bug in .NET code.
- **`KeepAliveInterval`** sends WS PINGs. Stateful proxies (Nginx, App Gateway) close idle TCP after 60–240s — without pings, your connection dies silently and the next send throws `WebSocketException`.
- **Kestrel limits**: `Limits.WebSockets.DefaultBufferSize` (default 4 KB) governs internal frame buffering, not message size; no built-in max-message limit — write your own length check or you'll OOM on a single 2 GB binary message.
- **`HttpContext` is unsafe past upgrade.** Extract user id, claims, headers up-front; capturing the context in a long-running Task is a use-after-free in disguise.
- **No reconnect built in.** The browser `WebSocket` constructor is fire-and-forget — closed means closed. Clients must implement reconnect-with-backoff themselves; that's the single biggest reason teams move to SignalR.
- **Per-connection state in a `Dictionary<string, Connection>`** breaks the moment you scale out. Either run single-instance (and accept the SPOF), shard by user id with sticky sessions, or use a backplane (Redis pub/sub / SignalR / a real broker).
- **`CloseAsync` waits for the peer ack.** Under shutdown timeouts, prefer `CloseOutputAsync` and let the receive loop observe `WebSocketState.CloseReceived` to terminate.
- **JSON over WS** with `JsonSerializer.SerializeAsync(stream, ...)` is allocation-heavy because each message is one frame. For high throughput, serialize to `ArrayBufferWriter<byte>`, send once, reuse the buffer.
- **`WebSocketException` inside the receive loop** is easy to swallow as "client just disconnected." Log the `WebSocketError` and `InnerException` — half the time it's actually `Kestrel`, IIS, or the client running out of buffer.
