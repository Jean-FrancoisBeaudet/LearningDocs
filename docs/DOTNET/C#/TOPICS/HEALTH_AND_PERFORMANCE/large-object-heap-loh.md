# Large Object Heap (LOH)

_Targets .NET 10 / C# 14. See also: [GC generations (concept)](../../MEMORY_MANAGEMENT/gc-generation.md), [GC generations (perf)](./gc-generations-gen0-gen1-gen2.md), [LOH fragmentation](./loh-fragmentation.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [System.IO.Pipelines](./system-io-pipelines.md), [Pinned objects](./pinned-objects.md)._

The **Large Object Heap** is a separate region of the managed heap reserved for allocations of **85,000 bytes or more**. Everything on the LOH is treated as Gen2 from birth — there's no "Gen0 LOH." This is one of those runtime details that goes unnoticed until a long-running service starts breathing heavily, and then it's everywhere.

## Why the LOH exists, and the 85,000-byte rule

Compacting large objects is expensive — copying a 1 MB array on every Gen2 is wasted work. The LOH is **swept, not compacted by default**, on the theory that big objects are usually long-lived and few in number. The 85,000-byte threshold is hardcoded in the runtime and not configurable in production releases.

The threshold is **bytes allocated**, including the object header (~16 bytes on x64) and method-table pointer. Element-size arithmetic that crosses the boundary:

| Allocation | Element size | Element count | Total | Heap |
|---|---|---|---|---|
| `new byte[85_000]` | 1 | 85,000 | 85,000 | LOH |
| `new int[21_250]` | 4 | 21,250 | 85,000 | LOH |
| `new int[21_249]` | 4 | 21,249 | 84,996 | SOH |
| `new long[10_625]` | 8 | 10,625 | 85,000 | LOH |
| `new Guid[5_313]` | 16 | 5,313 | 85,008 | LOH |
| `new (long, long)[5_313]` | 16 | 5,313 | 85,008 | LOH |
| `new string('x', 42_500)` | 2 (UTF-16) | 42,500 | ~85,016 + header | LOH |
| `new double[10_000]` | 8 | 10,000 | 80,000 | SOH |

The rule is strict: 84,999 bytes goes to SOH, 85,000 to LOH. Pre-sizing collections to *just under* this boundary is a known idiom — but a brittle one (see gotchas).

## What's special about the LOH

Four perf-relevant differences from the SOH:

1. **Collected with Gen2.** Every LOH allocation eventually triggers a full GC. Allocate hot on the LOH and you're paying full Gen2 pause cost on the cadence of LOH allocation rate.
2. **Sweep-only by default.** The runtime free-lists gaps left behind by dead large objects. Variable-sized allocations leave gaps the next allocation may not fit, fragmenting the heap. See [`./loh-fragmentation.md`](./loh-fragmentation.md).
3. **Synchronously zeroed.** Like all managed allocations, large arrays are zero-filled. For a 100 MB array that's 100 MB of memory bandwidth at allocation time — a measurable and *non-amortized* cost.
4. **Allocation cost grows with size.** SOH allocations are bump-pointer fast and constant-time; LOH allocations walk a free list and zero pages. A 10 MB allocation is not 10× a 1 MB allocation in microbenchmarks — it's worse.

## Common LOH offenders

In rough order of how often they appear in real services:

- **`new byte[N]` for I/O buffers**, where N is "big enough." Anything ≥ 85,000 lands here.
- **`MemoryStream` growth.** `MemoryStream` doubles its internal `byte[]` as it writes; the moment it crosses 85k it's on the LOH for the rest of its life. Common in JSON serialization buffers, response body buffering, file uploads.
- **`List<T>` resizing.** Doubling strategy. A `List<int>` past 21,250 items, a `List<long>` past 10,625, a `List<Guid>` or `List<(long,long)>` past 5,313 — all silently land on the LOH after the next resize.
- **`StringBuilder` growth.** Internally chunked, so less of an LOH problem than `MemoryStream`, but `StringBuilder.ToString()` returns a single `string` that *will* be on the LOH if the result is ≥ ~42,500 chars.
- **`string.Concat` / `string.Join` on large content.** Each intermediate is a fresh allocation; the final result is one too.
- **JSON / XML serialization.** `JsonSerializer.Serialize<T>(obj)` returns a `string`; `Utf8JsonWriter` over a `MemoryStream` produces a `byte[]`. Both can grow past 85k for moderately-sized payloads. `System.Text.Json` with `IBufferWriter<byte>` (e.g., `PipeWriter`) avoids this.
- **HTTP body buffering.** `await request.ReadFromJsonAsync<T>()` is fine; manually `await new StreamReader(req.Body).ReadToEndAsync()` allocates a string proportional to body size.
- **Document-DB query results.** RavenDB / Cosmos / Elastic clients can deserialize large result lists into `T[]` or `List<T>` — same resize trap.
- **`Array.Resize` / `Array.Copy` on growing collections.** Each resize is a fresh allocation of the new size.

## Detection

Live signal:

```bash
dotnet-counters monitor System.Runtime --process-id <pid>
# watch:
#   loh-size           - bytes currently in LOH
#   gc-fragmentation   - fragmentation ratio (.NET 7+)
#   gen-2-gc-count     - rises when LOH is allocated heavily
```

Programmatic:

```csharp
GCMemoryInfo info = GC.GetGCMemoryInfo();
long lohSize = info.GenerationInfo[3].SizeAfterBytes;
long pohSize = info.GenerationInfo[4].SizeAfterBytes;
long fragmented = info.FragmentedBytes;
```

Index 3 of `GenerationInfo` is the LOH; 4 is the POH (since .NET 5).

Heavyweight tools:

- **PerfView** — *GC Heap Alloc Stacks*. Filter the stack rollup by allocation size ≥ 85,000; you get the call sites that hit the LOH.
- **dotMemory** — heap snapshots show "Large Object Heap" as a category; expand to see types and retention paths.
- **`dotnet-trace` + `dotnet-dump`** combo — capture a process dump under load, then in `dotnet-dump analyze`:
  ```
  > dumpheap -stat -mt
  > dumpheap -min 85000
  > eeheap -gc
  ```
  These show large object types, sizes, and the LOH segment layout.

## Mitigation playbook

**Default rule: don't allocate large arrays in steady-state code.** Pool, stream, or pre-size. The LOH is for the few unavoidable big allocations; everything else should be off it.

### 1. Pool reusable buffers

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(minimumLength: 100_000);
try
{
    // IMPORTANT: rented array length may exceed requested length
    var view = buffer.AsSpan(0, requestedLength);
    int read = await stream.ReadAsync(buffer.AsMemory(0, requestedLength), ct);
    Process(view[..read]);
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

The `Shared` pool buckets sizes; oversized rents return arrays sized to the next power of two. Treat `buffer.Length` as ≥ requested, not equal to it. See [`./arraypool-of-t.md`](./arraypool-of-t.md).

For very large or specialized buffer profiles, build a private pool: `ArrayPool<byte>.Create(maxArrayLength, maxArraysPerBucket)`.

### 2. Replace `MemoryStream` with `RecyclableMemoryStream`

```csharp
// Microsoft.IO.RecyclableMemoryStream
private static readonly RecyclableMemoryStreamManager s_streamMgr = new(
    new RecyclableMemoryStreamManager.Options
    {
        BlockSize = 128 * 1024,                   // small block pool
        LargeBufferMultiple = 1024 * 1024,        // large buffer pool granularity
        MaximumBufferSize = 16 * 1024 * 1024,
        AggressiveBufferReturn = true,
    });

await using var ms = s_streamMgr.GetStream();
await JsonSerializer.SerializeAsync(ms, payload, ct);
ms.Position = 0;
await ms.CopyToAsync(response.Body, ct);
```

The stream is backed by a small-block + large-buffer pool. Properly tuned, no allocation ever lands on the LOH. Watch the published metrics (`SmallPoolInUseSize`, `LargePoolInUseSize`, `LargeBuffersFree`) to confirm pool effectiveness.

### 3. Pre-size collections

```csharp
// Bad: silent LOH allocation when count grows past the threshold
var ids = new List<long>();
foreach (var row in rows) ids.Add(row.Id);

// Good: one allocation, sized exactly
var ids = new List<long>(rows.Count);
foreach (var row in rows) ids.Add(row.Id);

// Better when count is known and stable: an array
var ids = new long[rows.Count];
for (int i = 0; i < rows.Count; i++) ids[i] = rows[i].Id;
```

Same for `Dictionary<,>(capacity)`, `HashSet<>(capacity)`, `StringBuilder(capacity)`. The capacity argument is **the most common missed perf win** in real codebases.

### 4. Stream instead of buffering

```csharp
// Bad: read entire body into memory
string body = await new StreamReader(req.Body).ReadToEndAsync();
var doc = JsonDocument.Parse(body);

// Good: stream-parse
using var doc = await JsonDocument.ParseAsync(req.Body, cancellationToken: ct);
```

For multi-pass needs, write the response with `Utf8JsonWriter` directly into `PipeWriter` / `IBufferWriter<byte>` from [`./system-io-pipelines.md`](./system-io-pipelines.md).

### 5. POH for long-lived pinned buffers

If a buffer is pinned for the lifetime of the process (Kestrel's I/O buffers, P/Invoke handles, kernel APIs), allocate it on the **Pinned Object Heap** so it's segregated from the SOH compactor's path:

```csharp
byte[] ioBuffer = GC.AllocateArray<byte>(length: 64 * 1024, pinned: true);
// Lives on POH; never moves; doesn't fragment Gen2
```

The POH is collected with Gen2 but never compacted. Long-lived pins on the POH are the *correct* pattern; long-lived pins on the SOH cause heap fragmentation. See [`./pinned-objects.md`](./pinned-objects.md).

### 6. Build a "stay under 85k" knob into your APIs

When the contract is "this method may be called with arbitrary-sized inputs," bound the work in chunks of ≤ 80k bytes so each scratch allocation stays on the SOH. The chunking overhead is tiny next to one LOH-then-Gen2 hit.

## When LOH is fine

It's not zero-tolerance. A **stable, low-cardinality, long-lived** set of large allocations — say, a fixed pool of 10 buffers of 1 MB each, allocated at startup — is *exactly* what the LOH is good at. Pause cost is amortized over process lifetime; fragmentation never happens because allocations don't churn.

The bug shape is **variable-size, high-cadence** LOH allocation — that's what produces fragmentation, full GCs, and pauses.

**Senior-level gotchas:**
- **Strings are 2 bytes per char** internally. The 85k boundary is at ~42,500 characters, not 85,000.
- **`ArrayPool.Shared.Rent(85_000)` may return an array > 85k** and *itself* land on the LOH. The pool won't release that backing array; if you only need 85k once, the pool is now retaining a permanent LOH buffer. Use a private pool sized to your actual needs, or a smaller rent size.
- **`(long, long)` and other 16-byte tuples** hit the LOH at element counts that look small (5,313). Same for `Guid[]`, `(int, int, int, int)`. Always do the math.
- **`List<T>` resize lands on LOH silently** — `.Capacity` doubling produces an interim array that may exceed 85k, plus the new array. Two LOH allocations for one resize.
- **Returning a pooled buffer to the wrong pool** (e.g., a buffer from `ArrayPool<byte>.Create(...)` returned to `ArrayPool<byte>.Shared`) is undefined. Be explicit about which pool owns each buffer.
- **`GC.GetTotalMemory(forceFullCollection: true)`** triggers a full GC. Useful in tests; never on a hot endpoint.
- **`MemoryStream.GetBuffer()` returns the underlying `byte[]`** which may be larger than `Length`. Always slice with `AsSpan(0, (int)stream.Length)`.
- **`StringBuilder.ToString()` always allocates a fresh string** — chunking inside `StringBuilder` doesn't help the final `ToString()` allocation. If you can hand the consumer a `ReadOnlyMemory<char>` instead, do.
- **Async state machines holding a reference to a large pooled buffer** keep that buffer alive across awaits. Forget to `Return` it on the cancellation path and you've leaked one LOH-sized chunk per cancellation.
- **`System.Text.Json` source generators** + `Utf8JsonWriter` over `PipeWriter` is the LOH-free serialization pattern. Reflection-based `JsonSerializer.Serialize<T>(obj)` to a string is not.
- **The LOH grows but never returns to the OS** until the process exits, in older .NET versions. .NET 7+ regions can release LOH segments back, but only after compaction and only if the regions are empty. Long-running services may show high RSS even after the LOH is "free."
