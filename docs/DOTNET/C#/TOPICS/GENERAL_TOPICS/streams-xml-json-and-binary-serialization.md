# Streams, XML, JSON, and binary serialization

_Targets .NET 10 / C# 14. See also: [Span / Memory](../MEMORY_MANAGEMENT/span-memory.md), [ref return, ref struct, record struct](./ref-return-ref-struct-record-struct.md), [using and lock](../CONSTRUCTS/using-and-lock.md), [Async streams & `await foreach`](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md)._

Serialization is about moving **in-memory object graphs** to and from **byte streams**. This note covers the `Stream` abstraction, the three serialization families you'll actually pick from today (JSON, XML, binary), and the modern source-generated paths that replace reflection-driven APIs.

## `Stream` — the base abstraction

`System.IO.Stream` is an abstract byte sink / source. Every serializer, every network protocol, every `File.OpenRead` flows through it. Capabilities it declares but doesn't always implement:

| Property               | Meaning                                    |
|------------------------|--------------------------------------------|
| `CanRead`/`CanWrite`   | Bytes flow in one or both directions.      |
| `CanSeek`              | `Position`/`Length` meaningful, can `Seek`.|
| `CanTimeout`           | `ReadTimeout`/`WriteTimeout` honored.      |

### Reading/writing — prefer async, prefer `Memory<byte>`

```csharp
await using var file = File.OpenRead(path);                          // FileStream
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
try
{
    int read;
    while ((read = await file.ReadAsync(buffer.AsMemory(), ct)) > 0)
        Process(buffer.AsSpan(0, read));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

Notes:
- **`ReadAsync(Memory<byte>, CancellationToken)`** is the modern overload. The legacy `ReadAsync(byte[], int, int, ct)` allocates an `Overlapped` on older TFMs.
- **`await using`** because many streams override `DisposeAsync` to flush buffered writes asynchronously.
- **`ArrayPool<byte>.Shared`** to avoid `new byte[8192]` per call.
- **Don't assume a full read.** `Stream.ReadAsync` is allowed to return fewer bytes than requested — only `Stream.ReadExactlyAsync` guarantees the full count (and throws `EndOfStreamException` if it can't).

### `Stream.CopyToAsync`

The fastest way to pump stream-to-stream:

```csharp
await source.CopyToAsync(dest, bufferSize: 81920, ct);
```

Defaults already use pooled buffers. Large files through the network pipe: let it do the work.

### `MemoryStream` footguns

- `new MemoryStream(byte[] buf)` is **non-resizable**; `new MemoryStream()` grows. Know which constructor you took.
- `.ToArray()` **copies**; `.GetBuffer()` returns the internal buffer (size ≥ `Length`) — use `.AsSpan(0, (int)stream.Length)`.
- A disposed `MemoryStream` invalidates `GetBuffer()`. `RecyclableMemoryStream` (Microsoft.IO) is the production-grade alternative when churn matters.

### Pipelines — beyond streams

For zero-copy, back-pressured I/O, use **`System.IO.Pipelines`** (`PipeReader`/`PipeWriter`). ASP.NET Core Kestrel and SignalR are built on it. The mental model: the reader examines buffered data via `ReadOnlySequence<byte>` and advances only what it consumed; the writer requests a `Memory<byte>` slot and commits.

```csharp
async Task ParseAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult res = await reader.ReadAsync(ct);
        foreach (var frame in TryReadFrames(res.Buffer, out var consumed))
            Handle(frame);
        reader.AdvanceTo(consumed, res.Buffer.End);
        if (res.IsCompleted) break;
    }
    await reader.CompleteAsync();
}
```

Use streams for convenience, pipelines for throughput.

## JSON — `System.Text.Json` first, always

`System.Text.Json` (aka STJ) is the in-box serializer since .NET 3.0; performance-competitive, AOT-friendly, `Utf8` end-to-end.

### Reflection-based (simple)

```csharp
var opts = new JsonSerializerOptions(JsonSerializerDefaults.Web)
{
    WriteIndented = false,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
};

// CACHE the options — allocating per-call kills throughput.
string json = JsonSerializer.Serialize(user, opts);
User? u    = JsonSerializer.Deserialize<User>(json, opts);
```

The one rule: **cache `JsonSerializerOptions`.** Every new instance spins up internal metadata caches from scratch.

### Source-generated (AOT / hot path)

```csharp
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
internal partial class AppJsonContext : JsonSerializerContext { }

string json = JsonSerializer.Serialize(user, AppJsonContext.Default.User);
User?  u    = JsonSerializer.Deserialize(json, AppJsonContext.Default.User);
```

- No reflection at runtime, so trimming and NativeAOT work.
- Faster startup (no metadata probing).
- Two modes: `Metadata` (serializer still drives, generator supplies type info) and `Serialization` (generator emits full `Write`/`Read` code — fastest, narrower feature support). Default is `Metadata`.

### `JsonNode` / `JsonDocument`

- `JsonDocument` — immutable, **pooled** — use when you parse once and read a few values. **`Dispose()` it** to return the rented memory.
- `JsonNode` — mutable DOM (`JsonObject`, `JsonArray`, `JsonValue`). Allocates more; use when you're editing JSON shape.

### Custom converters

For polymorphism, discriminator fields, or oddly-shaped inputs:

```csharp
public sealed class UriConverter : JsonConverter<Uri>
{
    public override Uri Read(ref Utf8JsonReader reader, Type t, JsonSerializerOptions opts)
        => new(reader.GetString()!);
    public override void Write(Utf8JsonWriter writer, Uri v, JsonSerializerOptions opts)
        => writer.WriteStringValue(v.ToString());
}
```

Polymorphic serialization is now first-class via `[JsonDerivedType(typeof(Cat), "cat")]` on the base type — a discriminator is emitted/read automatically.

### When Newtonsoft.Json still wins

- Very dynamic shapes where `JObject`/`JArray` ergonomics matter (`$.some.deep.path`).
- Legacy behavior parity with pre-STJ services (e.g., case-insensitive deserialization defaults, `JsonConvert.PopulateObject`).
- Specific converters (e.g., `StringEnumConverter` with flags naming) not yet matched.

Otherwise, STJ is the answer.

## XML

Three APIs, three purposes:

| API                       | Shape                           | When                                             |
|---------------------------|---------------------------------|--------------------------------------------------|
| `XmlSerializer`           | Contract-based (attributes on types), reflection | SOAP interop, WSDL-generated contracts.      |
| `XDocument` / `XElement`  | LINQ to XML — ergonomic DOM     | Reading/shaping XML by hand, configuration files. |
| `DataContractSerializer`  | `[DataContract]`/`[DataMember]` | WCF heritage. Avoid in greenfield.              |
| `XmlReader` / `XmlWriter` | Forward-only, push/pull         | Huge documents where a DOM won't fit.           |

```csharp
// LINQ to XML — the one most people should use:
var doc = XDocument.Load(path);
var names = doc.Root!.Elements("user")
                .Where(e => (int)e.Attribute("age")! > 18)
                .Select(e => (string)e.Element("name")!)
                .ToList();
```

Don't feed untrusted XML to any of these with DTD processing enabled — always `new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, XmlResolver = null }` for XXE safety.

## Binary serialization

### `BinaryFormatter` is gone. Do not use it.

`BinaryFormatter` is **removed** from .NET 9+ and obsoleted since .NET 5. It deserializes arbitrary types — a textbook RCE surface. If a legacy system hands you `BinaryFormatter` payloads, sandbox the decode or replace the format.

### Modern choices

| Format / library              | Strengths                                                 | When                                           |
|-------------------------------|----------------------------------------------------------|------------------------------------------------|
| **MemoryPack**                | Zero-encode, source-generated, .NET-only, very fast       | Internal .NET-to-.NET IPC.                     |
| **MessagePack-CSharp**        | Compact binary, cross-language, schemaless                | Polyglot services, Redis payloads, MQ bodies.  |
| **Protocol Buffers (protobuf-net or Grpc.Tools)** | Schema-first, versioned, cross-language | Public contracts, gRPC.                        |
| **Cap'n Proto, FlatBuffers**  | Mmap-friendly, zero-copy reads                           | Game / HPC scenarios.                          |

Example with MemoryPack:

```csharp
[MemoryPackable]
public partial record class User(int Id, string Name);

byte[] bin = MemoryPackSerializer.Serialize(user);
var u      = MemoryPackSerializer.Deserialize<User>(bin);
```

### Rolling your own

For tiny framed binary protocols (headers, length prefixes), skip any library and use `BinaryPrimitives` + `Span<byte>`:

```csharp
Span<byte> header = stackalloc byte[8];
BinaryPrimitives.WriteInt32LittleEndian(header, msgId);
BinaryPrimitives.WriteInt32LittleEndian(header[4..], payloadLen);
await stream.WriteAsync(header.ToArray(), ct);   // stackalloc + async = small copy; consider pooled Memory instead
```

For hashing/CRC: `System.IO.Hashing` (XxHash64, Crc32, Crc64). They take `ReadOnlySpan<byte>` and are source-generated under the hood.

## Senior-level gotchas

- **Not every `Stream.Read` fills the buffer.** Forgetting this is the #1 cause of "random protocol corruption." Use `ReadExactlyAsync` or loop.
- **`using var ms = new MemoryStream(); await x.WriteAsync(ms, ct); var bytes = ms.ToArray();`** allocates twice — once in the stream, once in `ToArray`. Prefer `GetBuffer().AsSpan(0, (int)ms.Length)` or an `ArrayBufferWriter<byte>`.
- **`JsonSerializerOptions` allocation per call** costs more than the serialization itself on small payloads. Hold a `static readonly` instance.
- **STJ is case-sensitive by default**; Newtonsoft is case-insensitive by default. This bites every migration. Set `PropertyNameCaseInsensitive = true` or use `JsonSerializerDefaults.Web`.
- **`[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]`** for value-type props — `WhenWritingNull` doesn't fire on `int? = null` if the property type is `int`. Know which variant you mean.
- **`JsonDocument`/`JsonElement` lifetime.** `JsonElement`s borrow into the document's pooled buffer — using them after `Dispose()` is a use-after-free. Materialize to `string`/`int` before the `using` block ends.
- **`XmlSerializer`'s first-use cost**: it synthesizes and JITs serializer assemblies on first use of a type. Production services pre-warm or use the `sgen` pre-generated serializer assemblies.
- **XXE** (XML External Entity) — default settings on `XmlReaderSettings` are not always safe in older TFMs. Always set `DtdProcessing = Prohibit` and `XmlResolver = null` for untrusted input.
- **BinaryFormatter payloads from disk/network are remote code execution.** Don't "just turn it back on" via `AppContext.SetSwitch` for convenience — migrate the format.
- **Pipelines look like streams but aren't.** Calling `ReadAsync` and forgetting to `AdvanceTo` deadlocks the pipe. The reader owes the writer a notification; no advance, no progress.
- **Source-generated JSON/Regex are the default in AOT.** Reflection-based calls throw at runtime under `PublishAot`. Catch this in CI with an AOT smoke test, not in production.
