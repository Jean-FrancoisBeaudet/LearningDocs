# System.IO.Pipelines

_Targets .NET 10 / C# 14. See also: [Span&lt;T&gt;](./span-of-t.md), [Memory&lt;T&gt;](./memory-of-t.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [Buffer management](./buffer-management.md), [Channel&lt;T&gt;](./channel-of-t.md), [Streaming patterns](./streaming-patterns.md), [Backpressure handling](./backpressure-handling.md)._

`System.IO.Pipelines` is a high-performance I/O abstraction designed for **byte-stream parsing pipelines** — exactly the kind of workload where naïve `Stream.ReadAsync` loops bleed allocations. Kestrel, SignalR, gRPC, and most binary-protocol servers in .NET sit on it. The premise: separate the producer (network → bytes) from the consumer (bytes → parsed messages) so each can move at its own rate, and share pooled buffers so neither has to copy.

## What problem it solves

The hand-rolled stream parser pattern looks like this:

```csharp
var buffer = new byte[4096];
int leftover = 0;
while (true)
{
    int n = await stream.ReadAsync(buffer.AsMemory(leftover), ct);
    if (n == 0) break;
    int total = leftover + n;

    int consumed = TryParseMessages(buffer.AsSpan(0, total));
    if (consumed < total)
    {
        // a partial message remains; shift it to the front
        Buffer.BlockCopy(buffer, consumed, buffer, 0, total - consumed);
        leftover = total - consumed;
    }
    else leftover = 0;

    // and what if a single message is bigger than 4096? grow the buffer.
}
```

Three problems:

1. **`Buffer.BlockCopy`** to keep partial-message leftovers — a per-read copy.
2. **Manual buffer growth** when one logical message exceeds the buffer size — allocate, copy, repeat.
3. **No back-pressure**: if the parser is slow, the network keeps reading and the buffer grows unboundedly.

`System.IO.Pipelines` fixes all three. The producer and consumer share a slab list of pooled buffers (`ReadOnlySequence<byte>`); the consumer reads, advances past what it consumed, and the producer reuses the freed slabs. If the consumer stalls, the producer is naturally throttled.

## Core types

| Type | Role |
|---|---|
| `Pipe` | The container; owns the buffers and coordinates reader ↔ writer. |
| `PipeWriter` | The producer side. Implements `IBufferWriter<byte>`. |
| `PipeReader` | The consumer side. Yields `ReadResult` containing a `ReadOnlySequence<byte>`. |
| `ReadResult` | `(Buffer: ReadOnlySequence<byte>, IsCanceled, IsCompleted)`. |
| `ReadOnlySequence<T>` | Multi-segment view; may not be contiguous. |
| `SequenceReader<T>` | Stateful reader over `ReadOnlySequence<T>` for parsing. |
| `PipeOptions` | Tuning: pool, thresholds, schedulers. |

## Producer side: `PipeWriter`

```csharp
async Task ProduceAsync(Stream source, PipeWriter writer, CancellationToken ct)
{
    while (true)
    {
        Memory<byte> buffer = writer.GetMemory(sizeHint: 4096);
        int read = await source.ReadAsync(buffer, ct);
        if (read == 0) break;

        writer.Advance(read);

        FlushResult result = await writer.FlushAsync(ct);
        if (result.IsCompleted) break;          // reader finished
    }
    await writer.CompleteAsync();
}
```

`GetMemory(sizeHint)` may return a buffer larger than the hint; `Advance` records how much you actually wrote. `FlushAsync` is the back-pressure point — it returns a `ValueTask<FlushResult>` that pauses the producer if the unconsumed buffered bytes have crossed `pauseWriterThreshold`.

## Consumer side: `PipeReader` and the `Advance` contract

This is the heart of Pipelines and the place beginners get wrong:

```csharp
async Task ConsumeAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadMessage(ref buffer, out Message msg))
            await Handle(msg, ct);

        // Two positions, both required:
        //   consumed: everything before this is gone
        //   examined: we've looked at everything up to here, please bring more
        reader.AdvanceTo(consumed: buffer.Start, examined: buffer.End);

        if (result.IsCompleted)
        {
            if (!buffer.IsEmpty) throw new InvalidDataException("incomplete final message");
            break;
        }
    }
    await reader.CompleteAsync();
}
```

The `consumed` / `examined` distinction is the whole API:

- **`consumed`** is "I've fully processed everything up to here — release these bytes."
- **`examined`** is "I've looked up to here and I need more before I can decide."

The pipe will not return from the next `ReadAsync` until either:

- New bytes have arrived **past `examined`**, or
- The writer has completed.

If you report `examined` somewhere already in the buffer, but `consumed = buffer.Start`, you've created an infinite loop — `ReadAsync` immediately returns the same `ReadOnlySequence` and the parser does the same thing again. **The rule:** if you didn't make forward progress, set `examined = buffer.End` so the pipe waits for more bytes.

`TryReadMessage` typically updates `buffer` to point past the parsed message:

```csharp
static bool TryReadMessage(ref ReadOnlySequence<byte> buffer, out Message msg)
{
    var reader = new SequenceReader<byte>(buffer);
    if (!reader.TryReadLittleEndian(out int length))   { msg = default; return false; }
    if (reader.Remaining < length)                     { msg = default; return false; }

    ReadOnlySequence<byte> body = reader.UnreadSequence.Slice(0, length);
    msg = Message.Parse(body);

    buffer = buffer.Slice(reader.Position).Slice(length);
    return true;
}
```

`SequenceReader<byte>` walks across segments without materialising a `byte[]`. `TryReadTo`, `TryReadExact`, `TryReadLittleEndian/BigEndian` cover the common cases.

## Connecting both ends

```csharp
var pipe = new Pipe(new PipeOptions(
    pool: MemoryPool<byte>.Shared,
    pauseWriterThreshold: 64 * 1024,
    resumeWriterThreshold: 32 * 1024,
    minimumSegmentSize: 4096,
    useSynchronizationContext: false));

// Two independent loops, naturally coordinated through the pipe.
Task producer = ProduceAsync(network, pipe.Writer, ct);
Task consumer = ConsumeAsync(pipe.Reader, ct);
await Task.WhenAll(producer, consumer);
```

`pauseWriterThreshold` / `resumeWriterThreshold` give you back-pressure out of the box — no semaphores, no `Channel.Writer.WaitToWriteAsync`. If the consumer can't keep up, `FlushAsync` simply doesn't complete until the consumer has drained below the resume threshold.

## When to use it (and when not)

| Scenario | Pipelines? |
|---|---|
| Custom binary protocol over TCP / Unix socket | Yes |
| HTTP/2-like framing layer | Yes |
| Parsing a message stream where messages can span reads | Yes |
| Producer → consumer of *objects* (not bytes) | No → [`Channel<T>`](./channel-of-t.md) |
| Reading a single, smallish blob from disk | No → `Stream.ReadAsync(Memory<byte>)` |
| Writing a fixed-format payload to a buffer | No → `IBufferWriter<byte>` directly (`ArrayBufferWriter<byte>`) |

`Channel<T>` and `Pipe` are siblings: `Channel<T>` is for *T* (already-parsed objects), `Pipe` is for bytes. A typical server has both — `PipeReader` parses frames, then writes the resulting objects into a `Channel<Message>` for downstream stages.

## Direct stream adapters

```csharp
PipeReader reader = PipeReader.Create(networkStream);     // wrap any Stream
PipeWriter writer = PipeWriter.Create(networkStream);

Stream s = pipe.Reader.AsStream();                        // back to a Stream
```

`PipeReader.Create(Stream)` is for adapting existing code; it allocates per read for the wrapper. Don't use it on the hottest path. Kestrel exposes `HttpRequest.BodyReader` (already a `PipeReader`) directly — use that, not `HttpRequest.Body`.

## Senior-level gotchas

- **The `consumed` / `examined` rule is the #1 source of Pipelines bugs.** Reporting `examined < buffer.End` while making no forward progress (`consumed = buffer.Start`) is an infinite loop. Reporting `examined = buffer.End` after consuming everything is correct. Reporting `examined` strictly less than `buffer.End` is only valid when you *did* consume something and want the pipe to give you the buffer again with whatever's already there.
- **`PipeReader.AsStream()` is for adapters, not hot paths** — it re-introduces the `byte[] + offset + count` allocation pattern you're trying to escape. The same caution applies to `PipeWriter.AsStream()`.
- **`result.IsCompleted` does not mean the buffer is empty** — the writer can complete with bytes still buffered. Always drain `buffer` before exiting on `IsCompleted`.
- **Calling `ReadAsync` after a completed result returns the same completion forever.** If your loop doesn't check `IsCompleted` and break, it spins.
- **`ReadOnlySequence<byte>` may be multi-segment.** Pattern-matching on `IsSingleSegment` and falling back to `.ToArray()` for the multi-segment case discards the entire benefit. Use `SequenceReader<byte>`.
- **`PipeOptions.UseSynchronizationContext` defaults to `true`** in older .NET versions. For server scenarios always set it `false` (or use `PipeOptions.Default` on .NET 6+ which already does). Otherwise continuations bounce back to a captured context that probably doesn't exist.
- **`pauseWriterThreshold` is in bytes**, not messages. Set it to a multiple of `minimumSegmentSize` to avoid stalling mid-segment. Default is 65,536 / 32,768 — fine for most workloads, but binary protocols with large frames may need it raised.
- **Don't share a `PipeReader`/`PipeWriter` across multiple consumers/producers.** A `Pipe` is single-reader-single-writer by design. Two readers will race the `AdvanceTo` and one will lose data.
- **A `PipeWriter` rented from `Memory<byte>` via `GetMemory` is owned by the pipe**, not the caller. Don't keep a reference past `Advance` / `FlushAsync` — the pipe may reuse the slab.
- **Pipe operations return `ValueTask`** — observe the [single-await rule](../ASYNC_PROGRAMMING/valuetask.md). Don't store a `ValueTask<ReadResult>` and `await` it twice; don't pass it to `Task.WhenAll` without `.AsTask()`.
- **Cancelling `ReadAsync` via `PipeReader.CancelPendingRead()`** sets `result.IsCanceled = true` on the *current* pending read. It does not throw. Code that only checks `IsCompleted` will silently treat a cancellation as a continuation.
- **Kestrel's `HttpContext.Request.BodyReader` is the right surface** for endpoints that parse request bodies in chunks — going through `Request.Body` re-buffers and allocates per read. Same for `Response.BodyWriter`.
- **`SequenceReader<byte>` is a `ref struct`** — it cannot cross `await`. Restructure parsers so the `SequenceReader` is built inside one synchronous helper per `ReadAsync` iteration.
