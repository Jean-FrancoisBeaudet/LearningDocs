# Mapster

_Targets .NET 10 / C# 14. See also: [Manual mapping](./manual-mapping.md), [AutoMapper](./automapper.md), [Mapperly](./mapperly.md)._

`Mapster` (NuGet: `Mapster`, MIT licensed) is the most direct alternative to AutoMapper. It compiles mapping expressions on first use (faster than AutoMapper's reflection-based approach by ~3-5×), supports the same convention-driven model, and — uniquely — offers **two execution modes**: a runtime mode (familiar to AutoMapper users) and a **compile-time source generator** (`Mapster.Tool` / `[GenerateMapper]`) that competes head-on with [Mapperly](./mapperly.md). For teams migrating off AutoMapper post-license-change, Mapster is usually the lowest-friction landing spot — the API surface is intentionally similar and the migration is mostly mechanical.

## Wiring (runtime mode)

```csharp
// Package: Mapster + Mapster.DependencyInjection
builder.Services.AddSingleton(TypeAdapterConfig.GlobalSettings);
builder.Services.AddScoped<IMapper, ServiceMapper>();

// Configure once at startup (singleton)
TypeAdapterConfig<Product, ProductDto>.NewConfig()
    .Map(d => d.Price,    s => s.Price.Amount)
    .Map(d => d.Currency, s => s.Price.Currency.Code)
    .Map(d => d.InStock,  s => s.Stock > 0);

TypeAdapterConfig<CreateProductRequest, Product>.NewConfig()
    .Map(d => d.Price, s => new Money(s.Price, Currency.From(s.Currency)))
    .Ignore(d => d.Id);
```

`TypeAdapterConfig.GlobalSettings` is the global config registry. Configure it during startup, before any mapping call, and treat it as immutable thereafter — concurrent reconfiguration races against compiled-expression caches.

## Usage

```csharp
public sealed class ProductService(IMapper mapper, AppDb db)
{
    public async Task<ProductDto?> GetAsync(int id, CancellationToken ct)
    {
        var entity = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
        return entity?.Adapt<ProductDto>();              // shorthand for IMapper.Map
    }

    public Task<List<ProductDto>> ListAsync(CancellationToken ct) =>
        db.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .ProjectToType<ProductDto>()                 // ← server-side projection (translates to SQL)
            .ToListAsync(ct);
}
```

`Adapt<T>()` and `IMapper.Map<T>` are equivalent — they materialize the source then build the destination. `ProjectToType<T>()` is the projection equivalent, the same critical distinction as AutoMapper's [`Map` vs `ProjectTo`](./automapper.md#map-vs-projectto--the-distinction-that-matters): the former loads the whole entity, the latter pushes the projection into the SQL.

## Compile-time mode (the underrated feature)

Add `Mapster.Tool` and decorate a partial class:

```csharp
// Package: Mapster + Mapster.Tool (msbuild source generator)
[Mapper]
public partial class ProductMapper
{
    public partial ProductDto ToDto(Product src);
    public partial Product   ToEntity(CreateProductRequest src);
}
```

The generator emits a sibling `.g.cs` file (under `obj/`) containing fully expanded constructor and property assignments — no reflection, no expression trees, no runtime cost beyond the destination allocation. Behaviorally identical to [Mapperly](./mapperly.md) at this point.

```csharp
public sealed class ProductService(ProductMapper mapper, AppDb db)
{
    public async Task<ProductDto?> GetAsync(int id, CancellationToken ct)
    {
        var entity = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
        return entity is null ? null : mapper.ToDto(entity);
    }
}
```

You can mix modes in the same project — runtime mappings via `TypeAdapterConfig` for ad-hoc dynamic shapes, source-generated mappings for hot paths or AOT-targeted assemblies.

## Conventions and per-property overrides

```csharp
TypeAdapterConfig<Order, OrderDto>.NewConfig()
    .NameMatchingStrategy(NameMatchingStrategy.IgnoreCase)
    .Map(d => d.LineCount, s => s.Lines.Count)
    .Map(d => d.Total,     s => s.Lines.Sum(l => l.Quantity * l.UnitPrice))
    .AfterMapping((src, dst) => dst.ComputedAt = DateTimeOffset.UtcNow);
```

`AfterMapping` / `BeforeMapping` hooks run for both `Adapt` and projection (when projection-translatable) — useful but easy to abuse. If the hook can't be expressed as an expression, `ProjectToType` will silently fall back to in-memory evaluation; check generated SQL.

## When to use

- **Migrating off AutoMapper** because of the license change — the API is the closest analogue and migration is mostly find-and-replace + namespace swap.
- You want **both** runtime mapping (for dynamic scenarios) and compile-time mapping (for hot paths) in one library.
- You need projection support (`ProjectToType`) and dislike Mapperly's still-evolving `IQueryable` story.
- You like fluent-config style and don't want attribute-only configuration.

## When NOT to use

- Pure AOT or zero-reflection requirement → [Mapperly](./mapperly.md) is more rigorously source-gen-only and trims cleaner.
- You want the absolute minimum library surface area in your project → use [manual mapping](./manual-mapping.md).
- You're already on AutoMapper, the codebase is large, the license isn't an issue, and the team has no pain → don't migrate for migration's sake.

See the cross-comparison table in [manual-mapping.md](./manual-mapping.md#cross-comparison).

**Senior-level gotchas:**
- `TypeAdapterConfig.GlobalSettings` is mutable global state. Configure it once at startup and don't touch it again — late mutation races against the compiled-expression cache. For tests, build a per-test `TypeAdapterConfig` instance and inject it via `IMapper`/`Mapper(config)` rather than mutating the global.
- `Adapt<T>()` (extension method on `object`) bypasses the DI'd `IMapper`, which means it skips your `ServiceMapper`'s service-resolution path. If your maps depend on injected services, use `IMapper.Map<T>()` instead — `Adapt<T>()` will silently drop the service hooks.
- `ProjectToType<T>` and `Adapt<T>` use the same config but evaluate differently. A custom `MapWith(...)` that calls a method or accesses a private field works at runtime (`Adapt`) and throws at projection (`ProjectToType`) because EF can't translate it. Always test the projection path against a real database, not just unit tests.
- Generated source-gen code lives in `obj/Generated/Mapster.Tool/...`. **Read it.** When mapping behaves unexpectedly, the answer is usually one search-and-open away. The generator emits clean readable C# — there's no excuse to debug blind.
- `[GenerateMapper]` requires the destination type to be in the same project as the mapper, or the generator can't see it. Cross-project mapping means the mapper class lives where the destination type lives.
- Settings inheritance: `TypeAdapterConfig<DerivedSrc, DerivedDst>` does not automatically inherit from `TypeAdapterConfig<BaseSrc, BaseDst>`. Use `Inherits<...>()` explicitly. This bites every team adopting Mapster after a polymorphic AutoMapper config.
- `IgnoreNullValues(true)` is essential for `PATCH`-style update mappings; without it, every null property on the request silently overwrites the destination's existing value with null.
- Two-way mappings via `TwoWays()` have the same trap as AutoMapper's `ReverseMap` — convenient and usually wrong. Forward and reverse mappings have different rules; write them separately.
- The runtime mode compiles expression trees on first use, which means **first request after deploy is slower** than steady state. Warm critical maps in `IHostedService.StartAsync` if cold-start latency matters.
- For the source-gen mode, analyzer warnings (e.g. "no matching property on destination") are silenceable but shouldn't be — treat them as build errors via `<TreatWarningsAsErrors>` or per-warning `<WarningsAsErrors>`. A silent warning is a missed mapping in production.
- `Mapster.Async` exists for async mapping (when destinations need awaitable construction). Almost no one needs it; the existence of the package leads to confused decisions. Default to sync mappings; await *around* the mapping call, not inside it.
