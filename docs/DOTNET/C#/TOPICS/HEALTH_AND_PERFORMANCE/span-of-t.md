# Span&lt;T&gt;

_Targets .NET 10 / C# 14. See also: [Memory&lt;T&gt;](./memory-of-t.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [Buffer management](./buffer-management.md), [Allocations and boxing](./allocations-and-boxing.md), [Pinned objects](./pinned-objects.md)._

`Span<T>` is a `ref struct` — a (`ref` to memory, length) pair — that lets you treat any contiguous block of `T` as a single typed view: a `T[]`, a `stackalloc` buffer, a slice of a `string`, native memory from `Marshal.AllocHGlobal`, or memory rented from a pool. The whole point is **typed access without copying** and, where possible, **without allocating**.

## What's in the struct

```text
ref struct Span<T> {
    ref T _reference;   // managed pointer (interior or otherwise)
    int   _length;
}
```

That `ref T` is what makes it a `ref struct`: the field carries a managed reference that the GC must update if the underlying object moves, and the runtime can only do that for things that live on the stack. Everything that follows from `Span<T>` flows from this constraint.

## Stack-only — and what that costs

Because the struct holds a managed reference to potentially interior storage, the runtime forbids it from ever sitting on the heap. So:

- **Cannot be a field on a class** (or any non-`ref` struct).
- **Cannot cross an `await` or `yield return`** — the state-machine box would put it on the heap.
- **Cannot be captured by a lambda** (closure → heap).
- **Cannot be a generic type argument** unless the receiving generic is annotated `where T : allows ref struct` (C# 13).
- **Cannot be boxed**, which also rules out `object` casts and most reflection paths.

If any of those constraints bite, the answer is `Memory<T>` — heap-friendly, with `.Span` to materialise a `Span<T>` when you actually need to touch the bytes.

## Construction shapes

```csharp
// From an array (no copy).
int[] arr = [1, 2, 3, 4, 5];
Span<int> s1 = arr;                       // implicit
Span<int> s2 = arr.AsSpan(1, 3);          // {2,3,4}

// From stackalloc (lifetime = current method frame).
Span<byte> s3 = stackalloc byte[256];

// Read-only over a string (no allocation, no copy).
ReadOnlySpan<char> s4 = "hello".AsSpan();

// From any IBufferWriter<T>.GetSpan / Memory<T>.Span / pool rental.
Span<byte> s5 = owner.Memory.Span;
```

Slicing is **always** O(1) and never allocates — `Slice` returns a new `Span<T>` over the same memory.

## `stackalloc` with a guard

`stackalloc` is the most powerful Span source — it gets you a buffer with **zero** heap traffic — but it's also the easiest way to overflow the stack. Always cap it:

```csharp
const int StackThreshold = 256;
Span<byte> buffer = length <= StackThreshold
    ? stackalloc byte[StackThreshold]
    : new byte[length];

buffer = buffer[..length];   // trim to the actual size
```

The conditional must produce both branches as the same span type — the compiler picks the wider lifetime. For larger budgets, rent from `ArrayPool<byte>.Shared` instead.

## Span-friendly BCL APIs

Modern .NET BCL is span-first. A short tour:

```csharp
// Parsing without strings.
int n = int.Parse("12345"u8.ToArray().AsSpan(), provider: null);  // bytes
int m = int.Parse("12345".AsSpan(), provider: null);              // chars
Utf8Parser.TryParse("12345"u8, out int v, out _);

// Searching, splitting (.NET 8+).
ReadOnlySpan<char> input = "a,b,,c";
foreach (Range r in input.Split(','))
    Console.WriteLine(input[r]);

// Copying without LINQ allocation.
Span<int> dst = stackalloc int[8];
src.AsSpan().CopyTo(dst);

// Formatting into a span.
int written;
((double)Math.PI).TryFormat(dst: stackalloc char[32], out written, "F4");
```

Anything you used to do with `string.Substring` + `int.Parse` + `string.Split` has a span equivalent that doesn't allocate.

## Reinterpreting and bridging

```csharp
// Treat a Span<int> as Span<byte> (little-endian by definition of the platform).
Span<int> ints = stackalloc int[4];
Span<byte> bytes = MemoryMarshal.AsBytes(ints);

// View a List<T>'s backing array as a span (don't mutate the list while the span is live).
List<int> list = [1, 2, 3];
Span<int> view = CollectionsMarshal.AsSpan(list);

// Bridge to native code.
unsafe
{
    fixed (byte* p = buf) NativeApi(p, buf.Length);
}
```

`MemoryMarshal.Cast<TFrom, TTo>` is the same idea over arbitrary unmanaged element types — useful for SIMD work and for parsing binary protocols without copying.

## When to reach for `Span<T>` and when to refuse

| Scenario | Use `Span<T>`? |
|---|---|
| Parsing/formatting in a hot loop | Yes |
| Slicing a buffer or string for one-frame work | Yes |
| Field on a class to hold buffers between method calls | No → `Memory<T>` |
| Returning bytes from an async method | No → `Memory<T>` / `IMemoryOwner<T>` |
| Capturing in a callback or lambda | No → restructure or `Memory<T>` |
| Library API surface intended for general consumption | Mostly accept `ReadOnlySpan<T>` and `Span<T>` for input; return `Memory<T>` if the caller may want to flow it |

The first rule of `ref struct`: it infects the call stack. Any method that takes a `Span<T>` parameter must itself be callable from a frame where the constraint holds. Don't push the type into a service layer that will be tempted to store it.

## `ReadOnlySpan<T>` vs `Span<T>`

Use `ReadOnlySpan<T>` for inputs by default. The implicit conversion `Span<T> → ReadOnlySpan<T>` is free, the reverse is not allowed. `ReadOnlySpan<char>` over a `string` is the standard way to accept "any kind of text" — it doesn't force callers to allocate `char[]`.

## Senior-level gotchas

- **Stack overflow from unbounded `stackalloc`**: never `stackalloc length` where `length` is from user input. Cap it with a constant and fall back to `ArrayPool` / `new[]`.
- **The indexer returns by `ref`**: `span[0] = 5` mutates the underlying buffer. That's the feature — no boxing, no copy — but it means a `Span<T>` over a `T[]` field is a write port to that field. `ReadOnlySpan<T>` returns by `readonly ref` and rejects assignment.
- **`ReadOnlySpan<char>` over a `string` is safe** because the runtime treats interior pointers into strings specially during compaction, but **DO NOT** mutate via `MemoryMarshal.GetReference` — strings are immutable and you'll corrupt the intern table.
- **`Span<byte>` over a `stackalloc` cannot survive a method return** even if you store it in a `ref struct` wrapper — the C# 11 / 12 ref-safety rules make this a compile error (CS8350 / CS8352). Don't fight it; lift to `ArrayPool`.
- **`IndexOf`, `Contains`, `SequenceEqual`, `Replace`** on `Span<byte>` and `Span<char>` are SIMD-accelerated on modern x64/ARM64. A naive `for` loop is almost always slower.
- **`ref struct` propagation infects callers** — adding a `Span` parameter to a virtual method on an interface implementation will force every override into a `ref struct`-aware signature. Plan the API boundary deliberately.
- **`Span<T>` does not own anything**. If the underlying array came from `ArrayPool.Rent`, the span keeps no record of that — returning the array invalidates every span over it with no exception, no warning. Keep ownership and views in the same scope.
- **Equality is by reference & length**, not content. `s1 == s2` is not allowed; use `s1.SequenceEqual(s2)` for value comparison.
- **Crossing an `await` is a compile error (CS4007)**, including holding the variable across a `using` block whose `Dispose` is async. Refactor the local lifetime; do not suppress with `#pragma`.
- **In a `ref struct` method, `this` itself is a `ref struct`** — careful with extension methods and instance methods on `Span<T>` that try to capture state.
