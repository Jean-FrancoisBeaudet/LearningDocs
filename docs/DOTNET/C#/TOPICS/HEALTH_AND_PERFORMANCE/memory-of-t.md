# Memory&lt;T&gt;

_Targets .NET 10 / C# 14. See also: [Span&lt;T&gt;](./span-of-t.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [Buffer management](./buffer-management.md), [System.IO.Pipelines](./system-io-pipelines.md), [ValueTask](../ASYNC_PROGRAMMING/valuetask.md)._

`Memory<T>` is the heap-friendly counterpart to [`Span<T>`](./span-of-t.md). It's a regular `readonly struct` that holds a reference to backing memory plus an `(int start, int length)` slice, and you materialise a `Span<T>` from it on demand via `.Span`. The whole reason it exists: `Span<T>` cannot live on the heap, cannot cross `await`, cannot be a field on a class. `Memory<T>` can do all three — and pays for it by carrying less power than `Span<T>`.

## Mental model

```text
struct Memory<T> {
    object? _object;   // T[]  |  string  |  MemoryManager<T>  |  null
    int     _index;
    int     _length;
}
```

The object reference is what makes it heap-safe. `Memory<T>.Span` does the work: it inspects `_object`, picks the right pinning strategy, and hands you back a fresh `Span<T>`. That `Span<T>` is then subject to all the usual `ref struct` rules — but only inside the calling frame. The `Memory<T>` itself is free to live in a field, be captured, or be `await`ed past.

## When you actually use `Memory<T>` vs `Span<T>`

| Need | Use |
|---|---|
| Tight, in-frame parsing/formatting | `Span<T>` |
| Async I/O — pass a buffer into `ReadAsync` | `Memory<T>` |
| Field on a class (e.g. a parser holding the current chunk) | `Memory<T>` |
| Captured by a lambda or stored in a closure | `Memory<T>` |
| Returned from an `async` method | `Memory<T>` (or `IMemoryOwner<T>`) |
| SIMD primitive work | `Span<T>` (materialise from `Memory<T>.Span`) |

Rule of thumb: **flow `Memory<T>` between methods, work in `Span<T>` inside one method.**

## Async I/O — the canonical use

The whole modern Stream / Pipelines surface is built around `Memory<byte>` and `ReadOnlyMemory<byte>`:

```csharp
public async ValueTask<int> ReadAsync(Memory<byte> destination, CancellationToken ct = default)
{
    int total = 0;
    while (total < destination.Length)
    {
        int n = await _inner.ReadAsync(destination[total..], ct);
        if (n == 0) break;
        total += n;
    }
    return total;
}
```

The legacy `Stream.ReadAsync(byte[] buffer, int offset, int count, CancellationToken)` overload is now a wrapper around the `Memory<byte>` one. Same with `Socket`, `PipeReader`, `FileStream`. Returning `ValueTask<int>` is part of the pattern — see [`ValueTask`](../ASYNC_PROGRAMMING/valuetask.md).

## Backing stores

A `Memory<T>` can be backed by:

- **`T[]`** — the common case. `array.AsMemory()` or `array.AsMemory(start, length)`.
- **`string`** — only as `ReadOnlyMemory<char>`. `"abc".AsMemory()`. No copy, no allocation.
- **`MemoryManager<T>`** — a custom owner. Use this when the memory comes from somewhere the BCL doesn't know about: a memory-mapped file, a `NativeMemory.Alloc` buffer, a pooled native buffer from a third-party library.

```csharp
sealed class NativeMemoryManager : MemoryManager<byte>, IDisposable
{
    private byte* _ptr;
    private readonly int _length;

    public NativeMemoryManager(int length)
    {
        _ptr = (byte*)NativeMemory.Alloc((nuint)length);
        _length = length;
    }

    public override Span<byte> GetSpan() => new(_ptr, _length);

    public override MemoryHandle Pin(int elementIndex = 0) =>
        new(_ptr + elementIndex, default, this);

    public override void Unpin() { /* always pinned */ }

    protected override void Dispose(bool disposing)
    {
        if (_ptr is not null) { NativeMemory.Free(_ptr); _ptr = null; }
    }
}
```

Then `new NativeMemoryManager(4096).Memory` is a `Memory<byte>` that the rest of the codebase consumes without knowing it's native.

## Ownership: `IMemoryOwner<T>` and `MemoryPool<T>`

`Memory<T>` itself carries **no** ownership information. It's a view, not a lease. To get a buffer plus a "give it back when done" contract, use `IMemoryOwner<T>`:

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(minBufferSize: 4096);
Memory<byte> buffer = owner.Memory;          // length >= minBufferSize, may be larger

int written = await stream.ReadAsync(buffer, ct);
Process(buffer.Span[..written]);
// owner.Dispose() returns the buffer to the pool here.
```

`MemoryPool<byte>.Shared` is implemented on top of [`ArrayPool<byte>.Shared`](./arraypool-of-t.md) — the `IMemoryOwner` wrapper is what flows ownership through async methods cleanly. Returning a raw rented `byte[]` requires a try/finally; returning an `IMemoryOwner<byte>` is just `using`.

The async-friendly producer pattern:

```csharp
public async ValueTask<IMemoryOwner<byte>> LoadAsync(string path, CancellationToken ct)
{
    var info = new FileInfo(path);
    var owner = MemoryPool<byte>.Shared.Rent((int)info.Length);
    try
    {
        await using var fs = File.OpenRead(path);
        int total = 0;
        while (total < info.Length)
            total += await fs.ReadAsync(owner.Memory[total..], ct);
        return owner;     // caller disposes
    }
    catch
    {
        owner.Dispose();
        throw;
    }
}
```

## Pinning for native interop

`Memory<T>.Pin()` returns a `MemoryHandle` that pins the underlying buffer (or carries a native pointer through, if the manager already pinned it). It defeats GC compaction for that object until disposed:

```csharp
using MemoryHandle handle = mem.Pin();
unsafe
{
    NativeApi((byte*)handle.Pointer, mem.Length);
}
```

Keep the pin scope short — pinned objects in Gen-2 fragment the heap and starve the relocating GC. Cross-link [Pinned objects](./pinned-objects.md), [LOH fragmentation](./loh-fragmentation.md).

## `ReadOnlyMemory<T>` is the API contract default

For inputs to public methods, prefer `ReadOnlyMemory<T>` the same way you prefer `ReadOnlySpan<T>` — it widens the set of legal callers (a `ReadOnlyMemory<char>` accepts both a `string` and a `char[]` slice) and signals that the callee won't mutate.

## Senior-level gotchas

- **`Memory<T>` carries no ownership info.** A `Memory<byte>` over an `ArrayPool` rent that's already been returned is a use-after-free with no exception — the pool will hand the same array out to someone else and your two consumers will silently scribble over each other. The only fix is discipline: ownership and views must live and die together. Use `IMemoryOwner<T>` whenever the buffer crosses a method boundary.
- **`Memory<T>` is a struct** — copies are fine and cheap, but a stale copy held in a long-lived field after the underlying owner was disposed is a footgun. Null out fields, or store the `IMemoryOwner<T>` not the `Memory<T>`.
- **Don't store `Memory<char>` over a `string`** — strings are immutable, but `Memory<char>` is the writable variant. Use `ReadOnlyMemory<char>` for the slice; the compiler won't catch mutation through `MemoryMarshal`.
- **`Memory<T>.Span` is not free** — for a `MemoryManager<T>`-backed memory, it calls a virtual method. In a hot loop, hoist the `Span<T>` out and slice that, don't re-call `.Span`.
- **Pinning has a cost beyond GC fragmentation** — a pinned object on the SOH disables compaction *for that segment*. Many short-lived pins are fine; a few long-lived ones can ruin Gen-2. If you need long-lived pinned memory, prefer the Pinned Object Heap (`GC.AllocateUninitializedArray<T>(n, pinned: true)`) or native memory.
- **`MemoryManager<T>` is a `class`** — it allocates. Don't build one per-operation; build one per-resource (per memory-mapped file, per native buffer) and keep it alive.
- **`Memory<T>.Pin()` on an `IMemoryOwner<T>` from `MemoryPool<T>.Shared`** still uses pinning — calling it inside a hot loop holds an SOH array pinned. If you find yourself doing this on every iteration, you've picked the wrong abstraction; either use `NativeMemoryManager` or stop pinning per-iteration.
- **`ReadOnlyMemory<char>` from `string.AsMemory()` shares the string's buffer** — the runtime relies on string immutability to keep this safe. Never `MemoryMarshal.AsMemory(ReadOnlyMemory<char>)` it back to writable; that's a heap-corruption bug waiting to happen.
- **Be careful equating `Memory<T>` with `==`** — it compares by struct field, not by content. Two `Memory<byte>` over different arrays with the same bytes are not equal. Use `MemoryExtensions.SequenceEqual(a.Span, b.Span)`.
