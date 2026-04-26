# Response Compression

_Targets .NET 10 / C# 14. See also: [Middlewares](./middlewares.md), [OutputCache](../CACHING/output-cache.md)._

`Microsoft.AspNetCore.ResponseCompression` compresses response bodies based on the client's `Accept-Encoding` header — Brotli, Gzip, Deflate. Useful when there is **no** upstream proxy already compressing (rare in production), for serving large JSON / static-asset fallback in single-process deployments, or when you want compressed bodies cached by `OutputCache`.

If your service sits behind nginx with `gzip on`, Cloudflare, Azure Front Door, or App Service "Dynamic compression", the middleware does redundant work and burns CPU for no gain. **Measure first.**

## Wiring

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddResponseCompression(opts =>
{
    opts.EnableForHttps = true;                 // see BREACH note below
    opts.Providers.Add<BrotliCompressionProvider>();
    opts.Providers.Add<GzipCompressionProvider>();
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        ["application/problem+json", "application/yaml"]);
});

builder.Services.Configure<BrotliCompressionProviderOptions>(o => o.Level = CompressionLevel.Fastest);
builder.Services.Configure<GzipCompressionProviderOptions>(  o => o.Level = CompressionLevel.Fastest);

var app = builder.Build();

app.UseResponseCompression();      // before everything that produces a body
app.UseStaticFiles();
app.UseRouting();
app.MapControllers();

app.Run();
```

## Provider choice

| Provider | Ratio | CPU | Notes |
|---|---|---|---|
| **Brotli** | Best | Higher than gzip at same level | Standard since 2017; supported by all evergreen browsers. Pick first if both client and server support it. |
| **Gzip** | Good | Low | Universal — every HTTP client understands it. Reliable fallback. |
| **Deflate** | Same as gzip without the gzip header framing | Low | Effectively dead. Don't bother. |

`Providers.Add<>()` registration order is **preference order** — the middleware picks the first provider the client also accepts.

## Compression level — pick `Fastest`, not `Optimal`

`CompressionLevel.Optimal` on Brotli is multiplicatively slower than `Fastest` for marginal ratio gains. On live traffic, you want `Fastest` — the bandwidth saved beyond that doesn't justify the CPU. `Optimal` is for **build-time** static-asset compression where CPU is one-shot.

```csharp
builder.Services.Configure<BrotliCompressionProviderOptions>(o => o.Level = CompressionLevel.Fastest);
```

## MIME type allow-list

Defaults compress text-shaped types (`text/*`, `application/json`, `application/xml`) and skip already-compressed binary types (`image/*`, `video/*`, `application/zip`, `application/pdf`). Extend the allow-list explicitly when adding new content types:

```csharp
opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
    ["application/problem+json", "application/yaml", "image/svg+xml", "application/x-ndjson"]);
```

**Never** compress already-compressed payloads (JPEG, MP4, `.gz`, `.br`, `.zip`) — wastes CPU and the result is typically *larger* than the input.

## HTTPS and BREACH

Compression on HTTPS responses opens the **BREACH** attack class: when a response mixes attacker-controlled input (e.g. a query parameter echoed in the body) with a secret (auth token, anti-forgery token, CSRF token), an attacker observing ciphertext lengths can extract the secret one byte at a time.

Mitigations:

- Don't reflect attacker-controlled input alongside secrets in the same response. (Hardest in practice.)
- Add per-response length randomization for bodies containing secrets (a few padding bytes).
- Set `EnableForHttps = false` for endpoints that contain secrets.
- Use HSTS plus short-lived tokens — reduces the practical attack window.

`EnableForHttps = true` is the modern default, but BREACH is a real attack vector. Audit endpoints that return forms with anti-forgery tokens *and* echo request input.

## Pipeline ordering

`UseResponseCompression()` must run **before** any middleware that produces a body — including `UseStaticFiles()` and any endpoint mapping. It wraps `Response.Body` with a compression stream; once another middleware has called `Response.StartAsync()` or written headers, the wrap is too late and the middleware silently does nothing.

For caching, the order is critical:

```csharp
app.UseResponseCompression();   // first
app.UseOutputCache();           // then
app.UseRouting();
app.MapControllers();
```

This way `OutputCache` stores the *compressed* body and replays it on cache hits — one compress per cache fill, not per request. Reverse the order and you compress on every single hit.

## When to use

- Single-process deployments with no upstream proxy doing compression.
- Behind a Kestrel + nginx pairing where nginx does *not* enable `gzip on`.
- APIs serving large JSON (>1 KB typical body, sustained throughput).
- When you want `OutputCache` entries stored already-compressed.

## When NOT to use

- Behind Cloudflare, Azure Front Door, AWS CloudFront, or App Service Dynamic Compression — they already compress and yours is double work.
- Endpoints serving already-compressed binary payloads.
- Tiny responses (<1 KB) where the gzip/Brotli framing overhead exceeds the savings.
- Endpoints that mix secrets and attacker-controlled data over HTTPS without BREACH mitigation.

## Senior-level gotchas

- `EnableForHttps = true` opens BREACH if your endpoints reflect client input alongside secrets. Don't flip the flag without auditing — and especially never on form POST responses that include anti-forgery tokens plus echoed input.
- The middleware automatically sets `Vary: Accept-Encoding`. Reverse proxies and CDNs must respect it or they'll serve a gzip body to a client that didn't ask for one (some user agents 4xx on that). Confirm your CDN's Vary handling — some collapse it by default.
- `CompressionLevel.Optimal` on Brotli is dramatically slower for marginal gain on live traffic. Use `Fastest`. Reserve `Optimal` for build-time pre-compression of static assets (e.g. webpack-emitted `.br` artifacts served by `app.UseStaticFiles()` with the `.br` content-encoding fallback).
- Ordering with `UseStaticFiles()` matters — compression must come *before* it, otherwise static files are served with their pre-existing headers locked. With `UseResponseCompression()` first, both static and dynamic responses are eligible.
- Streaming responses (`Response.Body.WriteAsync` driven by an `IAsyncEnumerable<T>`, server-sent events, chunked transfer) compress in chunks — that's fine, but each chunk has framing overhead. For SSE specifically, prefer no compression or compress only when the SSE producer batches large frames.
- `Response.StartAsync()` (often called implicitly by the first body write) locks headers. Any compression decision must happen *before* that — middleware ordering enforces it, but custom code that flushes early defeats it silently. The middleware will not log a warning.
- Per-MIME-type opt-in means new content types you serve (e.g. `application/x-ndjson`, `application/yaml`, `application/grpc-web`) are uncompressed by default until you extend `opts.MimeTypes`. Easy to miss until you add a new endpoint and bandwidth doubles.
- The middleware compresses `Response.Body`, but if your endpoint writes to `HttpResponse.BodyWriter` (a `PipeWriter`) directly, the compression stream is wrapped around it transparently — no special handling. The exception is third-party libraries that grab `Response.Body` *before* `UseResponseCompression()` ran (e.g. some health-check formatters); those bypass compression silently.
- `UseResponseCompression()` does *nothing* if the client doesn't send `Accept-Encoding`. Old corporate HTTP libraries sometimes strip it; if you measure no compression on requests you expect to be compressed, check the request headers in a trace before debugging the middleware.
- Compression buffering interacts poorly with `IHttpResponseBodyFeature.DisableBuffering()` — disabling buffering forces the compression stream to flush each write, which can produce many tiny gzip frames. Either keep buffering on, or set a sensible application-level batch size before you write.
