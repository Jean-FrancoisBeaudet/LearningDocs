# Reference types

_Targets .NET 10 / C# 14. See also: [Value types](./value-types.md), [Classes, records, structs](../OOP/classes-records-structs.md), [Encapsulation, inheritance, polymorphism](../OOP/encapsulation-inheritance-polymorphism.md), [Interfaces, abstract, sealed, anonymous](../OOP/interfaces-abstract-classes-sealed-anonymous-objects.md), [Default interface members](../GENERAL_TOPICS/default-interface-members.md), [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md), [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md), [Allocations and boxing](../HEALTH_AND_PERFORMANCE/allocations-and-boxing.md)._

A **reference type** lives on the GC heap; the variable holds a *managed pointer* to it. Assignment copies the pointer, not the bytes. Two variables can refer to the same instance, mutation through one is visible through the other, and the lifetime of the object is decided by the GC, not by scope. This is the default shape for everything with identity, polymorphism, or non-trivial size.

## What counts as a reference type

- `class` — including `static` classes (no instances) and `abstract` classes.
- `interface` — implemented by classes and structs; the interface variable itself is a reference.
- `delegate` — multicast invocation list, heap-allocated.
- `record class` (the default `record`) — class with synthesised value-equality.
- Arrays — `T[]`, `T[,]`, jagged `T[][]`. Always reference types, even for value-type elements (`int[]` is a reference; the elements are inline value types inside the array object).
- `string` — sealed reference type with value-like equality and immutability.
- `object` — the universal base; every type ultimately derives from it.
- A *boxed* value type — at runtime indistinguishable from any other reference type.

## Object header and heap layout

On a 64-bit runtime, every heap object carries:

- **Sync block index** — 8 bytes. Used for `lock`, `Monitor`, hash code stability for unmodified-`object.GetHashCode`, COM interop.
- **Method table pointer (MT)** — 8 bytes. Points to the type's vtable, interface map, and metadata. Drives virtual dispatch and `is`/`as`.
- **Payload** — fields, padded to alignment.
- **Padding** — minimum object size on 64-bit is 24 bytes.

`new object()` is therefore 24 bytes of overhead + 0 of payload. A `class Foo { int X; }` is also 24 bytes (header + 4 byte field + 4 byte padding). Useful for back-of-envelope GC pressure math.

```csharp
class Empty { }
// Unsafe.SizeOf<Empty>() — invalid; you can't take size of a reference type via Unsafe
// GC.GetAllocatedBytesForCurrentThread() — measures actual allocation cost in tests
```

## Storage and lifetime

The variable lives wherever any local lives — stack, register, captured-class field, static field. The object lives on the **GC heap**:

- **SOH (Small Object Heap)** — gen0 / gen1 / gen2.
- **LOH (Large Object Heap)** — objects ≥ 85 KB. Background-only collection on .NET; fragmentation matters.
- **POH (Pinned Object Heap)** — .NET 5+. Long-lived pinned buffers that you'd otherwise have to fragment gen2 with.

See [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md) for promotion, pinning, and DATAS. From the type-system perspective, the contract is: **reachability** keeps the object alive, **unreachability** makes it eligible for collection on the next pass that targets that generation.

## Aliasing and mutation

```csharp
var a = new Customer("Ada");
var b = a;        // b and a refer to the same instance
b.Name = "Lovelace";
Console.WriteLine(a.Name);   // "Lovelace"
```

This is the defining property and the most common source of "spooky action at a distance" bugs in mature codebases. Defensive copying (clone), immutability (`record` with `init`-only properties), or restricted mutability (private setters + invariant-preserving methods) are the three ways out.

## Identity vs equality

For a plain `class`:

```csharp
class Point { public int X; public int Y; }
var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };
p1 == p2;                       // false — reference comparison
p1.Equals(p2);                  // false — Object.Equals default is ReferenceEquals
ReferenceEquals(p1, p2);        // false
```

Override `Equals` when the type *has* a value-meaning identity that should override reference identity — but **rarely**, because the better default is to make it a `record class`:

```csharp
record class Point(int X, int Y);
new Point(1,2) == new Point(1,2);   // true
```

If you override `Equals(object)`, override:

1. `Equals(object?)` — the canonical override.
2. `GetHashCode()` — the contract is "equal objects must have equal hash codes". CA1815 / CA2218 will warn.
3. `==` and `!=` operators (optional but expected for value-meaning types).
4. `IEquatable<T>.Equals(T)` — strongly-typed, avoids casts and boxing for collection use.

`record class` synthesises all four; see [classes, records, structs](../OOP/classes-records-structs.md) for the generated members.

## `null` and Nullable Reference Types (NRT)

Reference types are intrinsically nullable in the CLR — *every* reference can be `null`. NRTs (`#nullable enable`, on by default in modern templates) add **compile-time** flow analysis:

```csharp
public string FindName(int id)        // T  — non-nullable
public string? TryFindName(int id)    // T? — nullable
public string? Maybe(string s) =>
    s.Length > 0 ? s : null;          // OK

string s = null;          // CS8600 warning
string? s2 = null;        // OK
Console.WriteLine(s2.Length);  // CS8602 — possible null deref
Console.WriteLine(s2!.Length); // suppressed; you've asserted non-null
```

NRTs are **annotations**, not runtime guarantees. The CLR will still happily store `null` in a `string` field through reflection, deserialisation, or interop. Use `ArgumentNullException.ThrowIfNull` at boundaries.

Useful nullability-flow attributes (`System.Diagnostics.CodeAnalysis`):

- `[NotNull]`, `[MaybeNull]` — output direction.
- `[NotNullWhen(true)]`, `[NotNullWhen(false)]` — for `Try…` patterns.
- `[MemberNotNull(nameof(Field))]` — on a method that initialises a non-nullable field.
- `[DisallowNull]`, `[AllowNull]` — input direction for `T?`/`T` mismatches.

```csharp
public bool TryGetUser(int id, [NotNullWhen(true)] out User? user) { … }

if (TryGetUser(7, out var u))
    Console.WriteLine(u.Name);   // u is User, not User?, here
```

## Inheritance and virtual dispatch

```csharp
public abstract class Shape { public abstract double Area { get; } }
public sealed class Square(double side) : Shape
{
    public override double Area => side * side;
}
```

- Single base class, multiple interfaces. (`object` is the implicit base if you specify none.)
- `virtual` / `override` — vtable-driven dispatch.
- `sealed` on a class blocks further subclassing; `sealed override` on a method blocks further overrides. **Default to `sealed class`** — JIT can devirtualise call sites.
- `new` (member hiding) — almost never the right answer; signals a design problem.
- `abstract` members force concrete subclasses to provide an implementation.
- `protected internal` — accessible to derivatives *or* same-assembly. `private protected` — derivatives *and* same-assembly.

Detail: [Encapsulation, inheritance, polymorphism](../OOP/encapsulation-inheritance-polymorphism.md), [Interfaces, abstract, sealed, anonymous](../OOP/interfaces-abstract-classes-sealed-anonymous-objects.md).

## Interfaces

Dispatched via the **interface map** in the method table. Every interface call is, in IR terms, a two-step: lookup interface slot in MT, jump to implementation.

```csharp
public interface ILogger
{
    void Log(string message);
    void Log(LogLevel level, string message) => Log($"[{level}] {message}");  // DIM (C# 8)
}
```

Default interface members (DIMs) let interfaces evolve without breaking implementers — see [Default interface members](../GENERAL_TOPICS/default-interface-members.md). Note: DIMs cannot access state, only other interface members.

Implementing an interface on a *struct* boxes it when you cast to the interface — one of the main reasons interfaces sit on the reference-type page. Generic constraints (`where T : ILogger`) avoid the box: the JIT specialises per `T`.

## `string`

A sealed reference type with semantics that look value-like:

- **Immutable** — every operation returns a new string. `string.Concat`, `string.Replace`, interpolation, `StringBuilder.ToString()` — all allocate.
- **Value equality** — `==` is overloaded to call `string.Equals`.
- **Interning** — string literals are pooled into the intern table. `string.Intern` lets you opt non-literals in, but the table is process-lifetime and never collected.
- **Hashing** — randomised per-process by default for hash-flooding resistance, so `"foo".GetHashCode()` differs across runs. Use a stable hash (`StringComparer.Ordinal.GetHashCode`) only when you control comparison.

Common allocation traps: concatenation in loops (use `StringBuilder` or `string.Create`), `ToLower`/`ToUpper` for comparison (use `StringComparer.OrdinalIgnoreCase`), `Substring` (use `AsSpan(start, length)` and a `ReadOnlySpan<char>` API).

## Arrays

```csharp
int[]   a = new int[10];          // reference type holding 10 inline ints
Point[] p = new Point[10];        // reference type holding 10 inline Points
string[] s = new string[10];      // reference type holding 10 references
int[,] m = new int[10, 10];       // multi-dimensional, contiguous
int[][] j = new int[10][];        // jagged: array of arrays
```

Arrays are **covariant** for reference-type element types — a long-standing CLR misfeature:

```csharp
string[] strings = new[] { "a", "b" };
object[] objects = strings;            // legal at compile time
objects[0] = 42;                       // ArrayTypeMismatchException at runtime
```

`List<T>` is *invariant*; prefer `IReadOnlyList<T>` / `Span<T>` / `ReadOnlySpan<T>` for safer covariant-ish APIs (`ReadOnlySpan<T>` is invariant; `IReadOnlyList<T>` *is* covariant in `T`).

Array length is `int` (max ~2.1 G elements), but byte size is gated by `<gcAllowVeryLargeObjects>` and the LOH limit. Arrays ≥ 85 KB land on the LOH.

## Delegates

```csharp
Action<int>      log = i => Console.WriteLine(i);
Func<int, bool>  isEven = i => i % 2 == 0;
EventHandler<T>  onTick = (s, e) => { … };
```

A delegate instance is a heap object containing:

- A target reference (or `null` for `static` methods on closures-free delegates).
- A method pointer.
- A multicast list (chained delegates).

Capturing a non-static lambda allocates a **closure class** + a **delegate instance** + (sometimes) a **`<>c__DisplayClass`** per call site. Static lambdas (no captures, marked `static` since C# 9) can be cached and avoid both:

```csharp
list.Where(static x => x > 0);   // no closure, cached delegate
```

Delegates that capture `this` are the most common cause of unintended object retention: an event subscription holds the subscriber alive until the subscription is released.

## Boxing — the bridge to value types

A boxed value type *is* a reference type instance: same object header, same heap layout, same GC rules. The `box` IL instruction allocates a heap object whose payload is a copy of the value-type bytes. `unbox.any` copies them back out.

```csharp
int n = 5;
object o = n;             // box — heap allocation
int n2 = (int)o;          // unbox — copy back
o.GetType();              // typeof(int), not typeof(object)
```

Boxing detail: [Allocations and boxing](../HEALTH_AND_PERFORMANCE/allocations-and-boxing.md).

## GC implications

- **Allocation cost** — bump-the-pointer in the gen0 nursery, very cheap (~tens of ns per allocation), until a collection is triggered.
- **Promotion** — surviving objects get promoted to gen1, then gen2. Gen2 collection scans the *entire* heap and is the expensive one.
- **LOH** — ≥ 85 KB objects skip generations and land on the LOH. They're collected only with gen2 (background since .NET Core 3.x).
- **Finalizers** — types with a finalizer are placed on the **finalization queue** at allocation; their reclamation takes *two* GC cycles (one to find them, one to free them after the finalizer runs). Cross-link: [Finalizer and Dispose pattern](../MEMORY_MANAGEMENT/finalizer-and-dispose-pattern.md).
- **Pinning** — `fixed` blocks and `GCHandle.Alloc(…, GCHandleType.Pinned)` block compaction in the affected segment. Long-lived pins fragment the heap; use the POH or `ArrayPool` instead.

A reference type with no payload still costs 24 bytes plus a slot in whichever generation it lives in. "Free" objects don't exist in CLR economics.

## `class` vs `record class` quick reference

| Need                                            | Pick                                       |
|-------------------------------------------------|--------------------------------------------|
| Identity, mutable state, behaviour-rich         | `sealed class`                             |
| Identity, polymorphic hierarchy                 | `abstract class` + `sealed` leaves         |
| Value-equality, immutable DTO, easy `with` copy | `record class`                             |
| Polymorphic root + closed variants              | `abstract record class` + sealed leaf records |

Depth and the synthesised members: [classes, records, structs](../OOP/classes-records-structs.md).

## Generic constraints

```csharp
where T : class            // T is a (non-nullable) reference type
where T : class?           // T is a reference type, nullable allowed
where T : notnull          // T is value type or non-null reference (excludes T?)
where T : new()            // T has a public parameterless ctor
where T : Foo              // T derives from Foo
where T : IFoo             // T implements IFoo
where T : U                // T derives from another type parameter U
```

Notes:

- `class` excludes `Nullable<U>` (which is a struct).
- `class?` matters for libraries that return `T?` and want to handle `T = string?` cleanly.
- `notnull` is *advisory*: the compiler warns but the CLR has no representation of "non-null reference".

See [Generic constraints and variance](../GENERAL_TOPICS/generic-constraints-and-variance.md).

## Senior-level gotchas

- **`object.GetHashCode` default is sync-block-based** — stable per instance, unrelated to fields. Putting the same logical entity into a new instance breaks `HashSet<T>` lookups. Override `Equals` + `GetHashCode` together, or use a `record class`.
- **Array covariance is a runtime trap.** `string[] → object[] → assign int? No, ArrayTypeMismatchException.` Lint with `dotnet_diagnostic.CA2208` and prefer `IReadOnlyList<T>`/`ReadOnlySpan<T>` at API boundaries.
- **Captured `this` retains the enclosing instance.** Long-lived subscriptions to events on a singleton are a classic memory-leak vector — the singleton's invocation list keeps every subscriber alive. Use `static` lambdas, `WeakReference<T>`, or explicit unsubscription patterns.
- **`record class.with` calls the protected copy constructor**, not the primary constructor. Validation in the primary ctor is *not* re-run. Put invariants in property `init` setters or write a `record` with a custom copy ctor.
- **`==` on a `record class` calls the synthesised structural `Equals`**, but `==` on `object` is reference identity. `(object)recordA == (object)recordB` will be `false` for two equal records — operator dispatch is a *compile-time* concept.
- **Reference identity equality + override pitfalls.** Override `Equals` without `GetHashCode` and the analyser will warn (CS0659/CA1815/CA2218). The CLR will still let it run — and will silently lose dictionary entries when keys are mutated.
- **`string.Intern` retains forever.** Useful for legitimately bounded keysets, catastrophic for arbitrary user input — the intern pool is process-lifetime, gen2-rooted, never collected.
- **`Type` instances are eternal** in non-collectible `AssemblyLoadContext`s. Every `typeof(T)`, `obj.GetType()`, `Type.GetType` is rooted by the runtime metadata. Cache them; do not retry-unload them.
- **Finalizers run on a single dedicated finalizer thread.** A blocking finalizer stalls *every* finalisable object in the process. Implement `IDisposable` and rely on the finalizer only as a last-resort safety net for unmanaged resources.
- **`async void` allocations leak through unobserved exceptions.** The state machine is a heap class; the unobserved exception escapes to `AppDomain.UnhandledException`. Use `async Task` everywhere except event handlers.
- **Reference assignment is not atomic for non-aligned references.** On x86/x64 it is in practice; on weakly-ordered architectures (ARM, prior to .NET 6 ARM normalisations) you can see torn or stale reads. Use `Volatile.Read`/`Volatile.Write` or `Interlocked.Exchange<T>` for cross-thread reference publication.
- **A `class` field with no initialiser starts as `null`.** Combined with NRTs, you'll get warnings unless you initialise in the ctor or use `[MemberNotNull(nameof(Field))]` on a helper. `required` properties (C# 11) close most of the gap.
- **Boxing a `Nullable<T>` with `HasValue == false` produces `null`.** Re-unboxing a non-null boxed `T` to `T?` works. This is the only "magic" the runtime does at the box layer.
- **`object.MemberwiseClone` is a shallow bitwise copy** — references are shared, value-type fields are duplicated. Almost never what you actually want; write the clone you mean.
- **Interface dispatch on a struct via a generic constraint avoids boxing**, but interface dispatch on a struct stored in an `IFoo` *field* boxes once at the assignment and forever after. Profile both paths if they're hot.
- **Delegates compare by target + method**, not by reference. Two `Action`s wrapping the same static method are `==`. Not so for instance methods unless the target reference matches.
- **`record class` with a mutable field bypasses structural equality if you write the field manually** — synthesised `Equals` only covers properties from the primary constructor. Mark extra state with `init`-only properties to keep equality coherent.
