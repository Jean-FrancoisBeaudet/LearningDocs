# Manual mapping

_Targets .NET 10 / C# 14. See also: [AutoMapper](./automapper.md), [Mapster](./mapster.md), [Mapperly](./mapperly.md), [Dapper](../DATA_ACCESS/dapper.md)._

"Manual mapping" means writing the `Entity → Dto` (or `Request → Command`) translation by hand — extension methods, constructor calls, or LINQ `Select` projections — instead of pulling in a mapping library. It's the baseline every other approach is measured against. After ~15 years of mapping libraries dominating the conversation, manual mapping has quietly become the default again for greenfield .NET services because (a) source generators and `record` types make it cheap, (b) AOT cares about reflection, and (c) the AutoMapper license change made teams audit their mapping deps.

## The four shapes

```csharp
// 1. Extension method — the "ToDto()" pattern
public static class ProductMappings
{
    public static ProductDto ToDto(this Product p) => new(
        Id:        p.Id,
        Name:      p.Name,
        Price:     p.Price.Amount,
        Currency:  p.Price.Currency.Code,
        InStock:   p.Stock > 0);
}

// 2. Constructor mapping (record DTO does the work)
public sealed record ProductDto(int Id, string Name, decimal Price, string Currency, bool InStock)
{
    public static ProductDto From(Product p) =>
        new(p.Id, p.Name, p.Price.Amount, p.Price.Currency.Code, p.Stock > 0);
}

// 3. IQueryable projection — translated to SQL by EF Core
public static IQueryable<ProductDto> ProjectToDto(this IQueryable<Product> source) =>
    source.Select(p => new ProductDto(
        p.Id, p.Name, p.Price.Amount, p.Price.Currency.Code, p.Stock > 0));

// 4. Update mapping (existing target, mutate in place)
public static void ApplyTo(this UpdateProductRequest req, Product target)
{
    target.Name  = req.Name;
    target.Price = new Money(req.Price, Currency.From(req.Currency));
}
```

Pick (1) for one-way object→object after materialization, (3) for EF queries (this is the one that matters for performance), (4) for `PUT`/`PATCH` against a tracked entity.

## End-to-end: minimal API + EF projection

```csharp
app.MapGet("/products", async (
    AppDb db,
    int page,
    int size,
    CancellationToken ct) =>
{
    var items = await db.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Id)
        .Skip(page * size).Take(size)
        .ProjectToDto()                    // ← SQL projects only the columns you mapped
        .ToListAsync(ct);

    return Results.Ok(items);
});
```

The whole `ProductDto` shape is encoded in the `Select`, so EF Core emits `SELECT Id, Name, Price_Amount, Price_Currency_Code, (Stock > 0) AS InStock FROM Products WHERE ...` — no `Include`, no extra navigation hydration, no ChangeTracker overhead. This is the single biggest win over runtime mapping libraries: `IMapper.Map(entity)` first **materializes the whole entity** then drops fields, while a manual `Select` lets the database return only what you need.

## When to use

- Greenfield .NET services where you'd rather have explicit code than another dep.
- AOT-compiled apps (manual mapping has no reflection — the compiler is the only "engine" needed).
- Hot paths where every allocation matters.
- Small mapping surface (≤ ~20 DTOs); the boilerplate stays manageable.
- Code reviews where reviewers want to *see* what's being mapped, not infer it from conventions.

## When NOT to use

- Large CRUD-heavy apps with dozens of similar `Entity → Dto` shapes — boilerplate compounds. Reach for [Mapperly](./mapperly.md) (still no reflection, just no boilerplate).
- Symmetric two-way mapping with fuzzy convention matching — manual code makes the asymmetry visible, which is usually a feature but occasionally tedious.
- When you've already standardized on a library across services and consistency is worth more than the per-file ergonomics.

## Cross-comparison

| Approach | Runtime cost | NativeAOT | License | Compile-time safety | Best fit |
|---|---|---|---|---|---|
| **Manual** | None | ✓ | n/a | ✓ (compiler) | Small surface, hot paths, AOT |
| [AutoMapper](./automapper.md) | Reflection (cached) | ✗ practical | **Commercial v15+** | ✗ (runtime) | Legacy codebases |
| [Mapster](./mapster.md) (runtime) | Reflection (compiled expr) | partial | MIT | ✗ (runtime) | AutoMapper migration |
| [Mapster](./mapster.md) (source-gen) | None | ✓ | MIT | ✓ | Greenfield, both modes available |
| [Mapperly](./mapperly.md) | None | ✓ | Apache 2.0 | ✓ (compiler) | Greenfield, AOT, perf |

The other three notes cross-reference back to this table rather than duplicating it.

**Senior-level gotchas:**
- **`.ToList().Select(...)` materializes twice.** This is the #1 manual-mapping bug: someone calls `.ToListAsync()` on the entity, then maps in memory, paying the `Include` tax for navigations they never returned. Always project **inside** the `IQueryable`, before terminal operators.
- `Select(p => p.ToDto())` does **not** translate to SQL — EF can't translate arbitrary method calls. Either inline the projection (option 3 above) or write a `ProjectToDto` extension that returns `IQueryable<TDto>` and use that. The compiler won't catch the mistake; you'll just see "EF loaded the whole entity" in your SQL profiler.
- Constructor mapping with `record` DTOs gives you compile-time enforcement that every property is mapped — add a property to the record, every existing call site fails to compile until you supply it. This is the underrated benefit over libraries.
- For `PUT`-style updates, beware "null means clear" vs "null means don't change" — manual update mappers make the distinction explicit (`if (req.Email is not null) target.Email = req.Email;`), which libraries often muddle.
- Owned types (`Money`, `Address`) project cleanly through manual mapping but are a configuration headache in AutoMapper/Mapster — manual is often the lesser pain there.
- When the boilerplate gets oppressive, the right answer is usually [Mapperly](./mapperly.md) or [Mapster source-gen](./mapster.md), **not** AutoMapper. You give up nothing about explicitness — the generated code is yours to read, and it's still allocation-free at runtime.
- Don't write a "mapping framework" inside your codebase. The first abstract `IMapper<TSrc, TDst>` interface you introduce is the moment you've reinvented half a library — go pick one or stay manual.
- Test your projections by snapshotting the SQL (`db.Products.ProjectToDto().ToQueryString()` in EF Core 7+). A manual mapping that *compiles* can still emit unexpected SQL — `ToQueryString()` makes the actual query visible without running it.
- Keep mapping code in a `Mapping/` folder per feature, not in a global `Mappings.cs`. Co-locating with the DTO and request types means deletes are easy and search-result noise stays low.
- For request validation that runs at projection time, see [FluentValidation](../VALIDATION/fluent-validation.md) — keep validation **separate** from mapping. A mapper that "also validates" couples two concerns and surprises future readers.
