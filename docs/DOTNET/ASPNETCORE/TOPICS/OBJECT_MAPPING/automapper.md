# AutoMapper

_Targets .NET 10 / C# 14. See also: [Manual mapping](./manual-mapping.md), [Mapster](./mapster.md), [Mapperly](./mapperly.md)._

`AutoMapper` (NuGet: `AutoMapper`) is the granddaddy of .NET mapping libraries — Jimmy Bogard's project that, for a decade, was the default answer to "how do I map `Entity → Dto`?". It uses **reflection** at startup to build a compiled expression tree per `Map<TSrc, TDst>`, then executes that at runtime. Convention-driven (matching property names auto-map), with `Profile` classes for explicit overrides.

## License change (read before adopting in 2026)

In **August 2024**, AutoMapper's owner announced that future versions move to a **commercial license**. Versions before the change remain free; versions after (v15+) require a paid license for commercial use. This single fact is the most important thing about AutoMapper today — it has caused widespread migration to [Mapster](./mapster.md) and [Mapperly](./mapperly.md) on greenfield projects. **Don't adopt AutoMapper for new code in 2026 unless your org has already paid for it.** Existing codebases on the old MIT-licensed versions can stay there; new code should pick one of the alternatives.

The rest of this note explains AutoMapper as it works, because you will inherit codebases using it for years to come.

## Wiring

```csharp
// Old: AutoMapper.Extensions.Microsoft.DependencyInjection
builder.Services.AddAutoMapper(cfg => cfg.AddMaps(typeof(Program).Assembly));
```

The DI extension scans assemblies for `Profile` subclasses and builds a single `IMapper` (singleton). Cold-start cost is non-trivial for large mapping graphs (hundreds of profiles → hundreds of milliseconds at first request).

## Profiles

```csharp
public sealed class ProductProfile : Profile
{
    public ProductProfile()
    {
        CreateMap<Product, ProductDto>()
            .ForMember(d => d.Currency, opt => opt.MapFrom(s => s.Price.Currency.Code))
            .ForMember(d => d.Price,    opt => opt.MapFrom(s => s.Price.Amount))
            .ForMember(d => d.InStock,  opt => opt.MapFrom(s => s.Stock > 0));

        CreateMap<CreateProductRequest, Product>()
            .ForMember(d => d.Price, opt => opt.MapFrom(s => new Money(s.Price, Currency.From(s.Currency))))
            .ForMember(d => d.Id,    opt => opt.Ignore());
    }
}
```

`CreateMap` is the unit of work. Conventions handle anything where source and destination property names match; `.ForMember(...)` covers the rest. `.Ignore()` is mandatory for unmapped destination members **only if** you've enabled strict configuration (recommended — see "gotchas").

## Usage in a service

```csharp
public sealed class ProductService(IMapper mapper, AppDb db)
{
    public async Task<ProductDto?> GetAsync(int id, CancellationToken ct)
    {
        var entity = await db.Products
            .Include(p => p.Price.Currency)   // ← have to remember this
            .FirstOrDefaultAsync(p => p.Id == id, ct);

        return entity is null ? null : mapper.Map<ProductDto>(entity);
    }

    public IAsyncEnumerable<ProductDto> ListActiveAsync(CancellationToken ct) =>
        db.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .ProjectTo<ProductDto>(mapper.ConfigurationProvider)   // ← server-side projection
            .AsAsyncEnumerable();
}
```

The two methods on the same class show AutoMapper's most important distinction: **`Map<T>` vs `ProjectTo<T>`**.

## `Map` vs `ProjectTo` — the distinction that matters

| | `mapper.Map<TDto>(entity)` | `queryable.ProjectTo<TDto>(config)` |
|---|---|---|
| Runs | After materialization | As part of the SQL |
| Loads | Whole entity + Includes | Only mapped columns |
| Allocations | Entity, then DTO | DTO only |
| Extra `Include` needed | Yes — for any nav touched in the map | No — projection drives the join |
| Use for | Single-entity / non-EF flows | List endpoints, queries, anything `IQueryable` |

The single most common AutoMapper performance bug: `db.Products.ToListAsync()` followed by `mapper.Map<List<ProductDto>>(entities)`. You loaded every column, hydrated change-tracker entries, then dropped 80% of the data. `ProjectTo` is what you wanted — it composes inside the `IQueryable` and EF translates the whole shape.

## Configuration validation in tests

```csharp
[Fact]
public void AutoMapper_configuration_is_valid()
{
    var config = new MapperConfiguration(cfg => cfg.AddMaps(typeof(ProductProfile).Assembly));
    config.AssertConfigurationIsValid();
}
```

Run this on every CI build. Without it, a missing `.ForMember` or a typoed property name turns into a runtime exception on the first request that touches that map — usually in production, usually on the worst possible endpoint.

## When to use (in 2026)

- You inherited a codebase that already uses AutoMapper at scale and rewriting all profiles isn't worth the churn.
- Your org has a commercial license already in place.
- You have deep convention-mapping needs across hundreds of similar shapes where the convention engine genuinely saves work.

## When NOT to use

- **Greenfield work in 2026.** License + reflection cost + AOT incompatibility = pick [Mapperly](./mapperly.md) or [Mapster](./mapster.md).
- NativeAOT — AutoMapper relies on runtime expression compilation that AOT can't trim correctly.
- Hot paths or memory-constrained services. Runtime mapping costs ~3-10× a hand-written mapper.
- Small mapping surface (< 20 DTOs). The library overhead and the licensing risk dwarf the boilerplate it saves.
- Anywhere `ProjectTo` doesn't apply (non-EF) and you need cluster-fast object→object mapping — [Mapperly](./mapperly.md) wins.

See the cross-comparison table in [manual-mapping.md](./manual-mapping.md#cross-comparison).

**Senior-level gotchas:**
- **The license change is the headline.** Audit your `packages.lock.json` for `AutoMapper` ≥ 15. If it's there, your legal/procurement team needs to know. The MIT-licensed earlier versions are still installable but you're freezing on a deprecated branch.
- `mapper.Map(...)` vs `queryable.ProjectTo<T>(...)`: same shape, opposite performance characteristics. Reviewers should reflexively flag any `db.X.ToListAsync()` followed by `mapper.Map`.
- `ProjectTo` only supports a subset of mapping features: no `IValueResolver`, no `IMappingAction`, no instance constructors with logic, no `ConvertUsing` with non-trivial expressions. The mapping engine doesn't tell you at config time which maps are projection-safe — you find out when the `IQueryable` throws.
- `AssertConfigurationIsValid()` does not catch projection-incompatible maps. There's a separate `IsCompiledProjection` check via `cfg.Internal()` API in newer versions; use it if you rely on `ProjectTo`.
- Circular references: `Order` → `Customer` → `Orders[]` → `Customer` → ... recurses to stack overflow. Configure `MaxDepth(N)` or `PreserveReferences()` — both have perf cost. Better: design DTOs as DAGs, not graphs.
- Reverse mappings via `.ReverseMap()` look convenient and are usually wrong. Two-way mapping has different validation rules in each direction (e.g., `Id` is generated on create, supplied on update). Write explicit forward and reverse maps.
- `IMapper` is registered as singleton, but the `MapperConfiguration` is what holds the compiled trees. Don't `new MapperConfiguration` per request — it triggers the cold-start scan again.
- Cold-start cost for hundreds of profiles is hundreds of ms. For Function/Lambda workloads with cold starts, this is visible in p99 latency. `[ModuleInitializer]`-warmed configuration is one mitigation; switching to a source-gen mapper is the other.
- Custom `ITypeConverter<TSrc, TDst>` and `IValueResolver` types must be registered with the DI container (or AutoMapper's internal container). Forgotten registrations turn into runtime "no service for type X" errors mid-request.
- `IgnoreAllPropertiesWithAnInaccessibleSetter()` and `ShouldMapProperty` look like nice escape hatches and reliably break either projection or symmetric mapping. Avoid configuration tricks; if a property shouldn't map, change the DTO.
- AutoMapper's exception messages are notoriously unhelpful for missing-member errors. Read the *innermost* exception — the outer one usually says "AutoMapper failed" without naming the field.
- For DI-tied profiles (resolvers that need services), `.ConstructUsing(...)` and `.ConvertUsing(typeof(MyConverter))` form the pattern, but the DI container must resolve the converter — make sure your `AddAutoMapper` call wires up a service-provider-aware mapper.
