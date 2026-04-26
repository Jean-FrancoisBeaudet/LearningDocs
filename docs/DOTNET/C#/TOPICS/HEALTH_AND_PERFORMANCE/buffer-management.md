# Buffer management

_Targets .NET 10 / C# 14. See also: [Span&lt;T&gt;](./span-of-t.md), [Memory&lt;T&gt;](./memory-of-t.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [System.IO.Pipelines](./system-io-pipelines.md), [Large Object Heap (LOH)](./large-object-heap-loh.md), [LOH fragmentation](./loh-fragmentation.md), [Backpressure handling](./backpressure-handling.md)._

"Buffer management" is the umbrella for the decisions that turn allocation-heavy I/O into allocation-light I/O: who owns the bytes, how big a slab is, when a buffer is reused, and how data flows from producer to consumer without copies. The pieces are in the toolbox; the skill is wiring them together.

## The toolbox at a glance

| Tool | Lifetime | Crosses async? | Pooled? | Use for |
|---|---|---|---|---|
| `T[]` (`new T[n]`) | GC | yes | no | One-off small buffers; nothing else |
| `stackalloc T[n]` | stack frame | no | n/a | Tight in-frame buffers ≤ ~256–1024 bytes |
| `Span<T>` | view; stack-only | no | n/a | In-frame work over any backing |
| `Memory<T>` | view; heap-safe | yes | n/a | Async-flowing buffer view |
| `ArrayPool<T>.Rent` | leased | yes (with care) | yes | Hot-path arrays, sized to workload |
| `MemoryPool<T>.Rent` | leased (`IMemoryOwner<T>`) | yes | yes | Async-flowing ownership |
| `ArrayBufferWriter<T>` | owned by caller | n/a | grows | Build-up writes when final size unknown |
| `RecyclableMemoryStream` | leased (`Dispose`) | yes | yes | Drop-in `MemoryStream` replacement |
| `Pipe` (`PipeReader`/`PipeWriter`) | leased through pipe | yes | yes | Network/parser pipelines with back-pressure |

## Decision tree

```text
Need a buffer for...

  in-frame, ≤ a few KB, never escapes?            stackalloc
  in-frame, larger or sized at runtime?           ArrayPool.Rent + try/finally
  field on a class / crosses await / closure?     Memory<T> from IMemoryOwner<T>
  build up of unknown size, hand off when done?   ArrayBufferWriter<T>
  legacy code wants Stream?                       RecyclableMemoryStream
  network producer + parser consumer?             System.IO.Pipelines
```

## The LOH cliff at 85,000 bytes

Any allocation ≥ 85,000 bytes (~83 KiB) lands on the [Large Object Heap](./large-object-heap-loh.md). LOH allocations are not compacted by default — they fragment, they live across Gen-2 collections, and they make `GC.GetTotalMemory(false)` look stable while real working set climbs. Two consequences:

- Anything routinely above that threshold **must** be pooled. Allocate-and-discard is a documented production fire.
- `byte[]` of exactly 85,000 starts on LOH; element-size-dependent for value-type arrays. The threshold is by **byte size**, not element count.

`ArrayPool<T>.Shared` pools up to ~1 MiB per array; for buffers between 1 MiB and your workload's ceiling, build a dedicated `ArrayPool<T>.Create(maxArrayLength: yourCeiling, ...)`.

## `IBufferWriter<T>` — the unifying write surface

`IBufferWriter<T>` is the consumer interface every modern .NET writer implements:

```csharp
public interface IBufferWriter<T>
{
    void Advance(int count);
    Memory<T> GetMemory(int sizeHint = 0);
    Span<T>   GetSpan(int sizeHint = 0);
}
```

Pattern:

```csharp
public static void WriteHeader(IBufferWriter<byte> writer, int length, byte type)
{
    Span<byte> span = writer.GetSpan(5);  // ask for at least 5 bytes
    BinaryPrimitives.WriteInt32LittleEndian(span, length);
    span[4] = type;
    writer.Advance(5);
}
```

The same code targets `ArrayBufferWriter<byte>` (in-memory build-up), `PipeWriter` (network pipeline), or any custom writer (e.g., a `BufferWriterSlim<byte>` from a third-party perf library). This is what makes serializers like `System.Text.Json` and `MessagePack-CSharp` zero-allocation on the write path.

## `ArrayBufferWriter<T>` for unknown-final-size build-up

```csharp
var writer = new ArrayBufferWriter<byte>(initialCapacity: 1024);
JsonSerializer.Serialize(new Utf8JsonWriter(writer), payload);
ReadOnlyMemory<byte> result = writer.WrittenMemory;   // exact length

await stream.WriteAsync(result, ct);
// writer goes out of scope; its array is GC'd. For hot paths,
// consider PipeWriter or RecyclableMemoryStream instead.
```

The growth strategy is "double when needed." For predictable sizes, pre-size with `initialCapacity` to avoid intermediate copies.

## `RecyclableMemoryStream` — the `MemoryStream` replacement

`Microsoft.IO.RecyclableMemoryStream` (NuGet: `Microsoft.IO.RecyclableMemoryStream`) is a pooled, slab-based `MemoryStream` that solves three problems with `MemoryStream`:

- `MemoryStream` doubles its array on every `Write` past the capacity — every doubling is a fresh allocation + copy + LOH risk.
- `MemoryStream.ToArray()` allocates a fresh array of exactly the written length.
- The internal array is never returned to a pool.

```csharp
private static readonly RecyclableMemoryStreamManager Manager = new();

using var ms = Manager.GetStream();
JsonSerializer.Serialize(ms, payload);
await ms.CopyToAsync(network, ct);
// Underlying slabs return to the pool on Dispose.
```

Used inside ASP.NET Core, Azure SDKs, and most high-throughput .NET services. If your code says `new MemoryStream()` on a hot path, this is almost always a free win.

## Slab vs single-buffer

When a parser may need to keep "the previous chunk" while reading the next (e.g., a logical message spans reads), **don't grow a single buffer**. Two reasons:

- Each grow is `new byte[n*2]` + `Array.Copy` + GC pressure on the old one.
- The buffer can balloon past the LOH threshold and stay there for the whole connection.

A slab list keeps each chunk pool-sized:

- `ReadOnlySequence<byte>` (used by [`PipeReader`](./system-io-pipelines.md)) is a linked structure of slabs that supports `SequenceReader<byte>` for parsing across boundaries without materialising a contiguous array.
- For your own parsers, take `ReadOnlySequence<byte>` as input where possible — accept that the data may be multi-segment.

## Sizing — the worst-case rule

The fastest pool rental is the one that fits in one bucket. Rent for the **worst expected size**, not for what you have *now*:

```csharp
// Bad: rent grows over the request, every grow is rent+copy+return.
byte[] buf = pool.Rent(64);
while (more) {
    if (need > buf.Length) {
        var bigger = pool.Rent(buf.Length * 2);
        buf.AsSpan(0, written).CopyTo(bigger);
        pool.Return(buf);
        buf = bigger;
    }
    Append(buf, ref written);
}

// Good: rent once at the protocol max.
byte[] buf = pool.Rent(MaxFrameSize);
```

If you genuinely don't know the ceiling, you almost certainly want [Pipelines](./system-io-pipelines.md), not a hand-rolled grow loop.

## Back-pressure is a buffer concern

Unbounded buffers don't run out of memory linearly — they exhaust it suddenly when a consumer stalls. Every buffer in a pipeline is a queue, and every queue needs a bound + a back-pressure signal. Cross-link [Backpressure handling](./backpressure-handling.md). The `Pipe` exposes this directly via `pauseWriterThreshold` / `resumeWriterThreshold`; for `Channel<T>`, use `BoundedChannelOptions`.

## Sensitive data

Pooled buffers are visible across requests. Anything that holds credentials, tokens, PII, or healthcare data must:

- `clearArray: true` on `ArrayPool.Return`.
- `CryptographicOperations.ZeroMemory(span)` for explicit zeroing of `Span<byte>` regions before release (handles compiler/JIT elision).
- Never log a `ToHexString(span)` of a buffer until you know what's in it.

## Senior-level gotchas

- **`MemoryStream` always allocates and may LOH-promote.** Replace with `RecyclableMemoryStream` on every hot serialization path. The change is one line at construction.
- **`StringBuilder` is not a buffer pool** — it's a chunk chain that hides allocations. Each `Append` past the current chunk allocates a new chunk twice the size. For high-throughput text building, write directly to a `Span<char>` or `IBufferWriter<char>`.
- **`BinaryWriter` over a `MemoryStream` doubles allocations** vs writing to a `Span<byte>` via `BinaryPrimitives.Write*` — the BinaryWriter encodes one field at a time and goes through the stream's virtual `Write(byte[], ...)` overload.
- **Do not grow rented buffers** — every grow is rent + copy + return. Size to the worst case once.
- **Ownership belongs to one place at a time.** A `Memory<byte>` flowing through three layers needs *one* code path that owns the rental and disposes it; the others get views. Splitting ownership ("the parser will return it") is a bug magnet.
- **`IBufferWriter<T>.GetSpan(sizeHint)` may return more than `sizeHint`.** Don't store `span.Length` as your buffer size — use the count you actually wrote with `Advance`.
- **`ArrayBufferWriter<T>.Clear()` resets but keeps the buffer** — good for reuse within a method, but the buffer it holds can be arbitrarily large from a prior write. Don't keep `ArrayBufferWriter` instances on long-lived fields without bounding.
- **A "buffer" backed by `new byte[n]` on every request is the default in legacy serialisers.** Profile with `dotnet-counters` `gc-heap-size` and `gen-0-gc-count` — if either is climbing under steady load, the buffer strategy is the first thing to fix.
- **LOH fragmentation hides** — total committed memory can grow even as logical heap stays flat. Track `working-set` alongside `gc-heap-size`. See [LOH fragmentation](./loh-fragmentation.md).
- **`ReadOnlySequence<T>` is multi-segment** — code that pattern-matches on `sequence.IsSingleSegment` and falls back to `.ToArray()` for the multi-segment case has thrown the entire benefit away. Use `SequenceReader<T>` instead.
