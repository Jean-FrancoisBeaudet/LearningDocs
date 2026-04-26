# Closure captures

_Targets .NET 10 / C# 14. See also: [Allocations and boxing](./allocations-and-boxing.md), [Low-allocation patterns in modern .NET](./low-allocation-patterns-modern-dotnet.md), [LINQ performance misuse](./linq-performance-misuse.md), [GC pressure](./gc-pressure.md), [Heap allocation patterns](./heap-allocation-patterns.md)._

A closure is a lambda that uses a variable from its enclosing scope. The compiler implements that capture by lifting the variable into a private compiler-generated **display class**, and the lambda becomes an instance method on that class. Every time control flow re-enters the scope where the closure is created, **the display class is `new`-ed up**. On a hot path, that allocation is invisible in the source and ruinous in the profiler.

## What the compiler actually emits

```csharp
int threshold = 10;
items.Where(x => x.Value > threshold);
```

Compiles to roughly:

```csharp
sealed class <>c__DisplayClass0_0
{
    public int threshold;
    public bool <Method>b__0(Item x) => x.Value > threshold;
}

var c = new <>c__DisplayClass0_0();   // ALLOCATION
c.threshold = 10;
items.Where(c.<Method>b__0);          // ALLOCATION (delegate)
```

Two allocations per call: the display class and the `Func<Item, bool>` delegate. Inside a per-request loop hit a million times a day, that's two million Gen-0 objects for one filter expression.

## The static-cache fast path — and what defeats it

A lambda that captures **nothing** (or only static state) is hoisted by the compiler to a static field on a synthesized `<>c` class. It allocates **once for the lifetime of the AppDomain**.

```csharp
items.Where(x => x.Value > 10);   // alloc-free — literal, no capture
```

Add **one** captured local and you lose the cache:

```csharp
int threshold = config.Threshold;
items.Where(x => x.Value > threshold);   // alloc per call site invocation
```

This is why the seemingly-trivial change "make this constant configurable" can quietly add measurable allocation pressure.

## Static lambdas (C# 9): make capture a compile error

The `static` modifier on a lambda forbids capture; the compiler errors instead of silently allocating:

```csharp
items.Where(static x => x.Value > 10);                     // ok, alloc-free
items.Where(static x => x.Value > threshold);              // CS8820: cannot capture `threshold`
```

Use it as a **deliberate guard** on hot-path lambdas. If the codebase later adds a capture, the build breaks instead of the perf regressing in production. Pair with the `state` parameter pattern below to pass values without capturing.

## The `state` parameter pattern

Most modern BCL APIs that accept a delegate also accept an opaque `TState` you can hand the lambda — exactly so you can use a `static` lambda and avoid the closure:

```csharp
// Bad: closure over `tenant` allocates every call.
_dict.GetOrAdd(key, k => Load(k, tenant));

// Good: tenant flows through the state arg; lambda is static.
_dict.GetOrAdd(key, static (k, t) => Load(k, t), tenant);
```

The same shape on `Task.Run`, `Task.Factory.StartNew`, `string.Create`, `ThreadPool.QueueUserWorkItem`, `ConcurrentDictionary.GetOrAdd`, `ConcurrentDictionary.AddOrUpdate`, `ImmutableInterlocked.Update`, and many more. If the API has a `TState` overload, prefer it.

```csharp
// string.Create — no closure, no intermediate string.
string trace = string.Create(20, (id, code), static (span, s) =>
{
    span[0] = '#';
    s.id.TryFormat(span[1..], out int w);
    span[1 + w] = '/';
    s.code.TryFormat(span[(2 + w)..], out _);
});
```

## Loop-variable capture trap

`foreach` over `IEnumerable` gives you a **fresh variable per iteration** (since C# 5), so this is safe:

```csharp
foreach (var item in items)
    tasks.Add(Task.Run(() => Process(item)));   // each task captures its own `item`
```

`for` does **not**. Each iteration shares the same `i`:

```csharp
for (int i = 0; i < 10; i++)
    tasks.Add(Task.Run(() => Console.WriteLine(i)));   // prints 10 ten times
```

Either copy to a local first (`int local = i; ... () => local`) or use the state-arg overload (`Task.Run(static s => Console.WriteLine(s), (object)i)`).

## LINQ in a hot loop

LINQ is the highest-volume closure source in real codebases:

```csharp
foreach (var batch in batches)
{
    var matches = items.Where(x => x.BatchId == batch.Id).ToList();
    // each iteration: 1 display class + 1 delegate + 1 List<T> + N enumerator allocs
}
```

In a tight loop, lift the predicate out, use a plain `foreach` with a guard, or precompute a lookup:

```csharp
var byBatch = items.ToLookup(x => x.BatchId);    // one alloc total
foreach (var batch in batches)
    foreach (var match in byBatch[batch.Id]) { ... }
```

The same trap hits **EF Core** especially hard: a captured local in a `Where` expression changes the compiled-query cache key per-instance, defeating the query plan cache and forcing re-translation. Use parameter-only predicates or the `EF.CompileAsyncQuery` family for hot queries.

## Async lambdas: closure × state machine

```csharp
foreach (var url in urls)
    _ = Task.Run(async () => await client.GetAsync(url, ct));
```

Per iteration: the closure (display class + delegate) **and** the async state machine box (`AsyncStateMachineBox<T>` on the heap). For a fan-out that touches a million URLs over a day, that's millions of Gen-0 objects for orchestration alone.

Restructure with `Parallel.ForEachAsync` (.NET 6+), which uses a worker pool instead of one task per item:

```csharp
await Parallel.ForEachAsync(urls, ct, async (url, t) => await client.GetAsync(url, t));
```

The lambda still captures `client`, but it's invoked from a fixed pool of workers — the closure is amortized.

## Spotting them

| Signal | Tool |
|---|---|
| Per-call display-class allocation | BenchmarkDotNet `[MemoryDiagnoser]` — non-zero `Allocated` on a "no-alloc" benchmark |
| Display classes by type name | PerfView **GC Heap Alloc Stacks**, filter `<>c__DisplayClass*` |
| Closure call-site analysis | `Microsoft.CodeAnalysis.PerformanceSensitiveAnalyzers` — `HAA0301` (closure alloc), `HAA0302` (display class), `HAA0603` (delegate cache miss), `HAA0604` (lambda → delegate alloc) |
| Hot-path lambda audit | Roslyn IDE warning **IDE0301**+: prefer `static` lambdas where possible |

The HAA analyzers (NuGet: `Microsoft.CodeAnalysis.PerformanceSensitiveAnalyzers`) only flag methods marked `[PerformanceSensitive]` — a one-time annotation per hot method gives you allocation hygiene as a build-time check.

## Pick the right tool

| Pattern | Allocation per invocation | When |
|---|---|---|
| `items.Where(x => x.Value > 10)` (no capture) | **0** (cached static delegate) | Always preferred when no state needs to flow |
| `items.Where(static x => x.Value > 10)` | **0**, with build-time guarantee | Hot paths where capture must be prevented |
| `items.Where(x => x.Value > threshold)` (capture) | display class + delegate | Cold paths only |
| `dict.GetOrAdd(k, static (k, s) => Load(k, s), state)` | **0** | Hot dictionary lookups, factory APIs |
| Hand-written class implementing the predicate | **0** after construction | Hot paths where the predicate has interesting state |

## Senior-level gotchas

- **A closure with no instance capture is allocation-free** thanks to the cached `<>c` singleton. Adding `this.` or any local-from-the-method changes the math entirely. The capture set is computed per lambda — don't trust intuition; check the IL with `sharplab.io` or ILSpy.
- **Captures lift the variable, not the value**: every closure over the same scope shares the *same* display class instance and sees subsequent mutations. This is why `for (int i = ...)` prints ten 10s and `foreach (var item in ...)` doesn't — the `foreach` iteration variable is per-iteration since C# 5.
- **Closures retain their captures for the delegate's lifetime**, not the surrounding method's. A long-lived event subscription with `obj.Event += () => Use(local)` keeps `local` (and its display class) alive until the unsubscription. Classic memory-leak vector.
- **`static` lambda is a guarantee, not a hint**. If the build compiles, capture is impossible. If a future change introduces a capture, you get CS8820 — not a perf regression that nobody notices for six months.
- **EF Core query expressions hash captured locals into the compiled-query cache key**. A `Where(x => x.Id == localId)` with different `localId` values compiles to a single cached plan; a `Where(x => x.Tenant == _injectedTenant)` from a per-request DbContext compiles to a *new* plan per scope unless `_injectedTenant` is reference-equal across requests. Watch for `Microsoft.EntityFrameworkCore.Query` warnings about query plan cache pollution.
- **Method groups are still delegate allocations**: `items.Where(IsValid)` allocates a `Func<Item, bool>` wrapping `IsValid` per call. The compiler caches it only when `IsValid` is static AND the call site is hot enough for the JIT's tiered compilation to notice. Don't assume "no lambda" means "no allocation."
- **`async` lambdas allocate twice on the heap per invocation**: the closure plus the state machine box. The fix is structural — extract the body to a named `async Task` method and pass state via a non-capturing wrapper.
- **`ConcurrentDictionary.GetOrAdd(k, valueFactory)` may call the factory more than once under concurrent races for the same key**. Combine that with a closure and you get N display-class allocations for one logical "add." The `(k, state)` overload + `static` lambda is the standard fix.
- **Captures inside expression-bodied properties run *every read*** — `public Func<int, bool> IsBig => x => x > _threshold;` allocates a closure per property access, not per object lifetime. If callers cache the return value, fine; if they re-read, you've made a hot-path leak. Convert to a cached field initialized once.
- **A closure over `this` is one capture, not zero** — even `() => Process(x)` inside an instance method allocates a display class that holds the instance reference. The cached static-singleton path applies only when the lambda truly references nothing from the outer scope.
- **Don't fight the compiler on cold paths**. Configuration loading, error handling, startup wiring — closures here are free in the profiler. Reserve `static` lambdas and state-arg gymnastics for the per-request hot loop you actually measured.
