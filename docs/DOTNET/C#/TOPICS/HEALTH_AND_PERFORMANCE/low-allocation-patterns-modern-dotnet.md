# Low-allocation patterns in modern .NET

_Targets .NET 10 / C# 14. See also: [Span&lt;T&gt;](./span-of-t.md), [Memory&lt;T&gt;](./memory-of-t.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [System.IO.Pipelines](./system-io-pipelines.md), [Channel&lt;T&gt;](./channel-of-t.md), [Heap allocation patterns](./heap-allocation-patterns.md), [Pinned objects](./pinned-objects.md), [`ref` returns and `ref struct`](../GENERAL_TOPICS/ref-return-ref-struct-record-struct.md)._

The "what to do instead" companion to [Heap allocation patterns](./heap-allocation-patterns.md). Modern .NET (Core 2.1 onward, accelerating in .NET 6–10) has assembled a coherent toolkit for writing throughput-critical code without allocations: stack buffers, pooled buffers, span-based parsing, frozen lookup tables, source generators, `ValueTask`, channels, pipelines. Used together they make zero-allocation hot paths a normal engineering target, not heroics.

## Stackalloc + Span: the cheapest buffer

`stackalloc` carves bytes out of the current stack frame; `Span<T>` gives you a safe API over them.

```csharp
Span<byte> hash = stackalloc byte[32];
SHA256.HashData(input, hash);
// hash is valid until the method returns
```

Rules:

- Sizing rule: cap stack buffers at ~1 KB. Anything larger risks `StackOverflowException` (especially under deep recursion or thread-pool threads with smaller default stacks).
- `Span<T>` is a `ref struct` — it cannot live on the heap, cannot be a class field, cannot be captured by a lambda, cannot survive an `await`. That's *why* it's safe with `stackalloc`.
- The conditional pattern for size-uncertain buffers:

```csharp
const int StackThreshold = 256;
byte[]? rented = null;
Span<byte> buf = size <= StackThreshold
    ? stackalloc byte[StackThreshold]
    : (rented = ArrayPool<byte>.Shared.Rent(size));

try
{
    var actual = buf[..size];
    DoWork(actual);
}
finally
{
    if (rented is not null) ArrayPool<byte>.Shared.Return(rented);
}
```

## ArrayPool&lt;T&gt;.Shared

The shared pool is process-wide, lock-free per-thread, bucketed by power-of-two sizes.

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(minimumLength: 4096);
try
{
    int read = await stream.ReadAsync(buf.AsMemory(0, 4096), ct);
    // ... use buf[..read] ...
}
finally
{
    ArrayPool<byte>.Shared.Return(buf, clearArray: false);
}
```

Rules:

- `Rent(n)` may return a buffer **larger** than `n`. Always slice (`buf.AsSpan(0, n)`).
- After `Return`, treat the buffer as if it were freed — never read it again, never keep a reference. Use it via `using` patterns or local scopes.
- `clearArray: true` zeroes the buffer on return; necessary only when the buffer carried secrets (keys, PII). It costs CPU.
- For longer or pinned-buffer scenarios, `MemoryPool<T>` returns `IMemoryOwner<T>` that disposes back to the pool — easier lifetime management around `Memory<T>`.
- `ArrayPool<T>.Create(...)` makes a private pool with custom buckets. Useful for size-class outliers but rarely worth the maintenance.

## string.Create and InterpolatedStringHandler

`string.Create` builds a string in one pass — exact size, no intermediates.

```csharp
public static string Trace(long id, int code) =>
    string.Create(20, (id, code), static (span, state) =>
    {
        span[0] = '#';
        state.id.TryFormat(span[1..], out var w1);
        span[1 + w1] = '/';
        state.code.TryFormat(span[(2 + w1)..], out _);
    });
```

`[InterpolatedStringHandler]` types let APIs receive an interpolated string **without** allocating the formatted `string` first:

```csharp
// ILogger has an interpolated handler — if the level is disabled, no string is built.
log.LogDebug($"received {request.Id} from {request.Tenant}");
```

`Debug.Assert`, `StringBuilder.Append`, `Console.WriteLine`, `Span.TryWrite`, and several `LoggerMessage` paths are all handler-aware. Random `void Foo(string s)` APIs are not — they get a regular formatted string.

## UTF-8 string literals

```csharp
ReadOnlySpan<byte> contentType = "application/json"u8;
ReadOnlySpan<byte> crlf = "\r\n"u8;
```

The `u8` suffix produces a `ReadOnlySpan<byte>` backed by data in the assembly's read-only segment — **never allocates**, ever. Perfect for protocol literals, headers, JSON keys, framing.

## Frozen / Immutable collections

```csharp
private static readonly FrozenDictionary<string, Country> ByIso =
    LoadAll().ToFrozenDictionary(c => c.IsoCode, StringComparer.OrdinalIgnoreCase);
```

`FrozenDictionary<K,V>` / `FrozenSet<T>` cost more to **build** than `Dictionary<,>`, but reads are faster (specialized hashers, no resize state, allocation-free probes). Right answer for static lookup tables loaded at startup.

`ImmutableDictionary` / `ImmutableArray` are different — they're for **persistent** data structures with structural sharing. Reads are slower than `Frozen`. Use Immutable when you need to "mutate" by producing a new copy that shares structure; use Frozen when you build once and read many.

## Source generators over reflection

Reflection-based machinery allocates: every reflective `MemberInfo`, every dynamic dispatch, every `Activator.CreateInstance`. Source generators replace this with code emitted at compile time:

| Generator | Replaces | Win |
|---|---|---|
| `System.Text.Json` source gen (`[JsonSerializable]`) | reflection-based `JsonSerializer` | no per-call delegate / property metadata allocs; AOT-safe |
| `Regex` source generator (`[GeneratedRegex(...)]`) | `new Regex(pattern)` (compiles at runtime, allocs cache) | code at compile time, no JIT, no caches |
| `LoggerMessage` source generator (`[LoggerMessage]`) | `ILogger.LogXxx($"...")` | zero alloc when level filtered, formatted span fast path otherwise |
| `ConfigurationBinder` source gen | reflective options binding | startup throughput, AOT-safe |
| `Microsoft.Extensions.Options.SourceGeneration` | reflective validation | no reflection at validate time |

These generators only help if you actually use the generated entry point — `[JsonSerializable]` types, `[GeneratedRegex]` partials, `[LoggerMessage]` partial methods. Mixing reflection-based and source-gen calls makes both slower.

## ValueTask and pooled async

```csharp
public ValueTask<int> ReadAsync(byte[] buf)
{
    if (_buffered > 0) return new ValueTask<int>(DrainSync(buf));   // sync path: no Task alloc
    return new ValueTask<int>(ReadCoreAsync(buf));                  // async path: alloc only here
}
```

`ValueTask<T>` is a struct that can wrap either a sync result or a `Task<T>`. The contract:

- **Consume once.** Never `await` it twice, never store and re-await. If you need to, call `.AsTask()` and pay the allocation.
- Don't return `ValueTask` from async methods that almost always actually await something — the wrapping cost pays off only on sync completions.
- For the hottest async methods, `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]` pools the state machine box. Costs measurable cycles for thread coordination — measure both ways.

## Channels and Pipelines

`Channel<T>` is the producer-consumer primitive. Bounded channels give you backpressure for free:

```csharp
var ch = Channel.CreateBounded<Work>(new BoundedChannelOptions(1024)
{
    FullMode = BoundedChannelFullMode.Wait     // producer waits when full
});
```

`System.IO.Pipelines` is the parser primitive. It owns the buffer pool, exposes data as `ReadOnlySequence<byte>` (a list of `ReadOnlyMemory<byte>` segments), and lets parsers `AdvanceTo(consumed, examined)` without allocating staging buffers. Kestrel, gRPC, RabbitMQ, and SignalR all parse via Pipelines.

```csharp
async Task ParseAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        var result = await reader.ReadAsync(ct);
        var buffer = result.Buffer;

        while (TryReadFrame(ref buffer, out var frame)) Handle(frame);

        reader.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) break;
    }
}
```

## Ref struct enumerators and params spans

C# 13's `params ReadOnlySpan<T>` parameter type lets you write variadic APIs without per-call array allocations:

```csharp
public static int Sum(params ReadOnlySpan<int> nums)
{
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

Sum(1, 2, 3);                    // no array alloc — compiler stack-buffers the span
```

`MemoryExtensions.Split` returns a `SpanSplitEnumerator<T>` (a `ref struct`) — `foreach` over it parses CSV/TSV/log lines without allocating an array of strings.

## Pinned Object Heap for long-lived buffers

For buffers handed to native code or kernel APIs that must not move, allocate directly to the POH:

```csharp
private static readonly byte[] Scratch = GC.AllocateArray<byte>(64 * 1024, pinned: true);
```

The buffer is on the POH, never compacted, immune to fragmentation. Detail in [Pinned objects](./pinned-objects.md).

## Verification

Always confirm the change with measurement, not intuition:

```csharp
[MemoryDiagnoser]
public class FormatBenches
{
    [Benchmark(Baseline = true)] public string Plain() => $"id={42}";
    [Benchmark]                   public string Created() => string.Create(5, 42, static (s, n) =>
    {
        "id=".AsSpan().CopyTo(s); n.TryFormat(s[3..], out _);
    });
}
```

The `Allocated` column is the primary number. In production, watch `dotnet-counters monitor System.Runtime` — `alloc-rate` (B/s) is the load-bearing metric; `gen-0-gc-count`, `gen-1-gc-count`, `gen-2-gc-count` are the consequences.

**Senior-level gotchas:**

- `Span<T>` and `stackalloc` cannot escape the method. The compiler enforces this — but it means you can't pass them to async methods or store them in fields. `Memory<T>` can cross awaits, at the cost of an extra indirection (no direct access; you go through `.Span` to use it).
- `ArrayPool` returns a buffer **at least** as large as requested — sometimes much larger (next bucket up). Always slice, and never assume `buf.Length == requestedSize`.
- Pool buffers are old. Once they survive the first GC after rental, they're in Gen2 forever. Don't measure pool wins with short BenchmarkDotNet runs — the pool's promotion cost is amortized.
- `string.Create` lets you allocate fewer strings, but you trade safety for speed — `Span<char>` mutation in the callback is your responsibility. The compiler will not catch a write past `Length`.
- Source generators help **only** if your call site invokes the generated entrypoint. `JsonSerializer.Serialize<MyType>(obj)` uses the generator; `JsonSerializer.Serialize(obj, typeof(MyType))` does not. Same trap with `Regex` (use the generated partial method, not `new Regex`).
- `ValueTask` is a footgun if mis-used: double-awaiting throws `InvalidOperationException` on debug builds, and on release silently corrupts the underlying `IValueTaskSource`. If you can't be sure, return `Task<T>`.
- `FrozenDictionary` build cost is significant — do it once at startup (a static initializer or `IHostedService`). Don't rebuild it per request.
- `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]` requires the method to **actually be hot enough** to amortize pool ops. For async methods called once per request, the pool overhead exceeds the alloc save.
- `Pipelines` looks similar to `Stream` but the contract is different: `AdvanceTo(consumed, examined)` tells the pipe what you parsed vs. what you peeked. Getting this wrong either replays bytes forever or drops them.
- `params ReadOnlySpan<T>` is **C# 13** — check `<LangVersion>`. Older code using `params T[]` allocates per call site.
- Don't pursue zero allocations everywhere. Cold paths (config load, error handling, startup) don't need pooling. The goal is "no allocations in the request hot path"; a `string.Format` in `Configure(...)` is not a problem.
