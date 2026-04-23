# Stackalloc, inline arrays, pointers

_Targets .NET 10 / C# 14. See also: [Span and Memory](./span-and-memory.md), [How GC works](./how-gc-works.md)._

Three tools for doing allocation-free work with contiguous memory:

| Tool | Where it lives | Lifetime | Safe code? |
|---|---|---|---|
| `stackalloc T[n]` | Current stack frame | Until method returns | Yes (since C# 7.2, when assigned to `Span<T>`) |
| `[InlineArray]` (.NET 8+) | Inside the containing struct | Lifetime of the struct | Yes |
| Pointers (`T*`, `fixed`) | Anywhere you put them | You own it | No — requires `unsafe` |

Each exists because the GC is expensive on the hot path and sometimes you need memory that doesn't participate in GC at all — or only transiently.

## `stackalloc`

`stackalloc` allocates memory on the **current stack frame**. It's free: a single subtraction on `rsp`. It disappears when the method returns.

```csharp
// Raw (unsafe) form — a pointer.
unsafe
{
    int* buf = stackalloc int[16];
    buf[0] = 42;
}

// Safe form (C# 7.2+) — assigns to Span<T>.
Span<int> buf = stackalloc int[16];
buf[0] = 42;

// Collection-expression form (C# 12).
ReadOnlySpan<int> primes = stackalloc[] { 2, 3, 5, 7, 11 };
```

### The size trap

**The stack is small** — typically 1 MB on Windows threads, 8 MB on Linux. A bad `stackalloc` on an untrusted size is a `StackOverflowException`, which **cannot be caught** and terminates the process.

The hybrid buffer pattern:

```csharp
const int MaxStack = 256;
Span<byte> buffer = length <= MaxStack
    ? stackalloc byte[MaxStack]
    : new byte[length];
buffer = buffer[..length];
```

- For `length <= 256`, use the stack (fast, no allocation).
- For larger inputs, fall back to a heap array (or `ArrayPool`).
- Always slice the buffer to the logical length, because both branches may return more than `length` bytes.

The constant is workload-dependent — 256 to 1024 bytes is typical. Put the `stackalloc` at the **top** of the method, not inside a loop, so the frame size is fixed and only one grow happens.

### `stackalloc` inside a loop

**Don't.** `stackalloc` inside a loop keeps growing the stack frame (or the JIT hoists it up, or it warns and refuses). The compiler emits a warning (`CS9081` in modern .NET). Hoist it out:

```csharp
// ❌ Bad
foreach (var item in items)
{
    Span<byte> scratch = stackalloc byte[64];
    Process(item, scratch);
}

// ✅ Good — one allocation, reused
Span<byte> scratch = stackalloc byte[64];
foreach (var item in items)
{
    scratch.Clear();
    Process(item, scratch);
}
```

### `stackalloc` and security

`stackalloc` does **not zero** the memory in release builds by default. On .NET, the runtime zeros stack memory for safety unless you opt out via `[SkipLocalsInit]`:

```csharp
[SkipLocalsInit]
public static int FastParse(ReadOnlySpan<char> s)
{
    Span<byte> scratch = stackalloc byte[512]; // contents are indeterminate
    // You MUST write before reading.
}
```

`SkipLocalsInit` shaves nanoseconds off the call by skipping the `memset`. Use it only in measured hot paths where you overwrite the buffer before reading. Getting it wrong means reading whatever was on the stack last — potentially other callers' secrets.

## Inline arrays (`[InlineArray(N)]`, .NET 8 / C# 12)

An **inline array** lets a struct embed a fixed-size array of `T` directly, in sequence, without allocation, without `fixed`, and without `unsafe`. It's essentially C's `T field[N];` for .NET.

```csharp
[System.Runtime.CompilerServices.InlineArray(8)]
public struct Buffer8<T>
{
    private T _element0;    // ONE field; compiler duplicates layout

    // Usage doesn't go through _element0; use indexers.
}

var buf = new Buffer8<int>();
buf[0] = 42;
buf[7] = 99;

// Convert to span for bulk operations.
Span<int> span = buf;   // implicit — provided by runtime helpers
span.Fill(0);
```

Under the hood, the compiler lays out N contiguous `T` slots and exposes indexer + `Span<T>` conversion. This is the machinery `System.Numerics.Vector<T>`, `UInt128`, and `Utf8JsonReader`'s internal rings now use — before .NET 8 these had to be hand-rolled with unsafe fixed-size buffers.

### When to use inline arrays

- **Small, fixed-size buffers inside a struct** that need deterministic layout and no heap alloc.
- **Ring buffers, parsers, state machines** with bounded working sets.
- **Interop structs** mirroring a C struct with an embedded array (`struct foo { char name[32]; ... }`).

```csharp
[InlineArray(32)]
public struct Name32
{
    private byte _first;

    public static implicit operator Span<byte>(Name32 n) => /* runtime-provided */;
}

[StructLayout(LayoutKind.Sequential)]
public struct FooInterop
{
    public Name32 Name;     // matches C's char name[32]
    public int Id;
}
```

### Inline array gotchas

- The `[InlineArray(N)]` struct must have **exactly one instance field**. The compiler uses it as the "template element"; the struct's actual size becomes `N * sizeof(T) + padding`.
- You cannot access the backing field directly by name after the first slot — indexer access (`buf[i]`) is the only way. That's by design.
- The struct is **value-typed**. Passing by value copies all N elements. If `N` is large, pass by `in` / `ref` to avoid the copy.
- `Span<T>` / `ReadOnlySpan<T>` conversions are provided; `IEnumerable<T>` is not. Use `foreach` via the pattern, or slice to a span.

## Pointers (`T*`, `fixed`, unsafe)

Raw pointers are the bottom of the C# abstraction stack. They require `unsafe` context (project `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` plus `unsafe` keyword) and are **not verifiable** — the JIT trusts you.

### `fixed` — pinning a managed object

```csharp
byte[] arr = new byte[1024];
unsafe
{
    fixed (byte* p = arr)
    {
        // While the block is active, 'arr' is pinned in memory — GC won't move it.
        NativeWrite(p, arr.Length);
    }
    // After the block exits, the pin is released; the GC can move arr again.
}

// Also works on strings (returns a pointer to internal char buffer):
unsafe
{
    fixed (char* p = "hello") { ... }
}

// And on 'ref'-returnable things:
unsafe
{
    fixed (int* p = &list[0]) { ... }   // pin list's backing array
}
```

Rules:
- `fixed` pins the **entire object** (an array can't be half-pinned).
- While pinned, the object cannot participate in compaction, which fragments the heap. Keep `fixed` blocks short.
- For **long-lived** pinned buffers, use the Pinned Object Heap instead: `GC.AllocateArray<T>(len, pinned: true)`. Never needs a `fixed` statement; never fragments the SOH.
- Don't return a pointer obtained inside `fixed` — the pin ends when the block does, and the pointer becomes dangling.

### `stackalloc` pointers

In raw `unsafe` code, `stackalloc` returns `T*`:

```csharp
unsafe
{
    int* buf = stackalloc int[16];
    buf[0] = 42;
}
```

Prefer `Span<int> buf = stackalloc int[16]` in safe code. The pointer form is there for interop with C-style APIs that want `int*`.

### `NativeMemory` — the unmanaged heap

For buffers you want on the native heap (not managed at all, not on the stack):

```csharp
unsafe
{
    byte* p = (byte*)NativeMemory.Alloc((nuint)size);
    try { UseNative(p, size); }
    finally { NativeMemory.Free(p); }
}

// Zeroed variant
byte* z = (byte*)NativeMemory.AllocZeroed((nuint)size);

// Aligned for SIMD / DMA
byte* a = (byte*)NativeMemory.AlignedAlloc((nuint)size, alignment: 64);
// ...
NativeMemory.AlignedFree(a);
```

`NativeMemory` replaced `Marshal.AllocHGlobal` / `AllocCoTaskMem` for modern unmanaged allocation (.NET 6+). It's a thin wrapper over `malloc` / `free` with alignment guarantees. Use it for:
- Buffers handed to native code that must outlive any pin.
- Large allocations you don't want the GC to even know about.
- Memory arenas with custom lifetimes.

**You own it.** Leaking it leaks process memory; the GC does nothing. Wrap in `SafeHandle` if possible.

### Function pointers (`delegate*`, C# 9)

C# 9 introduced unmanaged / managed function pointers — faster than `Delegate` because there's no allocation and no indirection through an invocation list.

```csharp
// Unmanaged: calling a native function by pointer.
unsafe
{
    delegate* unmanaged[Cdecl]<int, int, int> add =
        (delegate* unmanaged[Cdecl]<int, int, int>)NativeLibrary.GetExport(h, "add");
    int r = add(1, 2);
}

// Managed: invoking a static method by pointer (no Delegate object).
unsafe
{
    delegate*<int, int, int> mul = &Math.Multiply;  // fictional static
    int r = mul(3, 4);
}
```

Production use cases: high-performance interop, dynamic dispatch in runtime hot loops where `Delegate.Invoke` overhead matters, and source-generated dispatch tables.

## Interop with `Span<T>` / pointers / `ref`

The bridges between the three worlds:

```csharp
// Span -> pointer (requires fixed / unsafe)
unsafe
{
    Span<byte> s = buf;
    fixed (byte* p = s) { Native(p, s.Length); }
}

// Pointer -> Span (unsafe)
unsafe
{
    byte* p = (byte*)NativeMemory.Alloc(256);
    Span<byte> s = new(p, 256);
}

// Ref -> Span of length 1 (safe, no alloc)
ref int r = ref someField;
Span<int> singleton = MemoryMarshal.CreateSpan(ref r, 1);

// Span reinterpret (safe if layout-compatible, e.g., bytes <-> Guid)
Span<byte> bytes = stackalloc byte[16];
Span<Guid> guids = MemoryMarshal.Cast<byte, Guid>(bytes);    // 1 guid
```

`MemoryMarshal` is the "escape hatch" namespace — low-risk, low-allocation conversions between span-like types. `Unsafe` (in `System.Runtime.CompilerServices`) is the lower-level version for `ref`/`void*` reinterpret.

## When to use each

- **`Span<T>` over a `stackalloc`** — your first choice for a small, bounded, method-local buffer. Safe, fast, zero heap cost.
- **Inline arrays** — when you need a fixed-size buffer **as a field** of a struct, or to match a C struct layout. Also useful for SIMD-aligned local data.
- **`ArrayPool<T>` / `Memory<T>`** — when you need to cross an `await` or store the buffer in a field.
- **`NativeMemory`** — when the lifetime is long, the size is large, and you're OK with manual management (or are handing off to native code).
- **`fixed` pointers** — a short-lived pin for an interop call. Anything longer, move to POH.
- **Function pointers** — a specific optimization for known-hot dispatch; not a general-purpose replacement for delegates.

**Senior-level gotchas:**
- `stackalloc` on an unbounded or untrusted size is a **reliability bug**, not just a perf bug. `StackOverflowException` tears down the process — no `catch` can save you. Always bound with a constant, always fall back to the heap above the threshold.
- The compiler may **hoist** `stackalloc` out of a loop — or, if it can't, emit a warning. Never rely on per-iteration stack allocation; it's not how the JIT thinks about it anyway.
- `[SkipLocalsInit]` can turn a "safe" method into a secret-leaking one if you read before writing. Use it only where you can prove write-before-read by inspection. On methods called from untrusted input, leave the default (zero-init).
- **Inline arrays are value types.** `Buffer8<int>` passed by value copies 32 bytes. For anything beyond ~16 bytes, pass `in` or `ref` — otherwise you lose the alloc win to copy cost.
- A `fixed` block pins the **whole** object. `fixed (byte* p = &arr[10])` does not pin only element 10 — it pins `arr` in its entirety. For very long-lived pinning, move to the POH (`GC.AllocateArray<T>(n, pinned: true)`).
- `Marshal.AllocHGlobal` is older and uses the Windows process heap (`LocalAlloc` variant). `NativeMemory.Alloc` uses the CRT allocator (`malloc`). They're not interchangeable — free with the same namespace you allocated from.
- Mixing `Span<T>` and `ref struct`s with generics is still limited pre-C# 13. `Task.Run<Span<byte>>` fundamentally cannot work — a `Span<T>` can't live on the heap, and a `Task<T>` does. The `allows ref struct` constraint (C# 13) is the long-term fix, but most APIs haven't adopted it yet.
- Don't allocate a managed buffer, pin it, and hand it to native code that retains the pointer past your `fixed` scope. That's a use-after-free across the managed/unmanaged boundary. For async I/O patterns, use `Memory<T>.Pin()` — which returns a `MemoryHandle` whose lifetime you control — or use `GC.AllocateArray<T>(n, pinned: true)` so the buffer never moves.
- `delegate*` function pointers skip all the safety checks of `Delegate.Invoke` — no multicast, no null-check (well, still a null deref crash — but no friendly `NullReferenceException`), no GC rooting of the target method if it's a dynamic method. Know what you're giving up.
- **Alignment matters for SIMD.** `NativeMemory.AlignedAlloc(..., 64)` for AVX-512 loads, or use inline arrays whose layout is naturally aligned. Misaligned SIMD access on old hardware can fault; on modern hardware it's just slow.
- Inline arrays' implicit conversion to `Span<T>` exposes the backing storage — any mutation through the span mutates the struct. If you return the struct by value, the caller gets a **copy**, and their span points to **their copy**, not yours. This is almost never what you want; return `ref` or expose a method that takes a `Span<T>` argument.
