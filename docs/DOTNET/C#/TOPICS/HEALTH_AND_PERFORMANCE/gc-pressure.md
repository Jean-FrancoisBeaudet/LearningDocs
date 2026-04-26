# GC pressure

_Targets .NET 10 / C# 14. See also: [GC generations](./gc-generations-gen0-gen1-gen2.md), [Allocations and boxing](./allocations-and-boxing.md), [Closure captures](./closure-captures.md), [LINQ performance misuse](./linq-performance-misuse.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [Span&lt;T&gt;](./span-of-t.md), [BenchmarkDotNet](./benchmarkdotnet.md)._

**GC pressure** is a *rate* problem, not a *size* problem.

> The application is allocating fast enough — and/or promoting fast enough — that the garbage collector consumes a non-trivial fraction of CPU or breaches pause-time SLOs. Heap size may be perfectly stable; the cost is the work of collecting, not the memory held.

This is distinct from two adjacent problems:

| Problem | Signal | Root |
|---|---|---|
| **GC pressure** | High `% time in GC`; Gen0 churn; latency spikes correlated with collections | Allocation rate or promotion rate |
| **Memory leak** | Heap and Gen2 count grow monotonically with uptime | Something is rooting objects |
| **High working set** | Large but stable heap | App really needs that much memory; not a bug |

The same fix list does not work for all three. Pressure is fixed by **allocating less**; leaks are fixed by **finding the root**; high working set is fixed by **using less data or scaling out**.

## Quantitative signals

A web service is under GC pressure when, sustained at peak load, you see roughly:

- **`% time in gc` > 10%**, often 20–40%.
- **`alloc-rate` > ~100 MB/s** on hot endpoints (workload-dependent — 1 GB/s is normal for some pipelines, but only if Gen0 stays cheap).
- **`gen-0-gc-count` ticking by tens per second**.
- **Gen2 count flat or near-flat** (otherwise it's a leak, not pressure).
- **p99 latency tail tracks GC pauses** in correlated traces.

Below these thresholds, allocation work is normal and not worth chasing — a Gen0 at 100 MB/s with `% time in gc` at 4% is fine.

## Measurement recipe

Walk this ladder before changing code:

1. **Live counters** — fastest signal:
   ```bash
   dotnet-counters monitor System.Runtime --process-id <pid>
   ```
   Watch `% time in gc`, `alloc-rate`, the Gen0/1/2 counts, and `gc-heap-size`.
2. **Allocation profile** — what's allocating:
   ```bash
   dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime:0x1:5 \
       --process-id <pid> --duration 00:00:30
   ```
   Open in PerfView's *GC Heap Alloc Stacks* view, or convert to Speedscope. The top of the list is your target.
3. **Per-method accounting** — confirm the fix:
   ```csharp
   [MemoryDiagnoser]
   public class FormatterBenchmarks
   {
       [Benchmark]
       public string Old() => $"id={_id} name={_name}";

       [Benchmark]
       public string New() => string.Create(CultureInfo.InvariantCulture, $"id={_id} name={_name}");
   }
   ```
   `BenchmarkDotNet` reports `Allocated` per op. A code change that drops it is real; a change that doesn't is theatre.
4. **In-test assertion** — guard against regression:
   ```csharp
   long before = GC.GetAllocatedBytesForCurrentThread();
   target.HotPath();
   long delta = GC.GetAllocatedBytesForCurrentThread() - before;
   Assert.True(delta < 1024, $"hot path allocated {delta} B");
   ```

## The usual suspects

In rough descending order of how often they appear in real services:

- **Boxing.** `int → object`, `enum → object`, `(int x, int y) → object`. A `Dictionary<string, object>` cache that stores ints, an interpolated logger that hits the `object[]` overload, an `EventArgs` carrying a value type. Use generic methods, `IUtf8SpanFormattable`, and modern interpolated string handlers. See [`./allocations-and-boxing.md`](./allocations-and-boxing.md).
- **Closure captures.** A lambda capturing a local promotes that local to a heap-allocated closure object. In a hot loop this is one allocation per iteration. Hoist the lambda to a `static` if it doesn't actually capture. See [`./closure-captures.md`](./closure-captures.md).
- **LINQ chains in hot paths.** Each operator allocates an iterator, plus a closure for each lambda. `Where/Select/ToList` on a 100-element list inside a per-request handler is several KB per request. Use `for`/`foreach` and `Span<T>` extensions in hot code. See [`./linq-performance-misuse.md`](./linq-performance-misuse.md).
- **String concatenation and interpolation in loops.** `result += part;` inside a loop is O(n²) allocations. Use `StringBuilder` with capacity, `string.Create`, or `string.Concat(IEnumerable<string>)`.
- **`List<T>` / `Dictionary<TKey,TValue>` resizing.** Doubling produces interim arrays that escape to LOH past 85k bytes. Pre-size when the count is known: `new List<T>(expected)`, `new Dictionary<,>(expected)`.
- **`Task<T>` boxing on async hot paths.** Awaiting a method that always completes synchronously still allocates a `Task` if its return type isn't `ValueTask`. Use `ValueTask<T>` for high-frequency methods that often return without yielding (cache hits, validation paths). The compiler-generated state machine itself becomes a heap object only if the method actually yields — but every yield is one allocation.
- **DTO churn.** Returning a fresh response object per request is fine; allocating a hierarchy of nested DTOs and projecting into them with LINQ is not. For the highest-volume endpoints, write the response straight to the output buffer with `Utf8JsonWriter` and no intermediate objects.
- **`MemoryStream` for variable-size buffering.** Grows past 85k → LOH. Use `RecyclableMemoryStream` (Microsoft.IO.RecyclableMemoryStream) with sized pools.
- **Logging.** `ILogger.LogInformation("user {id} did {x}", id, x)` is fine. `ILogger.LogInformation($"user {id} did {x}")` allocates a string before any filter check; if the level is disabled, you've allocated for nothing. Use the source-generated `LoggerMessage` attribute on hot paths.
- **`async` over `Stream` without buffering.** Each `ReadAsync(byte[])` may allocate a `Task`. Prefer `ReadAsync(Memory<byte>)` with a pooled buffer.

## The fix toolkit

| Tool | What it removes | When |
|---|---|---|
| `Span<T>` / `ReadOnlySpan<T>` | Stack-only views — zero alloc parsing/slicing | Any contiguous-buffer work in a hot path |
| `Memory<T>` / `ReadOnlyMemory<T>` | Pooled-buffer handle that survives `await` | Async pipelines using pooled buffers |
| `ArrayPool<T>.Shared` | Heap allocation of large arrays | Buffers ≥ a few KB; especially ≥ 85k |
| `RecyclableMemoryStream` | LOH `MemoryStream` allocations | Variable-size buffering / serialization |
| `StringBuilder` w/ capacity | Repeated string allocations | Multi-step text construction |
| `string.Create` / `IUtf8SpanFormattable` | Intermediate string boxes | Hot-path formatting |
| `ValueTask<T>` | `Task<T>` allocation on sync completion | Cache hits, fast-path validation |
| `struct` types / `record struct` | Heap object per instance | Small, immutable, equality-by-value data |
| `FrozenDictionary<,>` / `FrozenSet<>` | Hash-table allocations + better cache locality on read | Read-only lookup tables built once |
| `System.Text.Json` source generator | Reflection-time delegate allocations | High-throughput JSON endpoints |
| `LoggerMessage` source generator | Object[] boxing per log call | Hot logging |

A worked example — a text-parsing hot path:

```csharp
// Before: alloc per call - Substring, Split, Parse(string)
public int CountValid(string csv)
{
    var parts = csv.Split(',');                 // string[]
    int n = 0;
    foreach (var p in parts)                    // each p is a substring
        if (int.TryParse(p, out var v) && v > 0) n++;
    return n;
}

// After: zero alloc - Span slicing, Parse(ReadOnlySpan<char>)
public int CountValid(ReadOnlySpan<char> csv)
{
    int n = 0;
    foreach (var range in csv.Split(','))       // ranges, no string[]
        if (int.TryParse(csv[range], out var v) && v > 0) n++;
    return n;
}
```

Same algorithm, two orders of magnitude fewer allocations, and the JIT vectorizes the comparison work.

## Is GC actually the bottleneck?

Pressure is a hypothesis until you've checked the alternative: **CPU bound somewhere else**. A profiler flame graph (`dotnet-trace` → Speedscope, or PerfView's *Any-Stacks* view) tells you whether time is in:

- `clr!WKS::GCHeap::*` / `coreclr!SVR::GCHeap::*` — GC code. Pressure is real.
- User code in tight loops — algorithmic bottleneck. GC is downstream noise.
- I/O / lock waits — neither CPU nor GC. Different problem.

Don't optimize allocations on a path that already runs in 50 µs and isn't on the critical timeline. The cost of `Span` discipline in code that's never hot is **maintenance**, not throughput.

## SLO-driven thinking

The right target isn't "zero allocations." It's "p99 latency under X ms with throughput Y rps." A 5% allocation-rate reduction that doesn't move the SLO is not a win — and might trade away clarity. Decide the budget first, then chase whatever is preventing it.

**Senior-level gotchas:**
- **`ArrayPool` doesn't fix pressure for free.** Forgetting to `Return` turns a "free" rent into a permanent leak; oversized rents land on the LOH; concurrent rent/return contention on the shared pool can become its own bottleneck. Wrap with `try/finally` or `using` and measure.
- **`ValueTask<T>` is not a drop-in replacement for `Task<T>`.** Awaiting it twice, calling `.Result` more than once, or storing it in a field is undefined. Use only as a method return type, awaited exactly once.
- **Source-generated logging** removes boxing only for known overload shapes. If you pass an `object` that isn't a primitive `IUtf8SpanFormattable`, you're back to allocating.
- **Profiler self-effect**: ETW allocation tracking samples allocations; under heavy churn the act of profiling distorts the numbers. Use `--providers Microsoft-Windows-DotNETRuntime:0x1:5` (informational) for trends, raise verbosity only briefly.
- **Don't conflate Gen2 promotion with pressure.** Stable Gen2 with high Gen0 churn = pressure. Growing Gen2 with low Gen0 churn = leak. Same `% time in gc` value can mean either.
- **Pinned objects and pressure interact badly.** Long-lived pins on the SOH prevent compaction → fragmentation → premature promotions → more Gen2 work. If you pin, pin on the POH (`GC.AllocateArray<T>(len, pinned: true)`).
- **Boxing through `dynamic`** is invisible until you trace it. Avoid `dynamic` on hot paths.
- **`Task.WhenAll(IEnumerable<Task>)`** allocates a `Task[]` and a closure if you used LINQ to build the source. Materialize with `.ToArray()` once, or use the `ReadOnlySpan<Task>` overload (.NET 9+).
- **`StringBuilder.ToString()` allocates a final string** — that's by design. The win is removing the intermediate strings during construction. If the consumer accepts `ReadOnlySpan<char>`, write directly to a pooled `char[]` instead.
- A regression that "only adds 200 bytes per request" at 10k rps is **2 MB/sec** of allocation. Multiply before dismissing.
