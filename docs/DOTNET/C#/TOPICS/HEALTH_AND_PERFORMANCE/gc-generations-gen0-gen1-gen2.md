# GC generations (gen0 / gen1 / gen2)

_Targets .NET 10 / C# 14. See also: [How GC works](../../MEMORY_MANAGEMENT/how-gc-works.md), [GC generations (concept)](../../MEMORY_MANAGEMENT/gc-generation.md), [GC pressure](./gc-pressure.md), [Server vs Workstation GC](./server-vs-workstation-gc.md), [LOH](./large-object-heap-loh.md)._

This note is the **diagnostics-and-tuning** view of generations. The conceptual mechanics — promotion rules, allocation budgets, regions vs segments — live in [`../../MEMORY_MANAGEMENT/gc-generation.md`](../../MEMORY_MANAGEMENT/gc-generation.md). Here we focus on **what a perf engineer measures, what the numbers mean, and what to change**.

The whole generational scheme is justified by the **weak generational hypothesis**: most objects die young. A healthy app exploits that — Gen0 collections are cheap and frequent, Gen2 collections are rare and expensive. When the curve goes the wrong way, your latency does too.

## Generation cost model

| Generation | Scans | Typical pause (BGC enabled) | Frequency in a healthy web service |
|---|---|---|---|
| **Gen0** | Ephemeral segment only (Gen0 region) | ~50 µs – 1 ms | 10s/sec under load |
| **Gen1** | Gen0 + Gen1 | ~100 µs – few ms | 1–5/sec |
| **Gen2 (foreground)** | Whole heap, blocking | 10 ms – 100s of ms | should be rare; spikes = problem |
| **Gen2 (background)** | Whole heap, mostly concurrent | Two short pauses (~ms each) | Acceptable; this is the normal Gen2 in BGC |
| **LOH / POH** | Collected with Gen2 | Same as Gen2 | Same as Gen2 |

Pause times scale with **live-set size**, not heap size. A 10 GB heap that's mostly dead collects fast; a 1 GB heap with deep live graphs is slower.

## Reading the counters

`dotnet-counters monitor System.Runtime --process-id <pid>` (or `--name <app>`) is the cheapest live signal. The columns that matter:

| Counter | What to watch for |
|---|---|
| `gen-0-gc-count` | Should grow continuously; rate matters more than absolute |
| `gen-1-gc-count` | ~5–10× lower than Gen0 |
| `gen-2-gc-count` | ~10–100× lower than Gen1; **monotonic growth = leak** |
| `gen-0-size` / `gen-1-size` / `gen-2-size` | Per-generation occupancy after the last collect |
| `loh-size` | Stable in healthy apps; growing = pooling missing |
| `poh-size` | Pinned object heap; should be small and bounded |
| `% time in gc` | < 10% healthy; 30%+ = throughput problem; 50%+ = mostly GC'ing |
| `gc-heap-size` | Total managed heap; correlate with working set |
| `gc-fragmentation` | .NET 7+; rising = SOH/LOH not compacting effectively |
| `alloc-rate` | Bytes/sec; correlate with `% time in gc` |

For one-shot capture in a unit test or smoke run:

```csharp
long gen0 = GC.CollectionCount(0);
long gen1 = GC.CollectionCount(1);
long gen2 = GC.CollectionCount(2);

GCMemoryInfo info = GC.GetGCMemoryInfo();
Console.WriteLine($"heap={info.HeapSizeBytes:N0} frag={info.FragmentedBytes:N0}");
Console.WriteLine($"loh={info.GenerationInfo[3].SizeAfterBytes:N0}");
Console.WriteLine($"poh={info.GenerationInfo[4].SizeAfterBytes:N0}");
```

Expose `GC.GetGCMemoryInfo()` from a `/healthz` or metrics endpoint. Heap and Gen2 trends over hours/days catch leaks long before they crash.

## Healthy ratios

Steady state on a well-behaved service:

- **Gen0 : Gen1 : Gen2 ≈ 100 : 10 : 1** counts.
- **Gen2 count grows logarithmically**, not linearly, with uptime.
- **LOH size is flat** (any churn is absorbed by `ArrayPool` / `RecyclableMemoryStream`).
- **% time in GC stays under ~10%** at peak load.

If Gen2 is climbing in lockstep with request count, you have a **leak** (something is rooting per-request data). If Gen2 is flat but `% time in GC` is high, you have **pressure** (see [`./gc-pressure.md`](./gc-pressure.md)) — high allocation rate without retention.

## Promotion traps

These are the patterns that quietly push short-lived objects into Gen2, where they sit until the next full GC:

- **Long-lived caches that capture per-request objects.** A `static Dictionary<string, RequestContext>` or singleton DI service holding a list of recent items will promote everything it touches.
- **Static event subscriptions** (`SomeStatic.Event += handler`). The publisher's delegate list keeps the handler — and its `this` capture — alive forever.
- **`async` state machines that yield for seconds.** The state machine is a heap object holding all locals captured across the `await`. If it survives one Gen0, it goes to Gen1; if it's still suspended at the next collection, Gen2.
- **Finalizable types**. An object with a finalizer is promoted **at least one extra generation** so the finalizer thread can run. If `A` has a finalizer and references `B`, `B` is also held until `A` is finalized — even if `B` has no finalizer of its own. Avoid finalizers on anything with non-trivial captures.
- **Closures that capture `this`** in a long-running task. The capture object holds the entire enclosing class graph.
- **Pooled buffers escaping into long-lived collections.** If you `ArrayPool.Rent` and forget to `Return`, the rented array gets promoted; you've turned a pooled buffer into a permanent leak.

## Tuning levers

Default behavior is right for most services. Reach for these when you've measured a problem.

| Setting | Effect | When to use |
|---|---|---|
| `<ServerGarbageCollection>true</ServerGarbageCollection>` | N heaps, parallel collect | Multi-core services (default in ASP.NET Core) |
| `DOTNET_GCHeapCount=N` | Pin number of heaps | 1-vCPU containers; cap on overprovisioned hosts |
| `DOTNET_GCHeapHardLimit=<bytes>` | Cap managed heap | Containers with strict memory limits |
| `DOTNET_GCHeapHardLimitPercent=<0-100>` | Cap as % of cgroup memory | Same, percentage-relative |
| `DOTNET_GCConserveMemory=0..9` | Trade throughput for smaller heap | Memory-tight services |
| `DOTNET_GCDynamicAdaptationMode=1` | Enable DATAS (auto heap-count tuning) | Variable-load services on .NET 8+ |
| `GCSettings.LatencyMode = SustainedLowLatency` | Avoid Gen2 in critical sections | Latency-sensitive bursts |
| `GC.TryStartNoGCRegion(bytes)` | Disable GC for a budgeted window | Trading allocations for guaranteed no-pause |

```csharp
if (GC.TryStartNoGCRegion(totalBytes: 16 * 1024 * 1024))
{
    try { ProcessLatencyCriticalBatch(); }
    finally { GC.EndNoGCRegion(); }
}
```

`NoGCRegion` does not magic away allocations — it pre-reserves a budget. Exceeding it falls back to a normal collection, defeating the point.

## Diagnostic walkthrough

When a service shows elevated p99 latency or memory growth, walk this ladder:

1. **`dotnet-counters monitor System.Runtime`** — confirm the symptom is GC-shaped (`% time in gc` high, Gen2 count climbing, heap growing). If the counters are clean, GC isn't the bottleneck.
2. **`dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime:0x1:5 --process-id <pid>`** — capture a 30–60 second window. Open in PerfView (GC Stats view) or Speedscope. Look for: GC pause distribution, allocation tick frequency, top types by allocation.
3. **Heap snapshot** with `dotnet-dump collect --process-id <pid>` followed by `dotnet-dump analyze`, then `!dumpheap -stat` and `!gcroot <addr>` on suspect types. Or use **dotMemory** for a visual retention path.
4. **Confirm in code** with [`BenchmarkDotNet`](./benchmarkdotnet.md) `[MemoryDiagnoser]` on the suspect method — get a per-call allocation number you can defend in a code review.
5. **Fix and re-measure.** A change you can't show on a counter or trace is a guess.

See also [`./profiling-perfview.md`](./profiling-perfview.md), [`./profiling-dotmemory.md`](./profiling-dotmemory.md), [`./dotnet-counters.md`](./dotnet-counters.md), [`./dotnet-trace.md`](./dotnet-trace.md), [`./dotnet-dump.md`](./dotnet-dump.md).

**Senior-level gotchas:**
- A "Gen0 collection is free" rule of thumb is wrong at high allocation rates. Each collection still pauses managed threads briefly. 100 Gen0/sec at 1 ms each = 10% of wall-clock spent in GC.
- **Async state machines on slow paths get promoted.** If your endpoint awaits a 500 ms downstream call, the state machine survives several Gen0s and lands in Gen1 or Gen2. Use `ValueTask` for sync-completion paths and keep captured locals small.
- A single line `_recent.Add(currentRequest);` in a static collection turns every Gen0 object reachable from `currentRequest` into a Gen2 inhabitant. This is the most common leak shape in real services.
- **`GC.Collect()` in production is almost always a bug.** It exists for benchmarks, perf tests, and very narrow "between batch phases" cleanups. If you think you need it in a request handler, fix the allocation pattern instead.
- Server GC's **per-heap budgets** mean Gen0 size in counters is *per heap*; aggregate is heap-count × the number you see. Don't compare Workstation and Server numbers naively.
- The **finalizer queue is itself a root.** An object pending finalization keeps its entire graph alive across at least one extra collection. Logging types that hold a `FileStream` finalizer can pin big graphs.
- **Setting locals to `null` in `Dispose` does not help the GC** in 99% of cases — the JIT already shortens local liveness to last use. The exception is *fields on long-lived objects* whose graph you genuinely want to release; there, clearing the field is meaningful.
- `GCSettings.LatencyMode` is a hint, not a guarantee. Under memory pressure the runtime will collect Gen2 anyway.
