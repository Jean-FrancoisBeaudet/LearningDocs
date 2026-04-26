# Refit

_Targets .NET 10 / C# 14. See also: [HttpClient & HttpClientFactory](../API_COMMUNICATION/httpclient-and-httpclientfactory.md), [RestSharp](./restsharp.md), [Polly](./polly.md), [Microsoft Resilience](./microsoft-resilience.md)._

`Refit` (NuGet: `Refit`, `Refit.HttpClientFactory`) turns a C# **interface** into a fully-wired typed `HttpClient`. You declare the contract once with attributes — `[Get("/users/{id}")]`, `[Post("/orders")]`, `[Body]`, `[Query]` — and Refit generates the implementation. It composes cleanly with `IHttpClientFactory`, `DelegatingHandler` chains, and the standard resilience pipeline. Pick it when your dependency is a stable HTTP API and you want a typed, mockable boundary instead of hand-rolling `GetFromJsonAsync` everywhere.

## The interface as the contract

```csharp
[Headers("User-Agent: acme/1.0", "Accept: application/json")]
public interface IGitHubApi
{
    [Get("/users/{login}")]
    Task<GitHubUser> GetUserAsync(string login, CancellationToken ct);

    [Get("/users/{login}/repos")]
    Task<IReadOnlyList<Repo>> GetReposAsync(
        string login,
        [Query] RepoFilter filter,
        CancellationToken ct);

    [Post("/repos/{owner}/{repo}/issues")]
    Task<Issue> CreateIssueAsync(
        string owner,
        string repo,
        [Body] NewIssue body,
        [Header("Authorization")] string bearerToken,
        CancellationToken ct);

    [Multipart]
    [Post("/upload")]
    Task UploadAsync(
        [AliasAs("file")] StreamPart file,
        [AliasAs("name")] string name,
        CancellationToken ct);

    // Non-throwing: inspect StatusCode/Error yourself
    [Get("/orgs/{org}")]
    Task<IApiResponse<Org>> TryGetOrgAsync(string org, CancellationToken ct);

    // Stream the raw response — no buffering
    [Get("/exports/{id}")]
    Task<HttpResponseMessage> DownloadExportAsync(string id, CancellationToken ct);
}

public sealed record RepoFilter(string Sort, [property: AliasAs("per_page")] int PerPage);
```

`AliasAs` is the only reliable way to rename query/form properties — `[JsonPropertyName]` does **not** apply to query strings. For collection params: `[Query(CollectionFormat.Multi)] string[] tags` produces `?tags=a&tags=b` (vs. `Csv`/`Pipes`/`Ssv`).

## DI wiring

```csharp
builder.Services
    .AddRefitClient<IGitHubApi>(new RefitSettings
    {
        ContentSerializer = new SystemTextJsonContentSerializer(new JsonSerializerOptions(JsonSerializerDefaults.Web)),
    })
    .ConfigureHttpClient(c => c.BaseAddress = new("https://api.github.com/"))   // trailing slash
    .AddHttpMessageHandler<AuthHeaderHandler>()
    .AddStandardResilienceHandler();
```

`AddRefitClient<T>` registers `T` as **transient** behind the factory-pooled `HttpMessageHandler` — same lifetime guarantees as a typed `HttpClient`. Inject `IGitHubApi` directly into your service; never resolve from `IServiceProvider`.

## Source-generated mode (AOT-friendly)

The default Refit emits IL at runtime via `Castle.DynamicProxy` — incompatible with NativeAOT and trim-unsafe. Switch to the source generator:

```xml
<!-- in csproj -->
<PackageReference Include="Refit.HttpClientFactory" Version="..." />
<PropertyGroup>
  <RefitInternalNamespace>Acme.Refit.Generated</RefitInternalNamespace>
</PropertyGroup>
```

The included `InterfaceStubGenerator` produces concrete classes at compile time — zero runtime reflection, AOT-safe, faster cold start. Mark interfaces you want generated with `[GenerateRefit]` if your project mixes both modes.

## Error handling

```csharp
try
{
    var user = await api.GetUserAsync("octocat", ct);
}
catch (ApiException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return null;
}
catch (ApiException ex)
{
    var problem = await ex.GetContentAsAsync<ProblemDetails>();   // typed error body
    logger.LogWarning("GitHub returned {Status}: {Title}", problem?.Status, problem?.Title);
    throw;
}
```

`ApiException` carries the status code, headers, and the raw response content (lazy-read). For non-throwing flows, return `IApiResponse` / `IApiResponse<T>`:

```csharp
var resp = await api.TryGetOrgAsync("anthropic", ct);
if (!resp.IsSuccessStatusCode) return Results.Problem(resp.Error?.Message);
return Results.Ok(resp.Content);
```

## Mocking

Because the contract is just an interface, mocking is trivial — no `HttpMessageHandler` fakes:

```csharp
var api = Substitute.For<IGitHubApi>();
api.GetUserAsync("octocat", Arg.Any<CancellationToken>())
   .Returns(new GitHubUser("octocat", 583231));

var sut = new MyService(api);
```

This is the single biggest reason to reach for Refit over a hand-written client.

## Senior-level gotchas

- `BaseAddress` needs a **trailing slash**. `new Uri("https://api.github.com", "users")` evaluates to `https://api.github.com/users` only if the base ends in `/` — otherwise the last segment is stripped. Same trap as raw `HttpClient`.
- `[Query]` on a complex object respects `[AliasAs]`, not `[JsonPropertyName]`. Mixing the two silently produces wrong query strings.
- Cancellation token must be the **last parameter** in the method signature. Refit doesn't bind it positionally elsewhere.
- `IApiResponse<T>` reads the **entire** response body to populate `Error.Content` even on 4xx/5xx. For huge error payloads (or streaming downloads with possible failure), return `HttpResponseMessage` directly and inspect manually.
- Default mode (dynamic proxy) **breaks NativeAOT and PublishTrimmed**. Switch to the source generator before publishing AOT — there's no warning, just runtime `MissingMethodException`.
- Method overloads on the same interface are not supported (the route attribute is the only key Refit can use). Give each operation a unique name.
- Refit registers as the **innermost** message handler in the factory's chain — a `DelegatingHandler` you add via `AddHttpMessageHandler` runs *outside* it and sees the request after Refit serialized it. If you need to mutate the body, do it before calling the interface, not in a handler.
- `[Headers]` on the interface sets defaults; `[Headers]` on a method appends; a method `[Header("X")]` parameter **replaces** any matching default for that call. Don't try to delete a header by passing `null` — pass `string.Empty` and the framework will drop it.
- `application/x-www-form-urlencoded` posts: use `[Body(BodySerializationMethod.UrlEncoded)] Dictionary<string, object>` (or a typed object). The url-encoded body method is *not* the default — `Json` is.
- For per-call `RefitSettings` (e.g. dynamic JSON options), use `RestService.For<T>(httpClient, settings)` directly — the DI registration takes one settings instance for the lifetime.
