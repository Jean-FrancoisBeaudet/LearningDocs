# ArrayPool&lt;T&gt;

_Targets .NET 10 / C# 14. See also: [Span&lt;T&gt;](./span-of-t.md), [Memory&lt;T&gt;](./memory-of-t.md), [Buffer management](./buffer-management.md), [Large Object Heap (LOH)](./large-object-heap-loh.md), [GC pressure](./gc-pressure.md), [Heap allocation patterns](./heap-allocation-patterns.md)._

`ArrayPool<T>` from `System.Buffers` is the standard answer to "I'm allocating a `T[]` on every request and the GC is suffering." Rent → use → return. Two reasons it pays off:

1. **Avoids LOH allocations** for buffers ≥ ~85,000 bytes (the LOH threshold for any reference-element array; element-size-dependent for arrays of structs).
2. **Avoids Gen-0 churn** for shorter-lived buffers on hot paths.

It's not a silver bullet — get the lifetime contract wrong and the pool turns into a leak.

## The shape

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(minLength: 4096);
try
{
    // buf.Length >= minLength; treat the rest as garbage and slice down.
    int n = await stream.ReadAsync(buf.AsMemory(0, requested), ct);
    Process(buf.AsSpan(0, n));
}
finally
{
    ArrayPool<byte>.Shared.Return(buf, clearArray: false);
}
```

Three things this snippet gets right:

- `Rent(n)` returns an array **at least** `n` long, often larger (next bucket up). Always slice to your real size.
- The `try/finally` returns the buffer even on exception. A leaked rental is silent — the pool just allocates a replacement next time, and your "no-allocation" claim is a lie.
- `clearArray: false` is correct here for a buffer that held only `byte`s and will be overwritten. For other element types or sensitive data, see below.

## `Shared` vs `Create`

```csharp
ArrayPool<byte> shared = ArrayPool<byte>.Shared;                       // process-wide
ArrayPool<byte> custom = ArrayPool<byte>.Create(
    maxArrayLength: 16 * 1024 * 1024,    // up to 16 MiB
    maxArraysPerBucket: 32);
```

- **`Shared`** uses a tuned, internal implementation: a thread-local rack of small arrays plus a global rack, capped at ~1 MiB per array. Anything bigger is allocated and never pooled — it goes straight to the LOH. Cross-allocator contention is rare in practice because of the thread-local rack.
- **`Create(...)`** gives you a fixed-strategy `ConfigurableArrayPool`. Use it when:
  - You need to pool genuinely large buffers (>1 MiB) where Shared would just allocate.
  - You're building a sub-system whose pool churn shouldn't pollute the global Shared pool's heuristics.
  - You want bucketed sizes you control.

For 99% of code, **use `Shared`**. Don't create a pool per service.

## The `clearArray` parameter

```csharp
ArrayPool<T>.Shared.Return(array, clearArray: ?);
```

Choose `true` when:

- `T` is or contains a **reference type** — otherwise the pooled array roots stale objects and prevents collection. (For `T` *with* references, .NET ≥ 6 clears anyway as a safety net, but be explicit.)
- The buffer held **secrets, credentials, PII, tokens** — a pooled buffer survives across unrelated requests.

Choose `false` when:

- `T` is a primitive (`byte`, `int`, `double`, `char`) you'll fully overwrite next time, with no security concerns. Skipping the zero-fill is meaningfully faster on large buffers.

If you're unsure: clear it.

## Pool buckets and the MaxArrayLength cliff

`ArrayPool<T>.Shared` partitions arrays into power-of-two buckets up to `2^20` (~1 MiB). A `Rent(2_000_000)` is **not** pooled — it's `new byte[2_000_000]`, and `Return` of it is a no-op. The pool just hands you whatever array exists in the next bucket up; if none, it allocates one.

For genuinely large transient buffers you want pooled, build your own `ArrayPool<T>.Create(maxArrayLength: yourCeiling, maxArraysPerBucket: ...)` and dedicate it to that workload.

## `ArrayPool<T>` vs `MemoryPool<T>`

```csharp
// ArrayPool<T>: returns the array directly. Cleanup is your job.
byte[] buf = ArrayPool<byte>.Shared.Rent(size);
try { ... } finally { ArrayPool<byte>.Shared.Return(buf); }

// MemoryPool<T>: returns IMemoryOwner<T>. Cleanup is `using`.
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(size);
Memory<byte> buf = owner.Memory;
```

`MemoryPool<byte>.Shared` is implemented over `ArrayPool<byte>.Shared`. Pick `MemoryPool` when ownership flows through async APIs (`return owner;` from an `async` method is clean; `try/finally` over an `await` is awkward). Pick `ArrayPool` when the rental is local and you want the lower-overhead direct surface.

## Common anti-patterns

| Anti-pattern | Why it bites |
|---|---|
| Rent without `try/finally` | An exception leaks the array; the pool grows; your service runs slightly slower forever |
| `Return` then keep using the array | Two consumers, one buffer, racing writes — silent corruption |
| `Return` the same array twice | Pool may hand it out twice — same outcome |
| `Return(buf)` where `buf` came from `new[]`, not the pool | The pool happily takes it; mixing pooled and non-pooled lifetimes makes lifetime bugs untraceable |
| Hold a slice of a rented array as a class field | Slice points into a buffer that may be reissued — use-after-free with no exception |
| Rent on every iteration of a tight inner loop | Defeats the pool's thread-local fast path through churn — pre-rent once, reuse, return at the end |
| Pool elements that are themselves expensive to clear | `clearArray: true` on a 16 MiB array of structs has real cost — measure |

## Visibility for diagnostics

`ArrayPool<T>` ships ETW events on the `System.Buffers.ArrayPoolEventSource` provider:

- `BufferRented`, `BufferAllocated` (pool missed, allocated new), `BufferReturned`, `BufferTrimmed`.

Capture with `dotnet-trace` or PerfView when investigating pool effectiveness. A high `BufferAllocated`-to-`BufferRented` ratio means your buckets are mis-sized for your workload — switch to a custom `ArrayPool<T>.Create`.

## Senior-level gotchas

- **`Rent(n)` may return an array larger than `n`** — always slice (`.AsSpan(0, n)` / `.AsMemory(0, n)`) before passing to consumers. Code that uses `buf.Length` as the "how much data is here" sentinel is broken from day one.
- **Returned arrays are not zeroed by default** — your sensitive bytes leak into the next renter who'll see them in their `Rent` result. For credentials, set `clearArray: true`. For arrays of reference types, modern .NET clears for safety regardless, but don't rely on that across runtimes — be explicit.
- **The Shared pool is per-process, not per-AppDomain or per-`AssemblyLoadContext`**. Plugin scenarios that load the same assembly twice still share the pool. Usually fine; rarely surprising.
- **Long rentals are slow rentals**: `ArrayPool` is most efficient when arrays come back to the same thread quickly. Holding a rented array across a long `await` (waiting on an external service, say) means the pool's thread-local rack is empty when you need it next, forcing the slower global path.
- **Pool trimming**: under memory pressure, the runtime trims the pool — recently-rented-back arrays may be released to the GC. After a Gen-2 you can see a temporary uptick in `BufferAllocated`. This is correct; don't try to defeat it.
- **Cross-thread rent/return is fine but not free**: the global rack uses a lock-free stack with CAS. High-contention churn (rent on Thread A, return on Thread B, every microsecond) measurably loses to a single-thread pattern. Design for "rent and return on the same logical work item."
- **`ArrayPool<byte>.Shared` and `MemoryPool<byte>.Shared` are not the same instance**, but `MemoryPool` delegates to `ArrayPool` for `byte`. For other element types, `MemoryPool<T>.Shared` is a thin wrapper that allocates an `IMemoryOwner<T>` per rental — that wrapper itself is a small heap allocation.
- **Don't pool arrays of mutable structs containing references** unless you're going to clear them. The struct's reference fields keep their referents alive — for arrays this is functionally a memory leak.
- **Don't write your own `ArrayPool<T>`** unless you have a benchmark proving the BCL one is wrong for your workload. Most "I built my own pool" code is slower and buggier than `ArrayPool<T>.Create`.
