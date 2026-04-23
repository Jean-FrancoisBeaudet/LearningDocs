# Span and Memory

_Targets .NET 10 / C# 14. See also: [Stackalloc, inline arrays, pointers](./stackalloc-inline-arrays-pointers.md), [How GC works](./how-gc-works.md), [GC generations](./gc-generation.md)._

`Span<T>` / `ReadOnlySpan<T>` are the two most important performance primitives added to the BCL in the last decade. They let you reference a **contiguous region of memory** — managed array, stack buffer, native pointer, or slice of any of the above — with the same API, zero allocations, and no copying.

`Memory<T>` / `ReadOnlyMemory<T>` are their heap-friendly cousins: same idea, but you can store them in fields, pass them through `async` methods, and hold them across awaits. The cost is that they're more restricted in what they can wrap (no stack memory) and slower to access (extra indirection).

## Shapes

```csharp
// Managed array — no alloc, no copy.
int[] arr = { 1, 2, 3, 4, 5 };
Span<int> s = arr.AsSpan(1, 3);           // {2, 3, 4}
ReadOnlySpan<int> ros = arr;              // whole array, read-only view

// Stack buffer.
Span<byte> stack = stackalloc byte[256];  // on the CURRENT stack frame

// Native pointer (unsafe).
unsafe
{
    byte* p = (byte*)NativeMemory.Alloc(256);
    Span<byte> native = new(p, 256);
}

// String — always read-only (strings are immutable).
ReadOnlySpan<char> part = "hello world".AsSpan(6);   // "world"

// Memory — heap-friendly analog.
Memory<byte> mem = new byte[256];
ReadOnlyMemory<char> rmem = "hello".AsMemory();
```

## `ref struct` — the whole point

`Span<T>` is declared `ref struct`. This one attribute dictates almost every rule:

```csharp
public readonly ref struct Span<T> { ... }
```

**Allowed**:
- Locals and parameters on the stack.
- Fields of other `ref struct`s.
- Returned from methods (with `scoped` rules — see below).

**Forbidden**:
- Fields of regular (non-`ref struct`) classes or structs.
- Boxing (no `object` cast, no use as a `T` in non-`allows ref struct` generics).
- Captured in a lambda or anonymous method.
- Held across an `await` or `yield return`.

The reason: a `Span<T>` pointing to `stackalloc` memory would dangle the moment its frame is popped. Restricting it to stack-only storage makes that impossible at compile time. The compiler enforces it; there's no runtime check to rely on.

As of C# 13, generic methods can opt into accepting `ref struct`s via the `where T : allows ref struct` constraint. Most BCL APIs haven't adopted this yet, so the restriction still bites in practice.

## Slicing

```csharp
Span<int> s = buffer;
Span<int> mid = s[10..20];       // same memory, just a new (ref, length) pair
Span<int> rest = s.Slice(10);    // equivalent

// Read without allocating.
ReadOnlySpan<char> text = "2026-04-22".AsSpan();
int year = int.Parse(text[..4]);
```

Slicing is O(1) and does not copy. The `ref` inside the span advances; the length shrinks. This is the magic that makes parsers, tokenizers, and framers ~10× faster than their `Substring`-based equivalents — you stop allocating intermediate strings.

## `Span` vs `Memory` — when to use which

Use `Span<T>` **when both hold**:
1. Your code is synchronous (no `await` between acquiring and using the span).
2. You don't need to store it anywhere except a local.

Use `Memory<T>` **when either holds**:
1. You need to `await` something while the buffer is in flight.
2. You need to put the buffer in a field — a queue, a pending-request map, a reusable state machine.

```csharp
// Sync parsing — Span everywhere.
public int ParseHeader(ReadOnlySpan<byte> buffer) { ... }

// Async I/O — Memory because ReadAsync awaits.
public async ValueTask<int> ReadChunkAsync(Memory<byte> buffer, CancellationToken ct)
{
    int n = await _stream.ReadAsync(buffer, ct);
    return n;
}

// Inside ReadChunkAsync, if you need fast bit-twiddling post-await:
// obtain a span for the sync portion only.
var span = buffer.Span;
```

The `.Span` property on `Memory<T>` materializes a span — cheap, but not free (it walks the `MemoryManager<T>` / array reference). Do it once per sync block, not per element.

## The `MemoryManager<T>` abstraction

`Memory<T>` is implemented as a `(object, int, int)` tuple: the backing reference, start offset, and length. The object can be:
- `T[]` — a managed array.
- `string` (for `ReadOnlyMemory<char>`).
- A subclass of `MemoryManager<T>` — your own allocator. Own-your-own-buffer scenarios: native memory wrappers, pooled buffer owners, shared-memory interop.

```csharp
public sealed class NativeMemoryOwner : MemoryManager<byte>
{
    private IntPtr _ptr;
    private int _length;
    public NativeMemoryOwner(int size)
    {
        _ptr = Marshal.AllocHGlobal(size);
        _length = size;
    }
    public override Span<byte> GetSpan() => new((void*)_ptr, _length);
    public override unsafe MemoryHandle Pin(int elementIndex = 0)
        => new MemoryHandle((byte*)_ptr + elementIndex);
    public override void Unpin() { }
    protected override void Dispose(bool disposing)
    {
        if (_ptr != IntPtr.Zero) { Marshal.FreeHGlobal(_ptr); _ptr = IntPtr.Zero; }
    }
}
```

In practice, you reach for `ArrayPool<T>` or `MemoryPool<T>` first — both give you an `IMemoryOwner<T>` — and only write a custom `MemoryManager` for unusual backing stores (mmap files, GPU buffers, pinned native buffers).

## `ArrayPool<T>` and `MemoryPool<T>`

Buffer reuse is how you avoid LOH churn:

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(minimumLength: 8192);
try
{
    // IMPORTANT: buf.Length may be >= minimumLength; slice to your need.
    Span<byte> view = buf.AsSpan(0, needed);
    Work(view);
}
finally
{
    ArrayPool<byte>.Shared.Return(buf, clearArray: false);
}
```

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(8192);
Memory<byte> mem = owner.Memory;
// dispose -> returned to pool automatically via using
```

Rules:
- `Rent` returns an array **at least** `minimumLength` — always slice.
- `Return` with `clearArray: true` when the data was sensitive (credentials, tokens). The pool does not zero by default.
- Never hand a rented buffer out past the scope of the `Return` — a second renter will see the same memory. Getting this wrong is a silent corruption bug.
- `ArrayPool<T>.Shared` is process-global. If you need isolation (e.g., to cap memory use per tenant), create a bounded pool with `ArrayPool<T>.Create(maxArrayLength, maxArraysPerBucket)`.

## Spans and strings

Parsing strings without allocating `Substring`s is one of the highest-ROI uses of spans in everyday code:

```csharp
public static bool TryParseDuration(ReadOnlySpan<char> input, out TimeSpan duration)
{
    // e.g., "1h30m", "45s", "2d"
    duration = default;
    long total = 0;
    int i = 0;
    while (i < input.Length)
    {
        int digitStart = i;
        while (i < input.Length && char.IsDigit(input[i])) i++;
        if (digitStart == i) return false;
        if (!long.TryParse(input[digitStart..i], out var n)) return false;

        if (i >= input.Length) return false;
        long mult = input[i] switch
        {
            's' => TimeSpan.TicksPerSecond,
            'm' => TimeSpan.TicksPerMinute,
            'h' => TimeSpan.TicksPerHour,
            'd' => TimeSpan.TicksPerDay,
            _   => -1
        };
        if (mult < 0) return false;
        total += n * mult;
        i++;
    }
    duration = new TimeSpan(total);
    return true;
}
```

All substring access is slicing. Zero allocations. Works on `string`, `char[]`, `stackalloc char[]`, or a `ReadOnlyMemory<char>.Span`.

## `SequenceReader<T>` and `ReadOnlySequence<T>`

Pipelines (`System.IO.Pipelines`) expose data as **multi-segment** `ReadOnlySequence<byte>` — a span is contiguous, but a sequence may be split across pooled buffers. `SequenceReader<T>` lets you parse across segment boundaries:

```csharp
public bool TryReadMessage(ref ReadOnlySequence<byte> buffer, out Message msg)
{
    var reader = new SequenceReader<byte>(buffer);
    if (!reader.TryReadLittleEndian(out int length)) { msg = default; return false; }
    if (reader.Remaining < length) { msg = default; return false; }

    var payload = buffer.Slice(reader.Position, length);
    msg = Parse(payload);
    buffer = buffer.Slice(reader.Position).Slice(length);
    return true;
}
```

Use this for any networking / framing code. It's what Kestrel, SignalR, and gRPC.NET use internally.

## Span safety rules (a.k.a. "scoped")

The compiler enforces lifetime invariants at compile time:

```csharp
Span<int> GetSpan()
{
    Span<int> local = stackalloc int[4];
    return local;                     // ❌ CS8352 — can't escape method
}

Span<int> Echo(Span<int> input) => input;  // ✅ — return with input's lifetime

[UnscopedRef]
Span<int> First(ref MyStruct s) => s.buffer;  // escape the ref, but now s must outlive the span
```

C# 11 introduced the `scoped` and `[UnscopedRef]` modifiers to make these rules explicit. Writing library code that deals heavily in spans, you'll bump into this — the compiler is picky, and for good reason.

## Span equality and search

```csharp
ReadOnlySpan<byte> a = stackalloc byte[] { 1, 2, 3 };
ReadOnlySpan<byte> b = stackalloc byte[] { 1, 2, 3 };
bool eq = a.SequenceEqual(b);                       // true
int idx = b.IndexOf((byte)2);                       // 1
bool starts = b.StartsWith(stackalloc byte[] { 1 }); // true

// For cryptographic comparisons — constant-time:
CryptographicOperations.FixedTimeEquals(a, b);
```

`IndexOf`, `SequenceEqual`, `StartsWith`, `IndexOfAny`, etc., are vectorized (SSE/AVX on x64, NEON on ARM64) and blow past hand-written loops.

## `CollectionsMarshal.AsSpan(List<T>)`

An escape hatch for mutating a `List<T>`'s backing array directly:

```csharp
List<int> list = new() { 1, 2, 3, 4, 5 };
Span<int> view = CollectionsMarshal.AsSpan(list);
view.Reverse();                         // mutates list in place
```

Caveat: any `Add`/`Remove` afterwards invalidates the span (backing array may be reallocated). Great for tight numerical kernels, dangerous if you forget that rule.

## Common anti-patterns

```csharp
// ❌ Don't do this — forces a copy, defeats the point.
byte[] arr = span.ToArray();

// ❌ Don't hold a Span<T> as a field.
class Bad { private Span<byte> _s; }  // compile error

// ❌ Don't await between Span acquisition and use.
Span<byte> view = owner.Memory.Span;
await DoOtherStuff();                 // Span is a local; fine, but patterns vary
// If you await and come back, re-take .Span rather than assuming validity.

// ❌ Don't return Span<T> from an async method — you'd have to store it; impossible.

// ❌ Don't store a Span<T> in an anonymous type or tuple returned from a lambda.
```

## Allocation cost comparison

| Operation | Allocation |
|---|---|
| `new byte[1024]` | ~1 KB on heap, Gen0 |
| `stackalloc byte[1024]` | 0 (stack frame) |
| `"hello".Substring(1, 3)` | ~18 B string on heap |
| `"hello".AsSpan(1, 3)` | 0 |
| `list.ToArray()` | O(n) bytes on heap |
| `CollectionsMarshal.AsSpan(list)` | 0 |
| `ArrayPool.Rent(1024)` on hit | 0 |
| `ArrayPool.Rent(1024)` on miss | ~1 KB on heap, Gen0 |

Spans are not *magic*. They avoid allocations and copies, but they don't make the work go away — walking 10,000 bytes still takes time. The win is in parsers, framers, serialization, and tight loops where the alternative was allocating an intermediate buffer.

**Senior-level gotchas:**
- **Don't "upgrade" an array-based API to a `Span`-based one blindly.** If the caller already has a `byte[]`, the overhead of calling `.AsSpan()` is zero — but if you force every caller to allocate a `byte[]` to create a `Span`, you've made things worse. Add a `ReadOnlySpan<T>` overload *alongside* the existing signatures.
- **`stackalloc` on untrusted length is a stack overflow waiting to happen.** Always bound it: `Span<byte> buf = size <= 256 ? stackalloc byte[256] : new byte[size];` — the "hybrid buffer" pattern.
- **`Span<T>.ToArray()` is a silent perf regression.** Code review should flag it the same as `.ToList()` in a hot path.
- The `ref` returned by `span[i]` **aliases the underlying storage**. Modifying `span[0]` through one `ref` while another `ref` holds the same index is fine for synchronous code but will not survive concurrent mutation. Spans are not thread-safe by default.
- **`Span<T>` equality is by-reference to the backing storage — but `SequenceEqual` is by value.** `a == b` between spans compares the underlying pointer, not the content. Don't confuse the two in tests.
- Using `Memory<T>.Pin()` returns a `MemoryHandle` with an unmanaged pointer. Forgetting to dispose it leaves the backing array pinned for potentially long GCs — a classic heap fragmentation bug. Always `using var h = mem.Pin();`.
- `MemoryPool<T>.Shared` and `ArrayPool<T>.Shared` are not the same type — `Memory<T>` returned from `MemoryPool` comes with an `IMemoryOwner` that wraps the rented array. Don't mix their ownership protocols.
- `MemoryMarshal.CreateSpan` and `MemoryMarshal.AsBytes` are unsafe bridges — they trust the caller about alignment, lifetime, and aliasing. They exist for interop and serialization; they are not general-purpose. Every use deserves a comment.
- Writing a serializer against `ReadOnlySpan<char>`? Remember strings are **UTF-16**. For writing to wire formats (JSON, protobuf, HTTP), you usually want `ReadOnlySpan<byte>` via `Encoding.UTF8.GetBytes(span, destination)` or the `Utf8JsonWriter` / `Utf8Formatter` APIs.
- `Span<T>` does not implement `IEnumerable<T>` (it can't — `ref struct`). `foreach` works because the compiler pattern-matches the `GetEnumerator()` method, not the interface. You can't pass it to LINQ.
- **`allows ref struct` generic constraints** (C# 13) are the long-term fix for the BCL's span awkwardness — expect many more generic APIs (`Task.Run<T>`, helper methods) to gain span support over the next two LTS releases.
