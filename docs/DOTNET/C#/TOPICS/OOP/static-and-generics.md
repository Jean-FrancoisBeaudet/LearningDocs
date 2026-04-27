# Static and generics

_Targets .NET 10 / C# 14. See also: [Classes, records, structs](./classes-records-structs.md), [Interfaces, abstract, sealed, anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md), [Constructors, properties, fields, methods](./constructors-properties-fields-methods.md), [Volatile and Interlocked](../THREAD_SYNCHRONIZATION/volatile-and-interlocked.md)._

`static` and generics seem unrelated until you remember they share one big property: **the CLR specialises code per type**. Each closed generic instantiation gets its own static field slots, its own static ctor invocation, and (for value types) its own JIT-compiled native code. Misunderstanding this leads to subtle bugs (one `static` that you thought was singleton-per-process is actually one-per-`T`) and wasted memory (every value-type instantiation pays for its own JIT'd method bodies).

## `static` classes

```csharp
public static class Math
{
    public static double Square(double x) => x * x;
}
```

A `static` class compiles to IL with both `abstract` and `sealed` flags — no instances, no derivation. Cannot have instance members, instance ctors, or be used as a generic type argument. The compiler turns every member access into a direct call (no `this`).

The two legitimate uses:
1. Pure utility namespace (`Math`, `File`, `Path`).
2. Container for extension methods.

If you find a `static class` accumulating mutable state, it's a code smell — `static` mutable state is process-global, hard to test, hard to reason about under concurrency, and a memory leak waiting to happen (it roots whatever it references for the AppDomain's lifetime).

## `static` members on non-static types

```csharp
public sealed class CounterService
{
    private static int _instances;
    public CounterService() => Interlocked.Increment(ref _instances);
    public static int Instances => _instances;
}
```

Static fields belong to the **closed** type. For `Foo<T>`, each `T` gets its own slot:

```csharp
public sealed class Foo<T>
{
    public static int Count;
}

Foo<int>.Count = 5;
Foo<string>.Count = 10;
Console.WriteLine(Foo<int>.Count);     // 5
Console.WriteLine(Foo<string>.Count);  // 10
```

This is the runtime's "reified generics" at work — different from Java's erasure, where `Foo<Integer>.Count` and `Foo<String>.Count` would share a slot.

### Static ctor ordering and thread safety

Already covered in [Constructors, properties, fields, methods](./constructors-properties-fields-methods.md#static-constructors). Key points:
- Runs once per closed generic type, on first member access.
- Thread-safe by the runtime (type-init lock).
- `beforefieldinit` IL flag (when no explicit static ctor is written) lets the runtime defer static-field init until the field is actually read — adding an explicit static ctor removes the flag and forces eager init.
- Throwing in a static ctor poisons the type permanently with `TypeInitializationException`.

### `[ThreadStatic]` and `AsyncLocal<T>`

```csharp
[ThreadStatic] private static StringBuilder? _scratch;

public static StringBuilder Scratch =>
    _scratch ??= new StringBuilder(256);
```

`[ThreadStatic]` gives one slot per thread — perfect for thread-local scratch buffers, dangerous in async code (the value doesn't flow across `await`). For async, use `AsyncLocal<T>`:

```csharp
private static readonly AsyncLocal<string?> _correlationId = new();

public static string? CorrelationId
{
    get => _correlationId.Value;
    set => _correlationId.Value = value;
}
```

`AsyncLocal<T>` flows with the `ExecutionContext` across `await`, `Task.Run`, and `ThreadPool` callbacks. Costs an `ExecutionContext` capture on every async transition — non-zero in hot paths.

## Static local functions

```csharp
public int ComputeChecksum(ReadOnlySpan<byte> data)
{
    int seed = 0xABCD;
    return Reduce(data, seed);

    static int Reduce(ReadOnlySpan<byte> d, int s)
    {
        int acc = s;
        foreach (var b in d) acc = (acc * 31) ^ b;
        return acc;
    }
}
```

The `static` modifier on a local function means it **cannot capture** outer locals. The benefit isn't readability — it's that the compiler doesn't allocate a closure object on each call. A non-static local function used in a hot path is a silent allocation per invocation. Default to `static` and pass state explicitly; the analyzer IDE0062 suggests it.

## Generics

### Generic types and methods

```csharp
public sealed class Box<T>
{
    public T Value { get; }
    public Box(T value) => Value = value;
}

public static T First<T>(IEnumerable<T> source) =>
    source.GetEnumerator().MoveNext()
        ? source.First()
        : throw new InvalidOperationException();
```

Generic methods can be inferred at the call site (`First(myList)`); generic types cannot — `Box(5)` doesn't compile, you must write `Box<int>(5)` or use a static factory.

### Generic constraints (overview)

```csharp
public T CreateCopy<T>(T input) where T : class, ICloneable, new()
    => (T)input.Clone();
```

Full coverage in [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md). The set in summary:

- `class`, `class?`, `struct`, `notnull`, `unmanaged`
- `new()`
- `BaseClass`, `IInterface`, `T2`
- `enum`, `delegate`
- `allows ref struct` (C# 13)

### Variance — `in` and `out`

```csharp
public interface IProducer<out T> { T Get(); }       // covariant
public interface IConsumer<in T>  { void Use(T v); } // contravariant
public interface IBoth<T>         { T M(T v); }      // invariant

IProducer<string> ps = …;
IProducer<object> po = ps;   // ok — out

IConsumer<object> co = …;
IConsumer<string> cs = co;   // ok — in
```

Variance applies only to **interfaces** and **delegates**, only to **reference type** arguments (no `IList<int>` covariance), and only when `T` is used in a position consistent with its annotation:
- `out T`: only as a return type.
- `in T`: only as a parameter type.
- `T` (invariant): both.

The runtime enforces it via metadata flags; the compiler checks the consistency at declaration time.

`Func<>`, `Action<>`, `IEnumerable<>`, `IReadOnlyCollection<>`, `IComparer<>`, `IEqualityComparer<>` all use variance. `IList<T>` does not (you can read AND write).

### Generic instantiation cost — value types vs reference types

The CLR shares method bodies across all reference-type instantiations of a generic — so `List<string>`, `List<Order>`, `List<Customer>` all share the same JIT'd code. Each closed type has its own static fields and method table, but the IL/native code is single-instance.

Value-type instantiations get **their own JIT'd code** per closed type — `List<int>`, `List<long>`, `List<Guid>` are three sets of native methods. This is the cost of erasure-free generics (no boxing) but it's a real working-set cost in code-heavy generic libraries.

Implications:
- A library exposing `Foo<T>` with 100 value-type uses across the codebase pays 100× JIT'd-code cost.
- ReadyToRun / NativeAOT pre-compile a fixed set; "rare" instantiations may JIT at runtime.
- For "most-frequent value type" specialisations, hand-written wrappers (`IntFoo`, `LongFoo`) sometimes still win — e.g. that's why .NET ships `System.Collections.Generic.IEnumerator<int>` inlined cases via `EnumerableInternal` helpers.

### Static abstract members on interfaces (recap)

Already covered in [Interfaces, abstract, sealed, anonymous](./interfaces-abstract-classes-sealed-anonymous-objects.md#static-abstract--static-virtual-interface-members-c-11). Mention here because they fundamentally change what generics can express — you can now write generic algorithms over types that have **shared static behaviour** (parsing, math, factory methods) without virtual dispatch:

```csharp
public static T SumAll<T>(IEnumerable<T> items) where T : INumber<T>
{
    T acc = T.Zero;
    foreach (var i in items) acc += i;
    return acc;
}

SumAll(new[] { 1, 2, 3 });          // int
SumAll(new[] { 1.5, 2.5, 3.5 });    // double
SumAll(new[] { 1m, 2m, 3m });       // decimal
```

The JIT specialises per closed `T`; the operator `+` becomes a direct opcode for primitive types — same speed as a hand-written `int Sum(int[]).`

### Generic methods vs generic delegates

```csharp
public static class Cache<TKey, TValue> where TKey : notnull
{
    private static readonly Dictionary<TKey, TValue> _store = new();
    public static TValue GetOrAdd(TKey key, Func<TKey, TValue> factory) =>
        _store.TryGetValue(key, out var v) ? v : _store[key] = factory(key);
}
```

`Func<TKey, TValue>` is invariant in `TKey` but covariant in `TValue`. So `Func<object, string>` is assignable to `Func<object, object>` but not to `Func<string, string>`. Subtle when designing public APIs; use the right delegate by intent.

## Senior-level gotchas

- **Static fields are per-closed-generic-type.** `MyClass<int>._cache` and `MyClass<string>._cache` are *different* fields. Singletons via `static` on a generic type aren't process-global.
- **Static ctor on a generic type runs once per closed `T`.** Heavy initialisation (file I/O, DB lookup) in a generic static ctor multiplies by the number of `T`'s used.
- **`static` mutable fields are unfenced** unless explicitly synchronised. A `public static int Counter;` mutated by multiple threads has unspecified behaviour without `Interlocked` / `volatile` / `lock`.
- **`[ThreadStatic]` does not flow across `await`.** Reading `[ThreadStatic]` in async code returns whichever pool thread happens to be running you. Use `AsyncLocal<T>` or pass state through method parameters.
- **`AsyncLocal<T>` mutation is `Set` — not "scope".** Setting it in an awaited method **replaces** the value for the rest of that flow; there's no nested-scope semantics. Use a stack-of-values pattern if you need scope.
- **Generic value-type instantiation cost** — every distinct `T` for a generic struct method gets its own JIT'd body. Avoid creating dozens of generic value-type instantiations in tight inner loops if startup time matters; prefer non-generic specialisations or limit the closed-set.
- **`new()` constraint calls `Activator.CreateInstance<T>()`** under the hood — slow and allocates. For high-throughput factories, take a `Func<T>` parameter instead, or use `T.Create(…)` via a static-abstract interface member.
- **Variance and arrays** — arrays are *covariant* on element type but unsafe: `string[] s = …; object[] o = s; o[0] = 5;` throws `ArrayTypeMismatchException` at runtime. Prefer `IReadOnlyList<T>` or `ReadOnlySpan<T>`.
- **Constraint inheritance** — when `class Derived<T> : Base<T> where T : new()`, the constraint must be repeated on the derived class even if the base imposed it. The compiler does not flow constraints through the hierarchy.
- **`static class` cannot implement an interface**, even an interface with only `static abstract` members. Reach for a sealed non-static class with private ctor + static-only API instead.
- **Static-abstract interface members are not callable through an interface reference** — `IFoo.M()` doesn't compile. Only `T.M()` from inside a generic constrained to `T : IFoo` works. This is by design (the dispatch needs to know which closed type).
- **Generic methods can't be `partial` and produce overloads** — partial method declarations and implementations must match exactly, including type parameter constraints.
- **`AsyncLocal<T>.Value` getter allocates** when capturing the `ExecutionContext` lazily on first read. Reading it in tight loops pays a small surcharge; cache the value in a local at the top of the method.
- **`static readonly` of a struct returns a copy on every read** unless you read through a `ref readonly` accessor. For large structs (e.g. config snapshots), `ref readonly` matters.
- **Generics + reflection** — `typeof(Foo<>)` is the open generic; `typeof(Foo<int>)` is closed. `MakeGenericType` constructs closed types at runtime; doing this in a hot path is slow (and trims badly under NativeAOT). Prefer source generators or `static abstract` interface members.
- **Type-parameter inference doesn't see constructor calls** — `new Box(5)` won't infer `T = int`; you must write `new Box<int>(5)` or supply a `Box.Create(5)` factory. C# 12+ has primary-ctor type inference for *records* via positional form, not for plain classes.
