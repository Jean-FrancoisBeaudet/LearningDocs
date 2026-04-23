# GC generations

_Targets .NET 10 / C# 14. See also: [How GC works](./how-gc-works.md), [Finalizer and Dispose pattern](./finalizer-and-dispose-pattern.md)._

The .NET GC is generational because of one empirical observation — **the weak generational hypothesis**: most objects die young. If young and old objects live in separate regions, you can collect just the young region most of the time and almost never touch the old one. That's where the 10–100× throughput difference between Gen0 and Gen2 comes from.

## The generations

.NET's managed heap is split into **four** generations (three for the SOH + a separate area):

| Name | Who lives here | How often collected | Typical pause |
|---|---|---|---|
| **Gen0** | Freshly allocated small objects | Very often | Microseconds |
| **Gen1** | Survivors of one Gen0 collection | Less often | Microseconds to low ms |
| **Gen2** | Survivors of Gen1; **all** LOH / POH allocations | Rarely | Low ms to tens of ms (background) |
| **LOH** (Large Object Heap) | Allocations ≥ 85,000 bytes | With Gen2 | Collected with Gen2 |
| **POH** (Pinned Object Heap, .NET 5+) | `GC.AllocateArray<T>(..., pinned: true)` | With Gen2 | Collected with Gen2; never compacted |

Gen0 and Gen1 together are called the **ephemeral generations** — they share a single segment (or region) per heap. Gen2 can span many segments.

> **Everything in LOH and POH is Gen2**, by definition. There's no "Gen0 LOH."

## The 85,000-byte LOH threshold

Any managed allocation of **85,000 bytes or more** goes directly to the LOH. For typed arrays, the cutoff is slightly smaller than the raw byte count because of the object header and method-table pointer (16 bytes on x64). In practice:

- `new byte[85_000]` → LOH.
- `new int[21_250]` → LOH (21,250 × 4 = 85,000).
- `new double[10_000]` → LOH (10,000 × 8 = 80,000 — actually SOH; the 85k rule is strict).
- `string s = new string('x', 42_500)` → LOH (42,500 × 2 bytes/char + header ≥ 85k).

The number is hardcoded in the runtime and not configurable in production releases.

## Promotion

An object's generation only increases. Promotion happens when an object **survives** the collection of its current generation:

1. Allocated at Gen0.
2. Survives a Gen0 collect → promoted to Gen1.
3. Survives a Gen1 collect → promoted to Gen2. (Stays there forever, until it becomes unreachable *and* a Gen2 runs.)

A **Gen1 collection collects Gen0 + Gen1**. A **Gen2 collection collects Gen0 + Gen1 + Gen2 + LOH + POH** — a "full GC." You don't usually choose which; the runtime's heuristics decide based on allocation budgets, memory pressure, and `GCSettings.LatencyMode`.

## The allocation budget

Each generation has an **allocation budget** — a threshold measured in bytes. When the budget is exhausted, a collection of that generation is triggered. Budgets are **dynamic**: the runtime grows them when survival rate is high (the generation is effective) and shrinks them otherwise. This is why your steady-state GC rate self-regulates to the app's shape.

You cannot inspect budgets directly, but `GC.GetGCMemoryInfo()` shows last-collection thresholds and heap sizes.

## Why Gen2 is expensive

A Gen2 collection must:
- Scan **the entire heap** reachable from roots (not just the ephemeral segment).
- Compact the SOH (Gen0+Gen1+Gen2 on the SOH are slid together).
- Sweep (but not compact) the LOH by default — **LOH compaction is opt-in**: `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;` triggers a one-time compact on the next full GC. LOH fragmentation is real; this is your escape hatch.
- Update every reference in the heap to point to moved objects.

Background GC (enabled by default) runs most of this concurrently with user code, but the final sync point still causes a stop-the-world pause proportional to live-set size, not total heap size.

## Regions vs segments (.NET 7+)

Before .NET 7, the GC managed the heap as a small number of **segments** — large fixed-size memory chunks. .NET 7 introduced a **regions-based** heap on 64-bit: many small regions (typically 4 MB each) that can be reclaimed and reassigned between generations independently. Benefits:
- Better memory locality for ephemeral allocations.
- Easier to shrink the heap back to the OS when the app quiets down.
- Empty regions return to the OS rather than sitting as committed-but-unused segment headroom.

Effect on you as a developer: almost nothing — same API, better behavior. You can inspect region info via `GC.GetGCMemoryInfo()` (look at `HeapSizeBytes` vs `FragmentedBytes`).

## Observing generations

```csharp
long gen0 = GC.CollectionCount(0);   // count of Gen0 collections so far
long gen1 = GC.CollectionCount(1);
long gen2 = GC.CollectionCount(2);

GCMemoryInfo info = GC.GetGCMemoryInfo();
Console.WriteLine($"Heap: {info.HeapSizeBytes:N0} B");
Console.WriteLine($"Frag: {info.FragmentedBytes:N0} B");
Console.WriteLine($"LOH:  {info.GenerationInfo[3].SizeAfterBytes:N0} B");
Console.WriteLine($"POH:  {info.GenerationInfo[4].SizeAfterBytes:N0} B");
```

The `GenerationInfo` array covers Gen0..Gen2 plus LOH and POH. In production health endpoints, log the Gen2 count and heap size periodically — they are the best early-warning signals for memory leaks.

## The shape of a well-behaved heap

- **Gen0 count**: high, growing continuously. Each collection is cheap. Fine.
- **Gen1 count**: ~5–10× lower than Gen0. Fine.
- **Gen2 count**: ~10–100× lower than Gen1. In steady state, Gen2 count should grow slowly or stabilize. If it grows linearly with time, you have a leak.
- **LOH size**: stable, not monotonically growing.
- **% Time in GC** (perfcounter / `dotnet-counters`): under ~10% in a healthy app. Over 30% is an allocation-rate problem; over 50% means the app is mostly GC'ing.

## Practical allocation advice by generation

**Keep out of Gen2**:
- Don't retain short-lived data in long-lived containers. A per-request object that ends up in a static `Dictionary` is a leak.
- Event handler subscriptions where the subscriber is short-lived and the publisher is long-lived → subscriber graph gets promoted to Gen2 via the event's delegate list.

**Keep off the LOH**:
- Reuse large buffers with `ArrayPool<T>.Shared.Rent` / `Return` instead of `new byte[size]`.
- Prefer `StringBuilder` with an initial capacity over repeated `string +` in loops — avoids intermediate large strings.
- Careful with `List<T>` growth: doubling strategy means it **will** eventually allocate a large backing array. Pre-size with `new List<T>(expectedCount)` if you know the count.

**LOH compaction**:

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();   // paid explicitly; expensive
```

Use only at a known quiet moment — app startup settlement, between batch phases. Not during request handling.

## Server vs Workstation impact on generations

Server GC gives you **N heaps** (roughly one per logical core), each with its own ephemeral segment and LOH. A Gen0 collection runs in parallel across all heaps but still stops all managed threads during the stop-the-world phases. The advantage: wall-clock pause time drops roughly linearly with core count for the parallel phases. The cost: memory usage is higher because each heap reserves its own ephemeral budget.

This means Gen0 budgets are **per-heap** in Server GC. A 500 MB "Gen0 total" on an 8-heap Server GC is ~62 MB per heap, collected in parallel. In Workstation GC it's one 500 MB budget collected by one thread.

## Finalizable objects and generations

An object with a finalizer is promoted **at least one extra generation** before it can be freed: the finalizer runs after the first collection that finds it unreachable, but the actual memory is reclaimed in the next collection. If a Gen0 finalizable object becomes unreachable, it's promoted to Gen1 for finalization, then freed in the next Gen1 collection. This is another reason finalizers are expensive — they defer collection and grow the old generations.

**Senior-level gotchas:**
- The **85,000-byte threshold** is *bytes allocated*, not logical size. A struct with padding and an object header can cross the threshold at a lower "apparent" element count than you expect. For strings, remember it's 2 bytes/char on .NET (UTF-16 internal storage).
- **`List<T>` resize on large collections silently lands on the LOH.** Resizing from 16k to 32k elements of `ValueTuple<long, long>` creates a 512KB array — straight to LOH. Pre-size aggressively when you know the count.
- **`ArrayPool<T>.Shared.Rent(n)` may return a larger array than you asked for** — always use `AsSpan(0, n)` / `AsMemory(0, n)` to slice to the requested length. It reuses a bucket-based pool; don't assume `buffer.Length == n`.
- **LOH is not compacted by default.** Long-running services that churn large buffers slowly fragment the LOH. `ArrayPool` mitigates this completely by reusing buffers. `CompactOnce` mode is a bandaid.
- **Boxed value types** are Gen0 allocations. A logging line `$"value = {intValue}"` used to box pre-C# 10. Modern interpolated string handlers avoid the box for most primitive types — but only when the compiler is on a recent LTS and the target type implements the right pattern. Check IL when in doubt.
- **Async state machines** cross into Gen2 territory easily. If an `async` method captures a few megabytes of state (big closures, large local arrays) and yields to I/O for seconds, that state machine lives through Gen0 and Gen1 and gets promoted. High-throughput systems use `ValueTask`, small closures, and `ArrayPool` buffers to keep the box small.
- **Pinning blocks compaction.** A pinned object in the middle of Gen0 prevents survivors from being slid past it, which fragments the segment and accelerates promotion of other objects. Long-lived pins belong on the POH.
- **Gen2 collection frequency is a better leak indicator than heap size.** A growing heap might just be steady-state working set; a monotonically increasing Gen2 collection count means you're filling Gen2 and can't free it.
- Running `GC.Collect(2, GCCollectionMode.Forced, blocking: true, compacting: true)` costs *real money* in pause time. It exists for testing and quiet-moment cleanup. `GCCollectionMode.Aggressive` (.NET 7+) also shrinks heap segments back to the OS — useful after a batch job, never during request serving.
- **Container memory limits are not always detected.** `cgroups` v1/v2 handling has been buggy across versions. Set `DOTNET_GCHeapHardLimit` or `DOTNET_GCHeapHardLimitPercent` explicitly if you see the runtime allocating past the container's memory ceiling.
- **A Gen0 is not "free"**. It still pauses all managed threads briefly. A pathological Gen0 rate (e.g., 100/sec) can add up to a significant time-in-GC percentage even if each pause is ~1ms.
