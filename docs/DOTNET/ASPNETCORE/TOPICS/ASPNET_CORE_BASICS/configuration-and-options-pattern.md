# Configuration and Options Pattern

_Targets .NET 10 / C# 14. See also: [BackgroundService](./background-service.md), [Authentication and Authorization](./authentication-and-authorization.md), [Logging](./logging.md)._

`IConfiguration` is the layered key/value bag the host builds from `appsettings.json`, environment variables, command-line args, user secrets, and any `IConfigurationProvider` you bolt on. The **Options pattern** is the strongly-typed projection of that bag into POCOs your code injects via DI — and crucially, it gives you three lifetimes (`IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>`) with very different reload semantics. Picking the wrong one is the #1 reason "my config changes don't apply" tickets exist.

## Sources and precedence

`WebApplication.CreateBuilder` registers, in order (later overrides earlier):

1. `appsettings.json`
2. `appsettings.{Environment}.json` (e.g. `appsettings.Production.json`)
3. **User secrets** — only when `Environment == "Development"` and the project has a `UserSecretsId`
4. **Environment variables** — `__` (double underscore) is the section separator
5. **Command-line args** — `--Section:Key value` or `Section:Key=value`

Add Key Vault, AWS Parameter Store, etcd, or anything else after `CreateBuilder`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
    new DefaultAzureCredential());
```

Read raw values:

```csharp
string conn = builder.Configuration.GetConnectionString("Db") 
              ?? throw new InvalidOperationException("ConnectionStrings:Db missing");

int    port = builder.Configuration.GetValue<int>("Server:Port", defaultValue: 5000);

var section = builder.Configuration.GetRequiredSection("Email");   // throws if missing
```

`GetValue<T>` returns `default(T)` for missing keys — silently. Prefer `GetRequiredSection().Get<T>()` or `GetValue<T>("...") ?? throw ...` for keys that must exist.

## Strongly-typed binding: the Options pattern

```csharp
public sealed class EmailOptions
{
    public const string SectionName = "Email";

    [Required] public required string SmtpHost { get; init; }
    [Range(1, 65535)] public int SmtpPort { get; init; } = 587;
    [Required, EmailAddress] public required string From { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

builder.Services
    .AddOptions<EmailOptions>()
    .Bind(builder.Configuration.GetSection(EmailOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();                                        // fail-fast at boot if invalid
```

The shorthand `services.Configure<EmailOptions>(config.GetSection("Email"))` is equivalent but skips validation hooks — prefer the fluent form when you want validation.

## Three lifetimes — pick deliberately

```csharp
public sealed class EmailService(
    IOptions<EmailOptions>         optsStartupSnapshot,
    IOptionsSnapshot<EmailOptions> optsPerRequest,
    IOptionsMonitor<EmailOptions>  optsLive) { /* ... */ }
```

| Type | Lifetime | Reload behavior | Inject into |
|---|---|---|---|
| `IOptions<T>` | Singleton | Snapshot at first resolution; never reloads | Anything (singletons OK) |
| `IOptionsSnapshot<T>` | **Scoped** | Re-bound per scope (per HTTP request) | Scoped/transient services |
| `IOptionsMonitor<T>` | Singleton | Push-based; `OnChange` callback fires on source change | Anything; **only choice for singletons that need fresh values** |

**The captive dependency trap**: injecting `IOptionsSnapshot<T>` (scoped) into a singleton (`BackgroundService`, `IHttpClientFactory` typed client, `IHostedService`) silently captures the *startup* scope's instance. You believe you're getting fresh values per "iteration" — you're not. Use `IOptionsMonitor<T>` for singletons; reserve `IOptionsSnapshot<T>` for per-request scoped consumers.

```csharp
// In a BackgroundService — read CurrentValue every iteration
public sealed class Worker(IOptionsMonitor<EmailOptions> opts) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var current = opts.CurrentValue;                  // re-read each tick
            await DoWork(current, ct);
            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}
```

`OnChange` fires on every source notification — for `appsettings.json` that's every file write, including the editor's autosave that touches the file twice per keypress. Debounce if your handler is non-trivial:

```csharp
opts.OnChange(value =>
{
    // Debounce: only react if value actually changed
    if (!EqualityComparer<EmailOptions>.Default.Equals(value, _last))
    {
        _last = value;
        log.LogInformation("Email options changed: SmtpHost={Host}", value.SmtpHost);
    }
});
```

## Validation: data annotations, FluentValidation, custom

```csharp
builder.Services.AddOptions<EmailOptions>()
    .Bind(builder.Configuration.GetSection("Email"))
    .ValidateDataAnnotations()                                    // [Required], [Range], [EmailAddress], etc.
    .Validate(o => o.Timeout > TimeSpan.Zero, "Timeout must be positive")
    .ValidateOnStart();                                           // throws OptionsValidationException at boot
```

Without `ValidateOnStart()`, validation runs *lazily* on first resolution — meaning a misconfigured singleton might not throw until the first HTTP request, mid-traffic. Always pair `Validate*` with `ValidateOnStart`.

For complex rules, use FluentValidation via the `Microsoft.Extensions.Options.FluentValidation`-style integration:

```csharp
public sealed class EmailOptionsValidator : AbstractValidator<EmailOptions>
{
    public EmailOptionsValidator()
    {
        RuleFor(x => x.SmtpHost).NotEmpty();
        RuleFor(x => x.From).EmailAddress();
        RuleFor(x => x.SmtpPort).InclusiveBetween(1, 65535);
    }
}

builder.Services.AddSingleton<IValidateOptions<EmailOptions>, FluentValidateOptions<EmailOptions>>();
```

## Named options

When you have *multiple instances* of the same options type (per-tenant SMTP, per-environment HTTP client, per-feature flag set):

```csharp
builder.Services.Configure<EmailOptions>("primary",   config.GetSection("Email:Primary"));
builder.Services.Configure<EmailOptions>("backup",    config.GetSection("Email:Backup"));

public sealed class EmailService(IOptionsMonitor<EmailOptions> opts)
{
    public Task SendPrimary(...) => Send(opts.Get("primary"), ...);
    public Task SendBackup (...) => Send(opts.Get("backup"),  ...);
}
```

Named options are how `IHttpClientFactory` powers per-client configuration (`AddHttpClient("Acme", c => ...)`).

## Reload-on-change

`AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)` is the default in `CreateBuilder`. The host watches for file changes and re-binds. **Key Vault and most cloud sources don't push changes** — they require explicit polling intervals (`AzureKeyVaultConfigurationOptions.ReloadInterval = TimeSpan.FromMinutes(5)`).

For environment variables: there's no "watcher" — env vars are read once at startup, period. Restart the process to pick up changes (or use `IConfiguration` provider that polls a backing store).

## Custom `IConfigurationProvider`

When you need a config source the framework doesn't ship — feature flag service, a database table of toggles, a Git-backed repo:

```csharp
public sealed class DatabaseConfigurationSource(string conn) : IConfigurationSource
{
    public IConfigurationProvider Build(IConfigurationBuilder builder)
        => new DatabaseConfigurationProvider(conn);
}

public sealed class DatabaseConfigurationProvider(string conn) : ConfigurationProvider
{
    public override void Load()
    {
        // Populate Data dictionary; framework wraps it as IConfiguration
        Data = LoadFromDb(conn);
    }
}

builder.Configuration.Add(new DatabaseConfigurationSource(connString));
```

For most use cases, prefer existing libraries (`Microsoft.FeatureManagement`, `Polly` for transient resilience around lookups) over rolling your own provider.

## `builder.Configuration` vs `app.Configuration`

`WebApplicationBuilder.Configuration` is mutable until `builder.Build()` is called; after that, `WebApplication.Configuration` is the read-only view. Add config sources before `Build()`; resolve values either before or after.

## Senior-level gotchas

- **`IOptions<T>` snapshot is forever.** Editing `appsettings.json` won't change values seen by an `IOptions<T>`-injected service. Switch to `IOptionsMonitor<T>` (singleton) or `IOptionsSnapshot<T>` (scoped) for live reload.
- **`IOptionsSnapshot<T>` in a singleton = captive dependency.** The container resolves it once, in the *root* scope, and you get the startup-time values forever — the worst kind of bug because the *type system* says you have reload semantics. Use `IOptionsMonitor<T>` for singletons.
- **`OnChange` fires multiple times per file save.** Editors write twice (truncate + write) and `FileSystemWatcher` raises events for both. Debounce or compare values before reacting.
- **Environment variables use `__` (double underscore) as the separator on every platform** — including Windows, even though `:` works in JSON. Cross-platform deployment scripts always use `__`: `Email__SmtpHost=mx.example.com`.
- **`GetValue<T>` for a missing key returns `default(T)`.** Silent failure: a missing `MaxRetries` becomes `0` and your `for (var i = 0; i < opts.MaxRetries; i++)` never runs. Prefer `GetRequiredSection().Get<T>()` and `[Required]` on the POCO.
- **`Bind` does not validate.** Even with `[Required]` on the POCO, `services.Configure<T>(section)` happily binds missing fields to `default`. `ValidateDataAnnotations().ValidateOnStart()` is the only fail-fast path.
- **Secrets in `appsettings.json` are the #1 leak vector.** Even on private repos, IDE telemetry, browser extensions, and accidental git pushes leak them. Use `dotnet user-secrets` for dev, Key Vault / Parameter Store for prod. `appsettings.json` should commit-clean.
- **`required` members on a POCO need a parameterless constructor** for `Bind` to instantiate. Or use `[SetsRequiredMembers]` on the chosen constructor. C# 11 `required` semantics interact awkwardly with reflection-based binders.
- **Validation runs lazily without `ValidateOnStart`.** A misconfigured singleton can boot fine and then throw `OptionsValidationException` mid-request. Always `ValidateOnStart()` for boot-time fail-fast.
- **`GetSection(...)` always returns a section** — even one that doesn't exist (it's just empty). `section.Exists()` is the explicit check; `GetRequiredSection` throws on missing.
- **Hierarchical override is per-key, not per-section.** If `appsettings.json` has `{ Email: { SmtpHost: "x", SmtpPort: 25 }}` and env vars set `Email__SmtpHost=y`, the merged result is `{ SmtpHost: "y", SmtpPort: 25 }` — you don't lose `SmtpPort`. Useful for partial overrides; surprising if you expected env vars to fully replace the section.
- **`IConfigurationProvider.Load()` is sync.** Async-loaded sources (HTTP, Key Vault) must block synchronously inside `Load`, which can hang boot if the source is unreachable. Wrap with timeout/retry, or load asynchronously into a memory provider after boot.
- **`IOptionsMonitor<T>.CurrentValue` is a property access, not a snapshot.** Each read can return a different instance after a reload. Don't cache it; re-read at the top of every iteration.
- **`OptionsBuilder<T>.PostConfigure(Action<T>)` runs after `Configure` and after binding** — great for derived defaults (e.g. `o.Timeout ??= TimeSpan.FromSeconds(30)`), but runs *before* validation. Don't `PostConfigure` to "fix" invalid input, fix the input.
