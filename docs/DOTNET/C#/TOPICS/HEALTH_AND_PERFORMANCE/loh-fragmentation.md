# LOH fragmentation

_Targets .NET 10 / C# 14. See also: [Large Object Heap](./large-object-heap-loh.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [Pinned objects](./pinned-objects.md), [dotnet-dump](./dotnet-dump.md), [profiling-perfview](./profiling-perfview.md)._

The LOH is **swept, not compacted**, by default. When a large object dies, its bytes become a free-list entry; they don't slide together. Allocate variable-sized large objects long enough and the heap looks like a comb: lots of free space in total, but no single gap big enough for the next allocation. That's fragmentation.

The end-state is unpleasant: a long-running service with **rising RSS, a stable live set, and eventually `OutOfMemoryException` for an allocation the heap "should" have been able to satisfy.**

## What the heap actually looks like

Imagine the LOH as a sequence of segments (or, on .NET 7+, regions):

```
[ A: 1 MB live ][ free 200 KB ][ B: 500 KB live ][ free 150 KB ][ C: 800 KB live ]
```

Total free = 350 KB. Now the app tries to allocate a 250 KB array. The 200 KB gap is too small; the 150 KB gap is too small. The runtime walks the free list, fails to fit, and either:
- Extends the LOH segment (RSS grows; OS may not be able to back it).
- On .NET 7+ regions, allocates a new region.
- If the heap is already at `GCHeapHardLimit` or the OS can't commit, throws `OutOfMemoryException`.

This is **fragmentation pressure**, distinct from genuine memory exhaustion: the *bytes* are there, the *layout* isn't.

## Symptoms

The signature pattern across hours or days of a long-running service:

- **Process RSS climbs steadily** over hours.
- **`gc-heap-size` grows**, but `loh-size` (live LOH bytes) is roughly stable.
- **`gc-fragmentation`** counter (.NET 7+) trends up.
- **`GC.GetGCMemoryInfo().FragmentedBytes`** rises monotonically.
- **Eventually**, `OutOfMemoryException` from a `new byte[N]` or `List<T>` resize that succeeded an hour ago.
- Pause times grow because each Gen2 has to walk a longer and longer free list.

If you only see RSS growth without these LOH-specific signals, you have a **leak** ([`./gc-pressure.md`](./gc-pressure.md) discusses the distinction). Fragmentation requires LOH-sized live churn.

## Detection workflow

### Live, in production

```bash
dotnet-counters monitor System.Runtime --process-id <pid>
# columns to chart over time:
#   loh-size           - live LOH bytes
#   gc-heap-size       - total managed heap
#   gc-fragmentation   - explicit fragmentation ratio (.NET 7+)
```

A divergence — `loh-size` flat, `gc-heap-size` and `gc-fragmentation` rising — is the diagnostic.

```csharp
// Quick endpoint for /metrics
GCMemoryInfo info = GC.GetGCMemoryInfo();
return Results.Ok(new
{
    HeapBytes        = info.HeapSizeBytes,
    FragmentedBytes  = info.FragmentedBytes,
    LohLiveBytes     = info.GenerationInfo[3].SizeAfterBytes,
    LohFragBytes     = info.GenerationInfo[3].FragmentationAfterBytes,  // .NET 7+
    PohLiveBytes     = info.GenerationInfo[4].SizeAfterBytes,
});
```

### From a process dump

When live counters confirm fragmentation, capture and inspect a dump:

```bash
dotnet-dump collect --process-id <pid>
dotnet-dump analyze <dumpfile>

> eeheap -gc           # heap layout, segment sizes, free-list bytes per segment
> dumpheap -stat       # types by total size; sort by largest
> dumpheap -min 85000  # all LOH-resident objects
> gcroot <addr>        # who's keeping a particular object alive
```

`!eeheap -gc` shows free-list bytes per segment — the visualization of fragmentation. `!dumpheap -min 85000 -type System.Byte[]` is usually how the bad-actor type is found.

### Visual tooling

- **dotMemory** — heap snapshot, *Generations* view, expand "Large Object Heap." Switch to the *Fragmentation* tab to see free vs occupied per segment.
- **PerfView** — *GC Heap Alloc Stacks* + filter ≥ 85k bytes shows the *call sites* producing churn. *GC Stats* shows fragmentation as a percentage in the GC summary.

## Why it happens

LOH fragmentation requires **variable-sized, churning** large allocations. Common shapes:

- A service that buffers each request body into a `MemoryStream`, where bodies range from 100 KB to 5 MB. Each request allocates and frees a different size.
- A document-DB client that returns query results as `T[]` of varying length per query.
- An import job that allocates a fresh `byte[]` per file, sized to the file.
- A reporting service generating PDFs / spreadsheets where the output `MemoryStream` grows to a different final size each time.

Steady-state, **same-size** LOH allocations don't fragment — gaps left by dead objects are exactly the size of new allocations. That's why the most effective mitigation is to *make all your LOH allocations the same size*.

## Mitigations, in order of preference

### 1. Don't allocate on the LOH at all

The cheapest fix is to not have the problem. The LOH-avoidance toolkit:

- `ArrayPool<T>` for buffers (see [`./arraypool-of-t.md`](./arraypool-of-t.md)).
- `RecyclableMemoryStream` (Microsoft.IO) instead of `MemoryStream`.
- Pre-size `List<T>`, `Dictionary<,>`, `StringBuilder` to known counts.
- Stream large I/O instead of buffering: `IBufferWriter<byte>`, `PipeReader`/`PipeWriter`, `JsonDocument.ParseAsync` over a stream.
- Return `IAsyncEnumerable<T>` from query layers instead of `Task<List<T>>`.

This is the **only mitigation that works long-term** for high-throughput services.

### 2. Bucket your LOH allocations to fixed sizes

If you must allocate on the LOH, allocate **same-shape** buffers. Round up to the next power of two and rent from a private pool keyed by bucket:

```csharp
private static readonly ArrayPool<byte> s_pool = ArrayPool<byte>.Create(
    maxArrayLength:    16 * 1024 * 1024,   // cap
    maxArraysPerBucket: 32);

public byte[] RentForBody(int contentLength)
{
    // Always rent the next power of two; gaps left by returned buffers
    // exactly fit subsequent rents of the same bucket.
    int size = (int)BitOperations.RoundUpToPowerOf2((uint)Math.Max(contentLength, 4096));
    return s_pool.Rent(size);
}
```

Same-bucket rent/return cycles produce **same-sized free gaps** that the next rent fills exactly. Fragmentation never accumulates.

### 3. Manual LOH compaction (last resort)

The runtime supports a one-shot LOH compact. It's a full Gen2 with the LOH compacted in place — pause cost is proportional to total heap size. This is **not** a tuning knob; it's an emergency lever.

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(generation: 2,
           mode: GCCollectionMode.Aggressive,
           blocking: true,
           compacting: true);
```

`CompactOnce` self-resets to `Default` after one collection — you must set it again next time. `Aggressive` mode (.NET 7+) also returns committed segments to the OS, useful if you're trying to recover RSS after a batch phase.

When this is acceptable:
- **Off-peak windows** (a maintenance heartbeat that runs during the daily quiet hour).
- **Between batch phases** (a long-running job has just finished phase 1, will start phase 2 in 30 seconds, and would otherwise drag accumulated fragmentation into phase 2).
- **At process startup** after warmup, if the warmup itself produced fragmentation.

When it is **not** acceptable:
- Inside a request handler.
- On a timer with a frequency under "occasionally."
- As a workaround for not pooling buffers — fix the allocation pattern instead.

### 4. POH for permanent pinned buffers

If the actual root cause of fragmentation is **pinning** (long-lived `GCHandle.Alloc(..., Pinned)` or `Memory<T>.Pin()` keeping objects from compacting), allocate them on the POH:

```csharp
byte[] ioBuffer = GC.AllocateArray<byte>(64 * 1024, pinned: true);
```

The POH is its own segment, never compacted but never fragmenting Gen2 either. See [`./pinned-objects.md`](./pinned-objects.md).

## Validation

Fixing fragmentation needs proof, not vibes. Capture before / after:

```csharp
public static FragmentationSnapshot Capture()
{
    GCMemoryInfo info = GC.GetGCMemoryInfo();
    return new(
        HeapBytes:       info.HeapSizeBytes,
        FragmentedBytes: info.FragmentedBytes,
        LohBytes:        info.GenerationInfo[3].SizeAfterBytes,
        LohFragBytes:    info.GenerationInfo[3].FragmentationAfterBytes);
}
```

After deploying a fix, verify over hours of production load:
- `LohBytes` stable (or lower).
- `FragmentedBytes` stable (not growing).
- `gc-heap-size` no longer climbing in step with uptime.
- p99 latency within SLO during Gen2 collections.

If you ran a `CompactOnce`, log pause time observed for that GC and confirm it stayed within an acceptable window — a one-off compact on a 4 GB heap can take hundreds of milliseconds.

**Senior-level gotchas:**
- **`CompactOnce` is one-shot.** Setting it once does not enable continuous compaction. Code that "schedules a compaction every hour" must re-set the flag every time.
- **`GC.Collect` in production is almost always wrong.** The fragmentation-compact case is the one narrow exception, and it must be at a known quiet boundary. If you can't name that boundary, don't ship the call.
- **`ArrayPool.Shared.Rent` is not a panacea.** The shared pool buckets up to 1 MB, then falls through to plain allocation; oversized rents bypass the pool entirely and themselves land on the LOH. For requests > 1 MB, build a private pool with `ArrayPool<T>.Create`.
- **The shared pool also retains rented arrays indefinitely.** Once you rent a 256 KB buffer, that bucket holds it for the process lifetime. This is fine for steady-state services, harmful for short-lived utilities — pick the right pool for the lifetime.
- **`RecyclableMemoryStream` has its own LOH risk** if `LargeBufferMultiple` is misconfigured. Tune to your actual peak buffer size; monitor the manager's metrics.
- **32-bit processes hit address-space exhaustion long before byte exhaustion.** A fragmented LOH on a 32-bit process can fail at well under 2 GB. Use 64-bit. (If you can't, this is a hard problem with no good answer beyond compaction discipline.)
- **Background GC does not compact the LOH.** Even with BGC enabled, LOH compaction only happens with `CompactOnce` + a foreground Gen2.
- **`GC.GetGCMemoryInfo().FragmentedBytes`** is **whole-heap** fragmentation, not LOH-specific. Use `GenerationInfo[3].FragmentationAfterBytes` for LOH only (.NET 7+).
- **Pinning fragments the SOH, not just the LOH.** Long-lived pins anywhere block compaction; if the root cause of fragmentation is pinning, moving the pin to the POH is the fix, not LOH compaction.
- **Compacting the LOH costs more than compacting Gen2** because objects are large and the runtime has to copy them. A 1 GB LOH with 30% fragmentation can pause hundreds of ms even with `Aggressive` and `blocking: true`. Plan accordingly.
- **A "good" fragmentation ratio is < ~10%** of the LOH size on a steady-state service. Anything trending toward 30%+ is broken — even if it hasn't OOMed yet.
