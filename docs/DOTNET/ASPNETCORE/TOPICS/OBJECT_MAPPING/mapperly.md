# Mapperly

_Targets .NET 10 / C# 14. See also: [Manual mapping](./manual-mapping.md), [AutoMapper](./automapper.md), [Mapster](./mapster.md)._

`Mapperly` (NuGet: `Riok.Mapperly`, Apache 2.0) is a **source-generator-only** mapper. There is no runtime engine, no reflection, no expression compilation — at build time the Roslyn analyzer reads your `[Mapper]`-decorated partial class and emits the mapping methods as ordinary C# in `obj/Generated/`. The result is indistinguishable from a hand-written mapper at runtime: zero reflection, zero startup cost, fully NativeAOT-compatible, and cheap to debug because the generated code is readable C#. For greenfield .NET 8+ services in 2026, Mapperly is the default recommendation.

## The whole library in one example

```csharp
[Mapper]
public partial class ProductMapper
{
    public partial ProductDto ToDto(Product src);
    public partial Product    ToEntity(CreateProductRequest src);
    public partial void       UpdateEntity(UpdateProductRequest src, Product target);
}
```

That's it. The generator produces:

```csharp
// obj/Generated/Riok.Mapperly/.../ProductMapper.g.cs (excerpt)
public partial class ProductMapper
{
    public partial ProductDto ToDto(Product src) => new ProductDto
    {
        Id    = src.Id,
        Name  = src.Name,
        Price = src.Price,                   // (assuming name + type match)
        // ...
    };
    // etc.
}
```

If a destination property has no matching source — or the types don't convert cleanly — the analyzer raises a **build-time error** (`RMG020`, `RMG004`, etc.). You discover the problem the moment you save the file, not in production.

## Wiring

```csharp
// Just register the mapper as a service — there's no Mapperly-side configuration
builder.Services.AddSingleton<ProductMapper>();
```

No `AddMapperly()`, no profile scanning, no global config. The mapper class is just a class.

## Customization via attributes

```csharp
[Mapper]
public partial class ProductMapper
{
    [MapProperty(nameof(Product.Price.Amount),         nameof(ProductDto.Price))]
    [MapProperty(nameof(Product.Price.Currency.Code),  nameof(ProductDto.Currency))]
    [MapperIgnoreSource(nameof(Product.InternalNotes))]
    [MapperIgnoreTarget(nameof(ProductDto.ServerTime))]
    public partial ProductDto ToDto(Product src);

    // Hand-write what the generator can't infer; called automatically.
    private bool MapInStock(int stock) => stock > 0;
}
```

Anything the generator can't express, you write as a private method with the matching signature — Mapperly's convention picks it up. This is the escape hatch that keeps the model simple: there's no DSL, no fluent config; if attributes don't cover it, write the method.

## Update mapping (in-place mutation)

```csharp
[Mapper(EnumMappingStrategy = EnumMappingStrategy.ByName)]
public partial class ProductMapper
{
    [MapperIgnoreTarget(nameof(Product.Id))]
    [MapperIgnoreTarget(nameof(Product.CreatedAt))]
    public partial void Apply(UpdateProductRequest src, Product target);
}
```

`partial void` with two parameters → update mapping. Use `[MapperRequiredMapping(RequiredMappingStrategy.None)]` and `IgnoreNullValues = true` on the `[Mapper]` attribute for `PATCH`-style "ignore null source values" semantics.

## Collections and nested types

Collections work without configuration — `List<Product>` → `List<ProductDto>` generates a loop that calls the per-element mapper. Nested types (a `ProductDto` containing a `CategoryDto`) require Mapperly to find a matching method in the same mapper class; declare both `partial Product → ProductDto` and `partial Category → CategoryDto` on the same `[Mapper]` class and they wire together automatically.

## `IQueryable` projection

```csharp
[Mapper]
public partial class ProductMapper
{
    public partial IQueryable<ProductDto> ProjectToDto(IQueryable<Product> source);
}
```

The generator emits a `Select(...)` projection, which EF Core translates to SQL — same column-pruning win as `ProjectTo`/`ProjectToType` on the other libraries. There are feature limits compared to runtime mappers (no custom methods, only expression-translatable mappings), but for the 80% case of "flatten an entity into a DTO," it's transparent.

## Usage in an endpoint

```csharp
app.MapGet("/products", async (
    ProductMapper mapper,
    AppDb db,
    CancellationToken ct) =>
{
    var items = await mapper
        .ProjectToDto(db.Products.AsNoTracking().Where(p => p.IsActive))
        .ToListAsync(ct);

    return Results.Ok(items);
});
```

The endpoint emits SQL that selects only the columns appearing in `ProductDto`. No `Include`, no entity hydration, no extra allocations.

## When to use

- Greenfield .NET 8+ services. The combination of zero runtime cost, AOT support, compile-time errors for missing mappings, and readable generated code is hard to beat.
- NativeAOT — Mapperly was designed for this. No reflection means no trim warnings.
- Performance-sensitive services where every allocation matters.
- Teams that value compile-time guarantees over runtime flexibility.

## When NOT to use

- You need to swap mapping configurations at runtime (rare — usually a sign of overdesign). Mapperly is compile-time-only.
- You're heavily invested in AutoMapper-style profile inheritance with deep polymorphism — Mapperly handles inheritance but the model is less expressive than runtime libraries.
- `IQueryable` projection with custom value resolvers that can't be translated to SQL — the runtime libraries' fallback to in-memory eval is more forgiving (and more dangerous; see their gotchas).

See the cross-comparison table in [manual-mapping.md](./manual-mapping.md#cross-comparison).

**Senior-level gotchas:**
- The mapper class **must** be `partial`, the methods **must** be `partial`, and the file containing it must be in a project that references `Riok.Mapperly`. Forget any one of these and you get either a missing-method link error or "interface method has no implementation" at runtime.
- Generated code lives in `obj/Generated/Riok.Mapperly/<assembly>/<mapper>.g.cs`. **Read it when in doubt** — it's plain C# you'd be comfortable code-reviewing. This is the killer debugging story Mapperly has over reflection-based libraries.
- Analyzer warnings are real bugs. `RMG020` ("source property not mapped to any target") and `RMG004` ("ignored target property has no mapping") are the most common. Treat them as errors via `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` or whitelist with `<WarningsAsErrors>RMG020;RMG004;...</WarningsAsErrors>`.
- Constructor mapping with multiple constructors: Mapperly picks one heuristically and is sometimes wrong. Annotate the intended ctor with `[MapperConstructor]` to be explicit. Records with positional parameters work seamlessly.
- Enum mapping defaults to `ByValue` — same underlying integer value, different enum names. If your source `Status.Active = 1` and target `OrderState.Pending = 1`, that maps silently. Set `EnumMappingStrategy = EnumMappingStrategy.ByName` on the `[Mapper]` attribute for safer string-name semantics, or `[MapEnum(...)]` per case.
- Nullability awareness is **strict**. Mapping `string` → `string?` is fine; `string?` → `string` raises a warning unless you use `[MapperIgnoreSource]` or supply a default via `[MapPropertyFromSource("...", Use = nameof(GetDefault))]`. This is a feature; embrace it.
- Update mapping (`partial void Apply(Src, Dst)`) silently overwrites every target property by default. Set `IgnoreNullValues = true` on `[Mapper]` for `PATCH`-style semantics, or per-method via `[MapperIgnoreSource(...)]` to skip selected source properties.
- Object initializer vs property assignment: Mapperly prefers an object initializer (`new T { ... }`) when the type has a parameterless ctor and writable setters. For init-only properties (`init`), it uses the same syntax but at runtime the assignment happens during construction — works as expected.
- DI hooks: Mapperly mappers can have constructors. If your mapping needs an injected service (e.g., a localizer for enum names), declare `public ProductMapper(IStringLocalizer loc)` and call it from a private mapping helper method. Register the mapper as `Scoped` or `Singleton` accordingly.
- `IQueryable` projection methods cannot use private helper methods — only expression-translatable code reaches the database. The generator will warn if you mix the two; pay attention.
- Mapperly is **case-sensitive on property names** by default. `Product.SKU` → `ProductDto.Sku` does not match. Use `[MapProperty(...)]` to bridge or rename one side.
- Migrating from AutoMapper/Mapster: the typical pattern is mapper-class-per-aggregate. A `Profile` with 30 `CreateMap` calls becomes a single partial class with 30 `partial` method declarations. The generator does the rest.
- For libraries you publish to NuGet that include Mapperly mappers, consumers don't need to reference Mapperly themselves — the source generator runs at *your* build time. The generated code is what ships.
