# Serialization overhead

_Targets .NET 10 / C# 14. See also: [Connection pooling](./connection-pooling.md), [Inter-service call optimization](./inter-service-call-optimization.md), [Low-allocation patterns in modern .NET](./low-allocation-patterns-modern-dotnet.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [System.IO.Pipelines](./system-io-pipelines.md)._

In a healthy distributed system, serialization is the dominant cost on every inter-service hop after network. The "obvious" answer — pick a fast serializer — is half the story. The other half is DTO shape, payload size, allocation profile, and whether the serializer can run under NativeAOT. A 1 KB JSON object in `Newtonsoft.Json` allocates ~30 objects on serialize and ~50 on deserialize; the same object in `System.Text.Json` source-gen allocates ~3, in MessagePack ~1, in MemoryPack often zero. Multiplied by RPS, that's the difference between "GC dominates the trace" and "GC doesn't show up".

## What you're actually paying for

Three independent costs, often conflated:

1. **Wire size** — bytes the network has to ship. JSON is verbose, protobuf is compact (≈30–50 % of equivalent JSON), MessagePack sits between.
2. **CPU** — bytes/sec the serializer can produce or consume. Reflection-based serializers re-walk the type each call; source-gen and IL-emit pay the cost once.
3. **Allocation** — heap pressure per call. The cost not visible in BenchmarkDotNet `Mean` but very visible in `% time in gc` under load.

Optimizing one without measuring the others is the typical trap. A binary serializer that's 5× faster but allocates 10× more might tank a 1000 RPS service while looking great in micro-bench.

## The serializer landscape (modern .NET)

| Serializer | Wire format | Speed | Allocation | AOT-safe | Use case |
|---|---|---|---|---|---|
| `System.Text.Json` (reflection) | JSON | Good | Moderate | No (reflection) | Public APIs, config, dev/test |
| `System.Text.Json` source-gen (`JsonSerializerContext`) | JSON | Very good | Low | Yes | Public APIs at scale, AOT |
| `Newtonsoft.Json` | JSON | OK | High | No | Legacy, complex `JObject` editing, `JsonConverter` ecosystem |
| `MessagePack-CSharp` (`[MessagePackObject]`) | MessagePack | Excellent | Very low | Yes (with `mpc.exe` codegen) | Internal RPC, caches, queues |
| `MessagePack` typeless | MessagePack | Very good | Moderate | Limited | Polymorphic payloads |
| `Google.Protobuf` (gRPC default) | protobuf | Excellent | Low | Yes | gRPC, schema-driven contracts |
| `protobuf-net` | protobuf | Excellent | Low | Partial | gRPC alternative, decorated POCO |
| `MemoryPack` (.NET 7+) | MemoryPack (custom binary) | Best-in-class | Often zero | Yes | Hot internal paths, blittable types |
| `Bond`, `FlatBuffers`, `Cap'n Proto` | various | High | Very low | Mixed | Niche, schema-evolution-heavy |

Default in 2026: **System.Text.Json source-gen for public APIs, MessagePack or MemoryPack for internal hot paths, gRPC + protobuf when contracts must travel**.

## System.Text.Json with source generators

The reflection path of `System.Text.Json` builds and caches metadata on first use of each type — fine for steady-state, costly on startup, **broken under NativeAOT** (reflection metadata is trimmed). The source generator emits the metadata at compile time, allowing zero-reflection serialization and AOT-clean publishing.

```csharp
[JsonSourceGenerationOptions(WriteIndented = false, PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
[JsonSerializable(typeof(ApiError))]
internal partial class AppJsonContext : JsonSerializerContext;
```

Use it via the `JsonTypeInfo<T>` overloads:

```csharp
string json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
Order order = JsonSerializer.Deserialize(json, AppJsonContext.Default.Order)!;
```

ASP.NET Core wires this up:

```csharp
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

Wins: ~50 % CPU reduction over reflection, half the allocations, AOT-publishable, build-time errors on missing converters. The cost is one partial class per project that owns the contract types — almost always worth it.

## Cached `JsonSerializerOptions`

A subtle but high-impact trap on the reflection path:

```csharp
// Bad — new options each call. Reflection metadata cache is per-options instance,
// so this rebuilds the cache every call before .NET 6.
JsonSerializer.Serialize(value, new JsonSerializerOptions { WriteIndented = true });

// Good — single cached instance shared across calls.
private static readonly JsonSerializerOptions JsonOpts = new(JsonSerializerDefaults.Web)
{
    WriteIndented = false,
};
JsonSerializer.Serialize(value, JsonOpts);
```

.NET 6+ caches metadata more aggressively even with new options instances, but a shared options object remains the canonical pattern. Source-gen sidesteps the issue entirely.

## Newtonsoft.Json — when to keep it

Don't migrate just for sport. Keep `Newtonsoft.Json` when:

- Your contracts depend on `JObject` / `JToken` editing not yet matched by `JsonNode`.
- You have rich `JsonConverter` infrastructure that would take weeks to port.
- Polymorphic JSON with `$type` discriminators (System.Text.Json supports this since .NET 7 via `[JsonDerivedType]`, but the migration path is non-trivial).

Migrate when:

- Allocation profile shows up as a top-3 GC contributor in PerfView.
- You're publishing under NativeAOT (`Newtonsoft.Json` is reflection-heavy).
- Public APIs touch hot endpoints where the per-request cost matters.

## Binary formats: MessagePack, protobuf, MemoryPack

### MessagePack-CSharp

```csharp
[MessagePackObject]
public class Order
{
    [Key(0)] public long Id { get; init; }
    [Key(1)] public string Customer { get; init; } = "";
    [Key(2)] public long TotalCents { get; init; }
}

byte[] bytes = MessagePackSerializer.Serialize(order);
Order back = MessagePackSerializer.Deserialize<Order>(bytes);
```

`[Key(int)]` with integer indices produces the smallest payload (no field names on the wire). String keys (`[Key("name")]`) trade size for forward-compat. The `IL2CPP` / NativeAOT story uses the source-generator-based attribute (`[GeneratedMessagePackResolver]`) — same payload, no IL emit at runtime.

### Google.Protobuf (gRPC default)

Schema-first via `.proto` files. The C# code is generated from the schema, ensuring the contract is the schema, not the C# types:

```protobuf
syntax = "proto3";
message Order {
  int64 id = 1;
  string customer = 2;
  int64 total_cents = 3;
}
```

gRPC over HTTP/2 + protobuf gives 3–10× higher throughput than JSON-over-HTTP/1.1 on the same hardware — the difference compounds from binary encoding, header compression (HPACK), connection multiplexing, and the absence of JSON whitespace.

### MemoryPack (.NET 7+)

Custom binary format optimized for .NET. Source-generated, AOT-safe, often **zero-allocation** on blittable types (structs of primitives copy by `MemoryMarshal.Cast`).

```csharp
[MemoryPackable]
public partial record Order(long Id, string Customer, long TotalCents);

byte[] bytes = MemoryPackSerializer.Serialize(order);
Order back = MemoryPackSerializer.Deserialize<Order>(bytes)!;
```

Trade-off: format is .NET-specific (no Java/Go/JS clients out of the box). Use only when both ends are .NET.

## DTO design

Serializer choice is half the story; the other half is what you put on the wire. The same logical object can vary 5–10× in payload size based on naming and shape.

- **Use `record` with `init`** — immutable contracts serialize cleanly across all major libraries and document themselves.
- **Nullable reference types** — let the contract say what's required vs optional rather than relying on `string.Empty` sentinels.
- **Avoid `object`-typed properties** unless you genuinely need polymorphism. Polymorphic JSON requires a type discriminator and converter setup; protobuf requires `oneof` or `Any`. Both are sources of subtle bugs.
- **Flatten where it pays** — a 5-deep nested DTO produces 5 nested JSON objects with their own braces. Flatten to a single record when the nesting carries no semantic value.
- **String IDs vs integer IDs** — a UUID string is 36 bytes on the wire; the binary form is 16. For inter-service hops, pass binary; for public APIs, pass strings.
- **Don't ship internal entity types** — coupling the wire format to your DB schema makes both refactor-hostile. Have a DTO layer.

```csharp
// Good — record, init, nullable, no polymorphism
public sealed record OrderResponse(long Id, string Customer, long TotalCents, DateTimeOffset PlacedAt);

// Smell — entity type leaking, EF navigations on the wire
public class Order
{
    public int Id { get; set; }
    public string Customer { get; set; }
    public ICollection<OrderItem> Items { get; set; }   // serializes the navigation
    public Customer CustomerNav { get; set; }           // recursive shape
}
```

## Streaming serialization

For payloads that don't need to be in memory at once — bulk export, log shipping, gRPC server streaming:

```csharp
// JSON streaming — bounded memory regardless of payload size.
await using var stream = response.Body;
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Order>(
                  stream, AppJsonContext.Default.Order, ct))
{
    await Process(item, ct);
}
```

For maximum throughput, drop to `Utf8JsonReader`/`Utf8JsonWriter` over `Span<byte>` or `IBufferWriter<byte>` (System.IO.Pipelines). These are the building blocks `System.Text.Json` uses internally — direct use eliminates the per-call options/context resolution and lets you control buffer rental from `ArrayPool<byte>.Shared`.

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(8192);
try
{
    var writer = new Utf8JsonWriter(buffer.AsMemory());
    writer.WriteStartObject();
    writer.WriteString("customer"u8, order.Customer);
    writer.WriteNumber("totalCents"u8, order.TotalCents);
    writer.WriteEndObject();
    await stream.WriteAsync(buffer.AsMemory(0, (int)writer.BytesPending), ct);
}
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

`u8` literal suffix (C# 11) creates a `ReadOnlySpan<byte>` at compile time — no UTF-8 conversion at runtime.

## Compression

Two distinct decisions:

- **Wire compression (gzip/brotli/zstd)** at the HTTP layer trades CPU for bandwidth. Pays off on JSON (high redundancy), often *negative* on already-compressed payloads (PNG, JPEG, video). ASP.NET Core's `ResponseCompression` middleware gates by content type — verify the type list excludes pre-compressed assets.
- **In-payload compression** (gzipping a JSON field's value) is almost always wrong — opaque to logging, breaks streaming, multiplies decoding cost.

For internal RPC, prefer protobuf/MessagePack over gzip-on-JSON: the binary format already captures the redundancy, with no per-message CPU.

## Tooling

| Tool | Use |
|---|---|
| BenchmarkDotNet `[MemoryDiagnoser]` | Per-format Mean, Allocated — comparison harness |
| `dotnet-counters monitor System.Runtime` | `alloc-rate`, `% time in gc` deltas before/after migration |
| PerfView "GC Heap Alloc Stacks" | Top allocating call sites by serializer |
| gRPC built-in tracing (`Grpc.Net.Client.GrpcChannelOptions.LoggerFactory`) | Per-call serialization time |
| Wireshark / `tcpdump` + `tshark` | Actual wire size and round-trip count |
| `dotnet trace collect` with Microsoft-System-Net-Http source | HTTP/2 framing details |

## Senior-level gotchas

- **`JsonSerializerOptions` is expensive to create — and locks itself once first used.** Mutating an options instance after first serialize/deserialize call throws. Build it once, share it, treat it as immutable.
- **`[JsonStringEnumConverter]` is slower than int enum serialization** but produces stable-across-codegen contracts. Fine for public APIs where the enum value names matter; costly for internal hot paths where you control both ends.
- **`DateTime` round-trips are serializer-dependent.** STJ defaults to ISO-8601 round-trip-O (preserves kind and offset for `DateTimeOffset`), MessagePack stores ticks, protobuf has `google.protobuf.Timestamp` (seconds + nanos, UTC). Cross-language services should use `DateTimeOffset` and explicit UTC.
- **Protobuf has no `null` for value types** — `int32` defaults to `0`, indistinguishable from "not set". Use `optional int32` (proto3 since 3.15) or wrapper types (`google.protobuf.Int32Value`) when "absent" is meaningful.
- **gRPC has a 4 MB max message size by default.** Bulk responses silently fail with `RESOURCE_EXHAUSTED`. Either bump the limit (`MaxReceiveMessageSize` / `MaxSendMessageSize`) or — preferred — switch to server streaming, which has no per-message size cap.
- **Server streaming over gRPC is dramatically cheaper than batching into one message.** A million-row export as one `repeated` field allocates the entire collection on both ends; as a stream of individual messages, each message is small and bounded.
- **`PublishAot` breaks reflection-based serializers.** `Newtonsoft.Json` and the reflection path of `System.Text.Json` strip metadata and fail at runtime with cryptic "type info not available" errors. Switch to source-gen contexts before publishing AOT, and run AOT in CI to catch regressions.
- **Property naming policies cost CPU and allocation per write.** Source-gen bakes the wire name in. Reflection mode runs the policy (camelCase, snake_case) per property per call.
- **`JsonSerializer.Serialize<object>(value)` boxes value types and uses runtime polymorphism** — slow path. Always pass the static type or the source-gen `JsonTypeInfo<T>`.
- **Caching a deserialized payload is often more useful than caching the serialized one.** When the same response is served to many requests, hold the parsed DTO in a singleton; serialize on demand. Inverse direction: incoming payloads parsed once, dispatched many times — the parse cost dominates.
- **MemoryPack and MessagePack contracts evolve via key/index.** Removing or renumbering a key is a breaking wire change; adding a new optional one is fine. Document the contract evolution rules per type — version bumps are silent failures otherwise.
- **`ReadOnlySpan<byte>` UTF-8 literals (`"name"u8`)** are the only way to compare JSON property names without allocating; `Utf8JsonReader.ValueTextEquals` consumes a `ReadOnlySpan<byte>`. Avoid `reader.GetString() == "name"` on hot paths.
- **`HttpClient` deserializing JSON allocates a `Stream` reader per response.** `HttpClient.GetFromJsonAsync<T>(uri, AppJsonContext.Default.MyType, ct)` (with the source-gen overload) is the lowest-allocation shape — fewer wrapping streams, no `JsonSerializerOptions` per-call lookup.
- **Schema evolution forgiveness varies sharply.** protobuf is famously forgiving (unknown fields preserved through round-trips); MessagePack with integer keys breaks if you renumber; STJ ignores unknown JSON fields by default but `[JsonRequired]` (.NET 7+) catches missing ones at parse time.
