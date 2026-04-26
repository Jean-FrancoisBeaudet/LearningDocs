# Allocations and boxing

_Targets .NET 10 / C# 14. See also: [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md), [Heap allocation patterns](./heap-allocation-patterns.md), [Closure captures](./closure-captures.md), [Low-allocation patterns](./low-allocation-patterns-modern-dotnet.md)._

An **allocation** in .NET means asking the runtime to carve memory out of the managed heap. Every allocation eventually feeds the GC: the more you do, the more often Gen0 fires, the more cache lines you dirty, the higher your pause-time tail. **Boxing** is the most insidious kind of allocation, because the C# source code does not visibly say `new`. Knowing where the compiler emits a `box` instruction is the difference between a clean trace and a Gen0 storm.

## What counts as an allocation

The CLR has two storage classes:

| Storage | Cost | Who collects it |
|---|---|---|
| Stack frame slot | Free (just bumping `esp`) | Frame teardown on return |
| Managed heap object | Bump-pointer alloc + future GC scan/copy | The GC |

The IL opcode that produces a heap allocation is `newobj` (for classes) or `newarr` (for arrays). **Every** `new ClassName(...)` and `new T[...]` emits one of these — there is no escape analysis in the CLR JIT. (RyuJIT does limited stack-allocate-objects work for small known cases, but you cannot rely on it.)

```csharp
var p = new Point(1, 2);   // struct  → stack slot, no allocation
var b = new byte[1024];    // array   → newarr → heap (Gen0)
var l = new List<int>();   // class   → newobj → heap (Gen0)
```

`struct` ≠ "no allocation":

- A `struct` declared as a **field of a class** lives inside that class on the heap.
- A `struct` returned by a method is copied into the caller's stack slot — but if the caller assigns it to a `class` field, it lands on the heap.
- A `struct` **boxed** to `object` or to an interface (see below) is heap-allocated.

## Boxing: the IL mechanics

Boxing is the runtime operation that takes a value-type instance and wraps it in a freshly-allocated heap object so it can be referenced through `object` or an interface. Unboxing is the reverse: type-check, then copy the value out.

```csharp
int i = 42;
object o = i;       // box   → newobj-equivalent: allocates a small object, copies 4 bytes
int j = (int)o;     // unbox.any → typecheck + copy out
```

The `box` IL instruction allocates an object the size of `T` plus the standard object header (≈16 bytes on x64) and copies the value in. So a single `int → object` is ~24 bytes on the heap, every time. In a hot loop, this is what kills you.

The C# compiler emits `box` in more places than most devs realize:

```csharp
// 1. Assigning a value type to object / dynamic / Enum / ValueType
object o = 42;

// 2. Assigning to an interface variable (the value type implements the interface)
IComparable c = 42;          // boxes the int

// 3. Calling a base virtual method (object.ToString, Equals, GetHashCode) on a struct
//    that does NOT override it — the runtime boxes to dispatch through System.ValueType
struct S { } var s = new S();
s.ToString();                // boxes to ValueType.ToString()

// 4. Generic method body that uses the unconstrained T as object
T MakeNullableObject<T>(T x) => x;            // no box if T is a value type and JIT specializes
object Awkward<T>(T x) => x;                  // boxes if T is a value type — return slot is object

// 5. params object[] overloads
Console.WriteLine("x={0}", 42);               // boxes 42 into the params object[]

// 6. Interpolated string into a string-typed parameter (pre-C# 10 path)
//    With C# 10 InterpolatedStringHandler-aware overloads, the box is avoided.
string s2 = $"value={value}";                 // may box `value` if no handler-aware overload
```

Unboxing is cheaper than boxing (no allocation), but it still type-checks and copies. `unbox.any` throws `InvalidCastException` on mismatch. In tight numerical code, replacing `object`-keyed dictionaries with generic ones eliminates entire classes of unbox-and-cast.

## Everyday boxing traps

| Trap | What boxes | Senior fix |
|---|---|---|
| `IDisposable d = someStruct;` | the struct | take the struct by `ref` / generic constraint |
| `Dictionary<MyEnum, V>` (default comparer pre-.NET 7) | enum → `object` keys via comparer | .NET 7+ specialized; otherwise pass `EnumComparer<MyEnum>` |
| `EqualityComparer<T>.Default` on a struct that does not implement `IEquatable<T>` | comparer falls back to `object.Equals`, boxes both sides | implement `IEquatable<T>` |
| `string.Format("{0}", structValue)` (legacy path) | the struct | use C# 10+ interpolation with handler-aware overloads (`Console.WriteLine($"…")`, `LoggerMessage` source gen) |
| `Enum.HasFlag` (pre-.NET Core 2.1) | both flag and value | C# bitwise `(value & flag) == flag`, or modern HasFlag (intrinsic now) |
| `params object[]` overloads | every value-type arg | choose an overload taking `string`/`ReadOnlySpan<object?>`/typed args |
| `ArrayList`, `Hashtable`, non-generic `IEnumerable.GetEnumerator` | every element | use `List<T>`, `Dictionary<K,V>`, generic enumerators |
| `foreach` over `IEnumerable<T>` where the source is a `List<T>` | the **enumerator** boxes (struct enumerator → interface) | iterate the concrete `List<T>` (returns its struct enumerator) |

The `List<T>` enumerator case is especially common. `List<T>.GetEnumerator()` returns a `struct Enumerator`. If you `foreach` directly over `myList`, the compiler binds to that struct enumerator → no boxing. If you `foreach` over `IEnumerable<T> source = myList`, it binds to `IEnumerable<T>.GetEnumerator()` → boxes the enumerator. Tight loops over collections passed as `IEnumerable<T>` allocate.

## Allocations that look free

```csharp
// String concat in a loop — N intermediate strings, all garbage.
string s = "";
for (int i = 0; i < n; i++) s += i;       // O(n²) allocs

// LINQ — every operator allocates an iterator object + delegate per call site.
var xs = items.Where(x => x.Active).Select(x => x.Id).ToList();

// Captured locals — closure becomes a heap-allocated display class.
int threshold = GetThreshold();
items.Where(x => x.Score > threshold);    // allocates a closure class

// Task.FromResult is not free.
ValueTask<int> Lookup(int k) => new(_cache[k]);   // ValueTask: stack-only on success path

// DateTime.Now.ToString() — allocates the formatted string.
log.Info($"now = {DateTime.UtcNow:O}");

// async Task method — even if it completes synchronously, the state-machine box exists.
//    Switch to ValueTask + [AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]
//    only when measured to matter.
```

See [Closure captures](./closure-captures.md) for the lambda case in detail.

## Detecting allocations

- **BenchmarkDotNet `[MemoryDiagnoser]`** — emits `Allocated` and `Gen0/1/2` columns per benchmark. The first column to fix.
- **PerfView** — `Take Heap Snapshot` for retention; `Collect → AllocationTick` for sampled allocation paths with full stacks.
- **`dotnet-trace collect --providers Microsoft-DotNETCore-SampleProfiler,Microsoft-Windows-DotNETRuntime:0x1:5`** — captures GC + sample profiler; `dotnet-counters monitor System.Runtime` for live alloc rate (`alloc-rate` counter, bytes/sec).
- **dotMemory / JetBrains profiler** — allocation hotspots view, retention paths.
- **Roslyn analyzers** — the [Heap Allocations Viewer](https://marketplace.visualstudio.com/items?itemName=MukulSabharwal.HeapAllocationsViewer) extension, the `Microsoft.CodeAnalysis.PerformanceSensitiveAnalyzers` package (HAA rules), and `CA1859` / `CA1860` from the BCL analyzers.

```csharp
[MemoryDiagnoser]
public class StringBenches
{
    [Benchmark] public string Concat() { var s = ""; for (int i = 0; i < 100; i++) s += i; return s; }
    [Benchmark] public string Builder() { var sb = new StringBuilder(); for (int i = 0; i < 100; i++) sb.Append(i); return sb.ToString(); }
}
```

The output's `Allocated` column tells you everything. The number you want in a hot service path is **0 B**.

**Senior-level gotchas:**

- Generic methods with `where T : struct` get **JIT-specialized** per value type — no boxing inside the body. Generic methods with no constraint (or `where T : class`) share one code body that treats `T` as `object`, which is *exactly where covert boxing lives*.
- `EqualityComparer<T>.Default` boxes if `T` does not implement `IEquatable<T>` — every `Dictionary` lookup pays. Adding `IEquatable<T>` to your structs is a free perf win.
- C# 10+ interpolated string handlers (`[InterpolatedStringHandler]`) only avoid the box when the receiving API has an overload that accepts the handler. `ILogger.LogInformation`, `Debug.Assert`, `string.Create`, `StringBuilder.Append` got these. Random user APIs taking `string` did not.
- `foreach` over a struct enumerator (e.g. `List<T>`, `Dictionary<K,V>`, `Span<T>`) does not box. `foreach` over `IEnumerable<T>` always does. Erase abstraction at the call site of hot loops.
- `Tuple<…>` is a class — heap allocation per construction. `ValueTuple<…>` (`(a, b)`) is a struct — stack only unless boxed.
- `Nullable<T>` boxing is special-cased: the runtime boxes the underlying `T` (or `null`) — never the `Nullable<T>` wrapper. So `(object)(int?)42` produces a boxed `int`, identical to `(object)42`. This is also why `is int` works on a `int?`.
- `enum.HasFlag` is now a JIT intrinsic and no longer boxes (since .NET Core 2.1). Old "always avoid HasFlag" guidance is dated — but `(value & flag) == flag` is still clearer in protocol code.
- Boxing is most painful in **GC pressure** terms, not raw cycles. A box is ~24 bytes; 100k/s of them is 2.4 MB/s of Gen0 churn — measurable on `dotnet-counters` `gc-alloc-rate` and visible as Gen0 frequency in `gc-heap-size` plots. Watch the alloc-rate counter, not the wall-clock benchmark.
