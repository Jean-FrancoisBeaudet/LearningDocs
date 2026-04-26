# Pinned objects

_Targets .NET 10 / C# 14. See also: [How GC works](../MEMORY_MANAGEMENT/how-gc-works.md), [GC generations](../MEMORY_MANAGEMENT/gc-generation.md), [LOH fragmentation](./loh-fragmentation.md), [Low-allocation patterns](./low-allocation-patterns-modern-dotnet.md), [Span and Memory](../MEMORY_MANAGEMENT/span-and-memory.md)._

A **pinned** object is one the GC promises not to move during compaction. Pinning exists because the GC otherwise relocates objects freely — and unmanaged code, kernel APIs, and DMA controllers cannot tolerate a buffer that moves between when the pointer was handed over and when the operation completes. Pinning is the bridge to the unmanaged world. It is also expensive when used wrong: long-lived pins on the small object heap fragment the heap and inflate working set.

## What pinning actually means

Compaction is the GC's "slide everything down to close gaps" phase. A pinned object is left in place; the compactor flows the survivors around it. Several consequences fall out:

- Pinning **does not root** the object. A pinned object that becomes unreachable can still be collected — just not moved while pinned.
- Pinning is **per-GC operation**, not a permanent attribute. Most pin mechanisms are scoped (the `fixed` block, the lifetime of a `MemoryHandle`).
- Pinning **prevents compaction in the region the object lives in**, not the entire heap. But the practical effect is fragmented ephemeral segments.
- Pinning is invisible to allocations: pinned and non-pinned allocations sit interleaved on the SOH unless you specifically allocate to the POH.

## How to pin

```csharp
// 1. fixed — scoped, automatic unpin at block exit. Requires unsafe.
unsafe void HashFixed(byte[] data)
{
    fixed (byte* p = data)
    {
        Native.SHA256(p, data.Length, hash);
    }   // unpinned here
}

// 2. GCHandle — manual; you own the unpin.
var handle = GCHandle.Alloc(data, GCHandleType.Pinned);
try
{
    IntPtr addr = handle.AddrOfPinnedObject();
    Native.SHA256(addr, data.Length, hashAddr);
}
finally
{
    handle.Free();   // forgetting this both pins and roots forever
}

// 3. Memory<T>.Pin() — modern, dispose-based.
using var pin = data.AsMemory().Pin();
unsafe
{
    Native.SHA256((byte*)pin.Pointer, data.Length, hashPtr);
}

// 4. P/Invoke marshaling — implicit, scoped to the call.
[DllImport("crypto")] static extern void SHA256(byte[] data, int len, byte[] hash);
SHA256(data, data.Length, hash);   // marshaller pins for the duration of the call only
```

`fixed` is the right answer when the pin's scope is a single method body. `GCHandle.Alloc(..., Pinned)` is for buffers held across calls (and across `await`s) — but only if you really need to. P/Invoke implicit pins are usually fine: the marshaler pins for the call and unpins on return.

## Why long-lived pins are expensive

Imagine the SOH ephemeral segment as a row of variable-sized blocks:

```
| live | dead | live | PINNED | dead | live | dead |
```

After a Gen0 collection, the GC wants to compact:

```
| live | live | PINNED | live |   ...   free   ...    |
```

It cannot move the pinned block. The compactor flows around it. You're left with a **gap** before the pinned object that can only be reused for objects small enough to fit. As more pins accumulate over time, the heap grows more pockmarked: large allocations skip the gaps, the ephemeral segment expands, working set climbs, eviction rates worsen.

Symptoms in production:

- Process working set steady-state grows over hours/days while live-set is stable.
- `gc-fragmentation` counter climbs.
- Gen2 size large with low live-set ratio (visible via `GC.GetGCMemoryInfo()`).
- More frequent Gen2s as the runtime tries to recover space.

## The Pinned Object Heap (POH)

.NET 5 introduced a dedicated heap region for long-lived pinned buffers. The POH is:

- **Never compacted.** Pinning on POH is therefore free of fragmentation cost — the heap was never going to move anything anyway.
- **Collected with Gen2.** Allocations live a long time by design; that's the whole point.
- **Allocated explicitly** via `GC.AllocateArray<T>(int len, bool pinned: true)` or the uninitialized variant `GC.AllocateUninitializedArray<T>(int len, bool pinned: true)`.

```csharp
// Long-lived I/O ring buffer — straight to POH, never moves.
private static readonly byte[] RxBuffer = GC.AllocateArray<byte>(64 * 1024, pinned: true);

// Uninitialized variant skips the zero-fill (must overwrite before first read).
byte[] scratch = GC.AllocateUninitializedArray<byte>(8192, pinned: true);
```

Practical uses:

- IOCP / overlapped I/O buffers registered with the kernel for long-running socket reads.
- gRPC / Kestrel internals (Microsoft's high-perf libraries use POH-backed pools).
- DMA buffers for hardware drivers (rare, but the use case POH was designed for).
- Long-lived crypto scratch space passed to native libraries.

## When to use what

| Scenario | Use |
|---|---|
| Short, scoped P/Invoke buffer (one call, returns quickly) | `fixed` |
| P/Invoke where marshaler can do it | implicit (just pass `byte[]`) |
| Buffer passed to async I/O, held across awaits | `GCHandle.Alloc(..., Pinned)` or POH |
| Long-lived I/O buffer registered with kernel/IOCP | POH (`GC.AllocateArray(..., pinned: true)`) |
| Networking ring buffer, persistent scratch | POH |
| Pinning a `string` to read characters | `fixed (char* p = s)` (read only — never mutate) |
| Any pin held > a few microseconds across other allocations | POH |

Rule of thumb: **scoped → `fixed`; long-lived → POH; in between → measure.**

## Diagnosing pinning trouble

Tools, in order of how often you'll reach for them:

- **`dotnet-counters monitor System.Runtime`** — watch `gc-fragmentation` and POH size. A growing POH with stable working set is fine; a growing fragmentation counter on a stable workload is a smell.
- **PerfView** → "GC Heap Alloc Stacks" with the Pin Count view. Shows which call sites are pinning and how long pins live.
- **`dotnet-dump analyze` / WinDbg + SOS** — `!gchandles -type Pinned` lists every pinned handle with the type pinned. `!eeheap -gc` shows segment layout including pinned ranges.
- **`GC.GetGCMemoryInfo()`** programmatic — `PinnedObjectsCount` and `FragmentedBytes` give a single number for dashboards.
- **dotMemory** — has a "Pinned objects" view that walks GCHandles and POH.

A typical leak: someone called `GCHandle.Alloc(buf, GCHandleType.Pinned)` and either threw before `Free()` or stored it in a long-lived dict that's never cleaned up. The buffer is rooted (the handle counts as a root) **and** pinned — fragments forever, never collected.

## Patterns that combine well

```csharp
// Pool of POH-backed buffers — pin-cost-free pool for I/O.
public sealed class PinnedBufferPool
{
    private readonly ConcurrentBag<byte[]> _bag = new();
    private readonly int _size;

    public PinnedBufferPool(int size) => _size = size;

    public byte[] Rent() => _bag.TryTake(out var b) ? b : GC.AllocateArray<byte>(_size, pinned: true);
    public void Return(byte[] b) { Array.Clear(b); _bag.Add(b); }   // no risk of move; can hand back to native code repeatedly
}
```

This is roughly what Kestrel and `System.IO.Pipelines` do under the hood for socket I/O.

**Senior-level gotchas:**

- `GCHandle.Alloc(..., Pinned)` **must** be paired with `Free()` in a `finally`. A leaked pinned handle pins the object **and** keeps it rooted (the handle itself is a root). The GC has no way to recover.
- `fixed` only pins for the block. Re-entering the block re-pins; storing the pointer past `}` is undefined behavior. The JIT may even reuse the slot for something else.
- `Memory<T>.Pin()` returns a `MemoryHandle` — must be disposed. `using var pin = mem.Pin();` is the safe form.
- Pinning a `string` with `fixed (char* p = s)` pins the underlying char array used by the string object. **Never write through that pointer** — strings are interned, the same address may be shared across the process, and mutation corrupts every other reference. Read-only access is the only safe use.
- POH is collected only with Gen2. If you churn allocations on POH (rent/return constantly via `GC.AllocateArray<…>(pinned: true)`), every one becomes a future Gen2 trigger. POH is for **long-lived** buffers; pool them.
- `ArrayPool<T>.Shared` does NOT allocate from POH. If you need pinned pool buffers, allocate to POH yourself and pool them — or use `System.IO.Pipelines`, which manages a pinned buffer pool internally.
- `.NET Framework` had no POH. Pinning advice from pre-2020 sources (and many StackOverflow answers) is dated — they recommend strategies that exist precisely to work around the missing POH.
- The marshaler may copy small managed types instead of pinning them. `[In, Out] byte[]` parameters with `[MarshalAs(UnmanagedType.LPArray)]` may be marshaled by **bcopy**, not by pinning. Confirm with `[SuppressGCTransition]` + a profiler if perf matters.
- `Memory<T>` over **native** memory uses a custom `MemoryManager<T>`, not pinning — there's nothing to pin since nothing is managed. `Memory<T>.Pin()` then is a no-op pointer fetch.
- **Watch for nested pins.** Code that pins inside a callback that itself ran inside another `fixed` block can hold a pin for far longer than its author realized. A trace timeline reveals these; a stack trace at the pin point does not.
