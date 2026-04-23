# How the GC works

_Targets .NET 10 / C# 14. See also: [GC generations](./gc-generation.md), [Finalizer and Dispose pattern](./finalizer-and-dispose-pattern.md), [Span and Memory](./span-and-memory.md)._

The .NET GC is a **generational, tracing, compacting, mostly-concurrent** garbage collector. Each of those words is load-bearing:
- **Generational**: young objects are collected more often than old ones (see [GC generations](./gc-generation.md)).
- **Tracing**: it finds live objects by walking a graph from roots, not by reference-counting.
- **Compacting**: after freeing dead objects it slides survivors together to defragment the heap.
- **Mostly-concurrent**: Gen2 (the expensive one) can run on a background thread while your code is executing, with short pauses to sync.

## The three phases of a collection

Every GC cycle does three things in order:

1. **Mark** — walk the object graph from the **roots**, painting every reachable object as live.
2. **Plan / Sweep** — compute the new position of each live object (planning) and free the gaps.
3. **Compact** — slide live objects down to close the gaps; update every reference in the heap to point to the new address.

"Stop-the-world" pauses happen at specific synchronization points — not during the entire collection in background/concurrent modes. The fraction of wall-clock time all managed threads are paused is your real latency budget.

## Roots

A **root** is any reference the GC treats as "definitely live" without tracing further back. The root set:

- **Stack slots**: every local variable on every managed thread's stack that currently holds an object reference.
- **CPU registers**: JIT'd code can keep references in registers; the GC inspects them via stack maps.
- **Static fields**: all loaded types' `static` reference fields.
- **GC handles**: `GCHandle` instances (normal, pinned, weak — weak handles are special-cased).
- **Finalizer queue / f-reachable queue**: objects awaiting finalization are reachable from this queue, even if nothing in user code references them.
- **The thread-pool internal queues**, timers, async state machines captured as delegates, etc.

If the GC can reach your object from any root, it survives. Anything else is dead and its memory is reclaimed.

## The write barrier and card tables

How does a Gen0 collection know which Gen2 objects point to Gen0? It **doesn't scan Gen2** — that would defeat the purpose of generational GC. Instead, every reference-type write is instrumented:

```csharp
obj.FieldRef = other;        // JIT'd to: write the ref AND mark a card
```

The **write barrier** is a small piece of code the JIT emits after every reference-type field write. It flags a **card** — an 8-byte-aligned chunk of the heap, tracked in a **card table** (a bit array). During a Gen0 collection, the GC scans only the cards that are dirty, treating pointers inside them as potential roots into Gen0.

Consequences you can actually observe:
- Reference-type field writes are measurably slower than value-type writes. In tight loops, a `struct` field update does not hit the barrier; a `class` field update does.
- Writing to large `class[]` arrays repeatedly in Gen2 dirties many cards and makes Gen0 collections more expensive. `struct[]` doesn't have this problem (no references inside, so no card marking).
- `ref` assignments inside a `ref struct` or `Span<T>` are **not** written to the heap — they live on the stack — and are free of barrier cost. This is part of why spans are fast.

## Background GC and the ephemeral-only vs full split

The GC runs in one of several modes (see also [GC generations](./gc-generation.md)).

- **Ephemeral collection** (Gen0 or Gen0+Gen1): blocking, short — microseconds to low milliseconds for reasonable heap sizes. These use the stop-the-world mark/compact path, but only on the small ephemeral segment, so the pause is tiny.
- **Full / Gen2 collection**: expensive. This is where **background GC** matters.

**Background GC (BGC)** runs the Gen2 mark/sweep concurrently with user threads. The sequence is roughly:

1. Short pause: snapshot roots, enable concurrent marking.
2. Concurrent mark: walks the Gen2 heap while your code runs. Write-barrier catches any pointer updates (concurrent mark uses a tricolor / snapshot-at-the-beginning algorithm to stay correct).
3. Short pause: a small "final mark" / remark phase to capture references created during concurrent marking.
4. Concurrent sweep (workstation GC) or compact-with-pause (server GC regions).

A new Gen0/Gen1 (foreground) collection can still happen during step 2, without waiting for background GC to finish. This is why Gen2 latency on modern .NET is dominated by the short pauses, not the full Gen2 mark time.

## Workstation vs Server GC

Two top-level flavors, configured at process startup. You cannot switch modes at runtime.

```xml
<!-- in .csproj -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

| Dimension | Workstation GC | Server GC |
|---|---|---|
| GC threads | 1 | N (≈ one per logical core) |
| Heaps | 1 shared heap | N heaps (one per GC thread) |
| Throughput | Lower (one thread compacting) | Higher (parallel compact) |
| Pause goal | Short | Tolerates longer for throughput |
| Memory usage | Lower | Higher (each heap has its own ephemeral segment) |
| Default in | UI / desktop apps | ASP.NET Core, Azure Functions, services |
| Background GC | Yes | Yes (was "Concurrent GC" in older names) |

Rule of thumb: **ASP.NET Core / multi-core services → Server. Single-threaded / memory-constrained / containers with 1 vCPU → Workstation.** A 1-core container running Server GC is often *worse* than Workstation because Server GC's heap-per-core model wastes memory without parallelism payoff.

## GC latency modes

`GCSettings.LatencyMode` is a hint to the collector. Modes:

- `Interactive` (default): balance pause time and throughput.
- `Batch`: optimize for throughput, accept longer pauses. Typical for compute batch jobs.
- `LowLatency`: avoid Gen2 where possible. Temporary (seconds) window. Useful for a known-critical burst of work.
- `SustainedLowLatency`: same, but intended to stay on longer. Will still collect if memory pressure is severe.
- `NoGCRegion`: via `GC.TryStartNoGCRegion(long size)` — the GC is *disabled* for a chunk of allocation budget. If you exceed it, the runtime falls back to a normal collect (and throws if configured to).

```csharp
if (GC.TryStartNoGCRegion(totalBytes: 16 * 1024 * 1024))
{
    try { RunLatencyCriticalSection(); }
    finally { GC.EndNoGCRegion(); }
}
```

Use these sparingly. 95% of apps should leave the mode alone and fix allocations instead.

## GC triggers

A collection fires when one of:
- The current generation's **allocation budget** is exceeded (the most common trigger; budgets are tuned by the runtime dynamically).
- An explicit `GC.Collect()` call (you should almost never write this in production — it's a profiling / perf-test affordance).
- Low-memory notification from the OS (`GC.RegisterForFullGCNotification` / internal signals).
- LOH allocations accumulate past the LOH threshold (triggers a Gen2, since the LOH is collected with Gen2).
- `AppDomain.Unload` — historical; N/A in modern .NET Core+.

Tuning `DOTNET_GCHeapHardLimit` / `GCHeapCount` / `GCConserveMemory` environment variables adjusts budgets and heap counts. Container runtimes commonly set these from cgroup memory limits automatically.

## Pinning

A **pinned** object cannot be moved during compaction. Pinning happens via:
- `fixed` statement in `unsafe` code.
- `GCHandle.Alloc(obj, GCHandleType.Pinned)`.
- `Memory<T>.Pin()` (which internally uses a pinning handle).
- Passing an array into unmanaged code via P/Invoke marshaling (transient pin for the call).

Excessive or long-lived pinning **fragments the heap** because the compactor has to flow around pinned objects. .NET 5 added the **Pinned Object Heap (POH)** — an explicit segregated area for long-lived pinned buffers — as an escape valve. Use `GC.AllocateArray<T>(len, pinned: true)` to allocate directly there.

```csharp
byte[] buf = GC.AllocateArray<byte>(8192, pinned: true);  // lives on POH, never moves
```

This is the right approach for long-lived I/O buffers feeding P/Invoke or kernel APIs. Pinning on the SOH (small object heap) for seconds-plus is a real perf bug.

## Concurrent correctness: the tricolor invariant

To allow concurrent marking while user code runs, the GC uses a tricolor scheme:
- **White** = not yet visited / presumed dead.
- **Grey** = reachable but children not yet scanned.
- **Black** = fully scanned, children also reachable.

The invariant: **a black object must never point to a white object**. If user code creates such an edge (black `obj.Field = newWhiteThing`), the write barrier catches it and either re-marks the target grey or records the card. That's how concurrent marking stays correct under concurrent mutation.

You don't write this code, but you feel its cost: every heap reference-field write pays the barrier. In hot paths, this is why senior .NET favors `struct`s, arrays of value types, and `Span<T>` — they're invisible to the barrier.

## Observing the GC

- **`dotnet-counters monitor`** — realtime Gen0/1/2 counts, LOH/POH size, heap %, GC pause %.
- **`dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime:0x1:5`** — ETW GC events (AllocationTick, GCStart, GCEnd, heap stats, finalizer run).
- **PerfView / dotTrace / JetBrains dotMemory** — visual timeline, per-object allocation, retention roots.
- **`[MemoryDiagnoser]`** on BenchmarkDotNet — Gen0/1/2 collection counts + bytes allocated per op.
- **`GC.CollectionCount(gen)`**, **`GC.GetTotalMemory(false)`**, **`GC.GetGCMemoryInfo()`** — programmatic snapshots, good for health endpoints.

Add a `/metrics` / `/healthz` endpoint that reports `GC.GetGCMemoryInfo()`'s `HeapSizeBytes`, `MemoryLoadBytes`, and `PauseDurations`. You'll catch heap growth regressions in prod before they crash.

## Common symptoms and what they mean

| Symptom | Likely cause |
|---|---|
| Gen2 collection count climbing steadily | Leak — something is rooting objects (static dictionary, event handler, thread-static cache, DI singleton holding transient) |
| Gen0 count high, Gen2 flat, CPU spent in GC | Throughput-killing allocations in hot path — boxing, closure captures, `string` concat in loops, LINQ inside hot loops |
| Big LOH, steady growth | Oversized arrays (`new byte[85_000]+`), `string.Concat` on huge strings, `List<T>` resizing past LOH threshold |
| GC pause spikes but total allocation is modest | Pinning fragmentation — long-lived pinned buffers on the SOH |
| Working set huge in a 1-core container | Server GC default for N heaps; set `ServerGarbageCollection=false` or `GCHeapCount=1` |

**Senior-level gotchas:**
- `GC.Collect()` in production code is almost always a bug. The cases it helps (tests, perf harnesses, "warmup" between phases of a batch job) are narrow. If you think you need it, first prove the allocation pattern with a trace.
- **Server GC uses more memory by design.** Don't compare local-dev (workstation) memory numbers against prod (server) and panic — the baseline is different.
- The write barrier runs on **every reference-type field write**, including inside `struct`s whose fields happen to be reference types. A `struct { string Name; }` assignment still triggers the barrier.
- **Finalizable objects hold references to their entire object graph until finalized.** If `A` has a finalizer and holds a reference to `B`, `B` cannot be collected until after `A` is finalized — even if `B` has no finalizer. Reason enough to avoid finalizers on anything with interesting captures.
- Setting a field to `null` in `Dispose` to "help the GC" is almost always cargo-culted. It can help in one specific case: a static field or long-lived cache holding a big graph that's no longer needed. For local variables, the JIT already narrows their live range to "last use" — setting them to null afterwards is a no-op.
- `WeakReference<T>` looks like a free lunch for caches but is tricky: entries can vanish mid-request. Use `ConditionalWeakTable<K,V>` when you want a "key defines lifetime" attachment; use `IMemoryCache` for actual caching — it tracks size and age, `WeakReference` does not.
- `GC.KeepAlive(obj)` exists to prevent premature collection when an object's reference has been passed to unmanaged code but no GC-visible rooting exists. Rare but critical in interop scenarios; JIT aggressively shortens local lifetime otherwise.
- The GC can and will move objects. Do **not** cache `IntPtr` addresses of managed objects across potential collection points. Pin with `fixed` or `GCHandle.Alloc(..., Pinned)` — and release pins promptly.
- `ConcurrentGarbageCollection=false` is rarely the right switch. It disables background GC and brings full stop-the-world Gen2 pauses back. Only a memory-super-constrained scenario would benefit, and even then you're trading pause time for nothing measurable.
- **Allocations are not free even if the GC is fast.** A `Gen0` collection is cheap, but every allocation dirties cache lines, grows the ephemeral segment, and increases promotion risk. "Don't allocate in hot paths" is the first-order rule; "trust the GC" applies to cold code.
