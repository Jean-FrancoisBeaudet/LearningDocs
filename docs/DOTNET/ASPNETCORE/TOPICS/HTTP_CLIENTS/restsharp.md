# RestSharp

_Targets .NET 10 / C# 14. See also: [HttpClient & HttpClientFactory](../API_COMMUNICATION/httpclient-and-httpclientfactory.md), [Refit](./refit.md), [Polly](./polly.md), [Microsoft Resilience](./microsoft-resilience.md)._

`RestSharp` (NuGet: `RestSharp`) is the older, dynamic-style HTTP client library that predates `IHttpClientFactory` and `Refit`. Modern v110+ is internally a thin layer over `HttpClient`, exposing a fluent `RestRequest`/`RestResponse` API with built-in parameter binding, authenticators, and serialization. In greenfield .NET 10 code there are few reasons to pick it over a typed `HttpClient` + Refit — the practical use cases are: maintaining an existing RestSharp codebase, using its ergonomic OAuth1 / NTLM authenticators, or scripting one-off integrations where the dynamic shape beats writing a contract.

## Basic usage

```csharp
var client = new RestClient("https://api.github.com");

var request = new RestRequest("/users/{login}/repos")
    .AddUrlSegment("login", "octocat")
    .AddQueryParameter("sort", "updated")
    .AddQueryParameter("per_page", 10)
    .AddHeader("User-Agent", "acme/1.0");

var repos = await client.GetAsync<List<Repo>>(request, cancellationToken);
```

`RestClient` is the long-lived object (think: equivalent to `HttpClient`). `RestRequest` is per-call. Parameter helpers (`AddUrlSegment`, `AddQueryParameter`, `AddHeader`, `AddJsonBody`, `AddFile`) bind to the request — strongly preferred over interpolated URLs because they handle encoding correctly.

## Wiring with `IHttpClientFactory` (the only correct way)

`new RestClient("...")` constructs **its own** `HttpClient` internally — bypassing your factory pool, defeating DNS rotation, and one-by-one re-creating sockets per `RestClient` instance. Always pass in a factory-managed `HttpClient`:

```csharp
builder.Services.AddHttpClient("github", c =>
{
    c.BaseAddress = new("https://api.github.com/");
    c.DefaultRequestHeaders.UserAgent.ParseAdd("acme/1.0");
})
.AddStandardResilienceHandler();    // wrap RestSharp with the resilience pipeline at the HttpClient level

builder.Services.AddSingleton(sp =>
{
    var http = sp.GetRequiredService<IHttpClientFactory>().CreateClient("github");
    return new RestClient(http, disposeHttpClient: false);   // factory owns disposal
});
```

`disposeHttpClient: false` is essential: the pooled handler outlives the `RestClient` wrapper and `IHttpClientFactory` recycles it on its own schedule. Letting RestSharp dispose it would tear down a pooled, shared resource.

## Authenticators

```csharp
var options = new RestClientOptions("https://api.example.com")
{
    Authenticator = new JwtAuthenticator(jwt),                  // sets Authorization on every request
    // Authenticator = new HttpBasicAuthenticator(user, pass),
    // Authenticator = new OAuth1Authenticator.ForRequestToken(consumerKey, consumerSecret),
};
var client = new RestClient(httpClient, options, disposeHttpClient: false);
```

OAuth1 is one of the few areas where RestSharp meaningfully wins — Microsoft's stack offers no first-party OAuth1 helper.

## Streaming downloads

```csharp
var request = new RestRequest("/exports/large.csv");
await using var stream = await client.DownloadStreamAsync(request, ct)
    ?? throw new InvalidOperationException("empty response");

await using var file = File.Create("large.csv");
await stream.CopyToAsync(file, ct);
```

`DownloadStreamAsync` returns the raw `Stream` without buffering, equivalent to `HttpClient.GetAsync(..., HttpCompletionOption.ResponseHeadersRead)`.

## RestSharp vs raw HttpClient vs Refit

| Aspect | Raw `HttpClient` | **RestSharp** | Refit |
|---|---|---|---|
| Style | Imperative | Imperative, fluent | Declarative interface |
| Compile-time URL/param checking | ✗ | ✗ | ✓ |
| Source-generated / AOT-safe | ✓ | ✗ (runtime reflection) | ✓ (with generator) |
| Easy mocking | needs `HttpMessageHandler` fake | needs interface wrapper | mock the interface directly |
| Built-in OAuth1 | ✗ | ✓ | ✗ |
| Built-in resilience | needs `Microsoft.Extensions.Http.Resilience` | needs same | needs same |
| JSON helpers | `ReadFromJsonAsync` / `PostAsJsonAsync` | `AddJsonBody` / typed `GetAsync<T>` | automatic |
| Footprint | zero extra deps | extra dep, larger surface | extra dep, generates code |

## Senior-level gotchas

- **Never** construct `RestClient` from a URL string in production code — it bypasses `IHttpClientFactory`. Always inject the factory's `HttpClient` and pass `disposeHttpClient: false`.
- The v106 API (`IRestClient.Execute<T>(IRestRequest)` synchronous, `RestSharp.Newtonsoft.Json` separate package) is **everywhere on Stack Overflow** but materially different from v110+. Verify any sample matches the major version on your `RestSharp` package reference.
- v111+ defaults to `System.Text.Json`. If you need Newtonsoft (e.g. for `JsonConvert` attributes on shared DTOs), install `RestSharp.Serializers.NewtonsoftJson` and call `.UseNewtonsoftJson()` on the options.
- `RestResponse.IsSuccessful` is `IsSuccessStatusCode && ResponseStatus == ResponseStatus.Completed` — already covers transport errors. Don't manually combine it with another check.
- `AddJsonBody(null)` serializes the literal string `"null"`, which some servers reject with 400/415. Guard with `if (body is not null) request.AddJsonBody(body);`.
- RestSharp uses runtime reflection for parameter binding — **not AOT-friendly**, will fail at runtime under NativeAOT or aggressive trimming.
- `RestClient` is **thread-safe** for sending requests but mutating `DefaultParameters` mid-flight is not. Configure once at construction.
- Errors are not thrown by default — `RestResponse.ErrorException` carries them. Either check `IsSuccessful` after every call or wrap the client in a thin helper that throws. The `ThrowOnAnyError = true` option turns this on globally.
- For high-throughput services, consider whether RestSharp earns its weight. A typed `HttpClient` wrapper is ~30 lines and avoids the dependency entirely.
