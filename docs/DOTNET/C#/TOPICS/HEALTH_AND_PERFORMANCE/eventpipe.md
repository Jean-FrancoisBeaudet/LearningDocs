# EventPipe

_Targets .NET 10 / C# 14. See also: [ETW](./etw-event-tracing-for-windows.md), [dotnet-trace](./dotnet-trace.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [PerfView](./profiling-perfview.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [GC pressure](./gc-pressure.md), [BenchmarkDotNet](./benchmarkdotnet.md)._

EventPipe is the CLR's **in-process, cross-platform tracing transport**. The runtime emits events to an in-process bus; an out-of-process tool (or in-proc consumer) attaches over IPC, opens a session against a list of providers, and receives a stream of structured events serialised in the **NetTrace / Bedrock** binary format. Every cross-platform .NET diagnostic — `dotnet-counters`, `dotnet-trace`, `dotnet-monitor`, `Microsoft.Diagnostics.NETCore.Client` — sits on top of it.

EventPipe is **not** ETW. It can't see kernel events (file I/O, context switches, hardware counters, DPC/ISR). It can't trace a process it isn't attached to (no system-wide capture). What it gives up in breadth it pays back in portability and zero-admin operation: it works the same on Linux, macOS, Windows, and inside containers, with no privilege requirement and no kernel driver.

## Where it fits

| Strength | Weakness |
|---|---|
| Cross-platform; identical surface on Linux/macOS/Windows | CLR-only — no kernel, no native non-CLR providers |
| No admin / capability required; in-process | One session per process; no system-wide aggregation |
| Bounded overhead; bounded buffers; ring or stream | Cannot collect from a process that already crashed |
| Same provider/keyword model as ETW; portable knowledge | Buffer pressure can drop events under load if the consumer is slow |
| `.nettrace` opens in PerfView, Speedscope, Visual Studio | No GUI — all tooling is CLI or library-driven |

Reach for EventPipe when:

1. The host is **not Windows** — there is no other in-runtime trace bus on Linux/macOS.
2. You need to attach to a **running, locked-down container** without admin or kernel modules.
3. You want to ship a custom `EventSource` and have the same listener stack work in dev, CI, and production.
4. You need a programmatic in-proc consumer (a sidecar, a memory dump trigger, a custom counter exporter).

## Architecture

```
┌──────────── target .NET process ─────────────┐
│                                              │
│  EventSource ──┐                             │
│  Meter ────────┼─► EventPipe in-proc bus ──► │  IPC ───► dotnet-trace / dotnet-counters /
│  CLR providers ┘                             │           Microsoft.Diagnostics.NETCore.Client
│                                              │
└──────────────────────────────────────────────┘
```

- **Providers** publish events. CLR built-ins (`Microsoft-Windows-DotNETRuntime`, `Microsoft-DotNETCore-SampleProfiler`, `System.Runtime`), framework (`System.Net.Http`, `Microsoft.AspNetCore.Hosting`), and your own `EventSource` / `Meter` instances are all providers.
- **Sessions** are the subscription primitive. A consumer opens a session by sending a list of `(Provider, Keywords, Level, Arguments)` triples over IPC. The runtime starts buffering matching events.
- **IPC transport** is platform-specific:
  - **Linux / macOS** — Unix domain socket at `$TMPDIR/dotnet-diagnostic-<pid>-<disambiguator>-socket` (typically `/tmp/`).
  - **Windows** — named pipe `\\.\pipe\dotnet-diagnostic-<pid>`.
- **Wire format** is the **NetTrace** stream (Bedrock-based). Tools either consume it live or persist it as `.nettrace`. PerfView, Visual Studio, and Speedscope (after `dotnet-trace convert`) all read it.

## Built-in CLR providers

| Provider | Common keywords | Purpose |
|---|---|---|
| `Microsoft-Windows-DotNETRuntime` | `0x1` GC, `0x10` JIT, `0x20` NGen/R2R, `0x4000` Contention, `0x8000` Exception, `0x10000` ThreadPool, `0x200000` GCSampledObjectAllocationHigh | Core CLR runtime events (same set as ETW) |
| `Microsoft-Windows-DotNETRuntimeRundown` | n/a | Method-info "rundown" emitted on session stop so a trace is decodable after the fact |
| `Microsoft-DotNETCore-SampleProfiler` | n/a | CPU sampling at ~1 ms intervals — what `dotnet-trace --profile cpu-sampling` enables |
| `Microsoft-Diagnostics-DiagnosticSource` | filter via `FilterAndPayloadSpecs` | Forwards `DiagnosticSource` and `Activity` events |
| `System.Runtime` | n/a | The "well-known counters" `dotnet-counters` shows by default |

The keyword bitmask is identical to ETW — knowledge transfers directly. A typical CPU-and-GC trace is `0x1F00001 = 0x1 | 0x10 | 0x20 | 0x4000 | 0x10000 | 0x200000 | …`. The full table lives in [`profiling-perfview.md`](./profiling-perfview.md).

## Custom EventSource

```csharp
using System.Diagnostics.Tracing;

[EventSource(Name = "Acme-Orders")]
public sealed class OrdersEventSource : EventSource
{
    public static readonly OrdersEventSource Log = new();

    [Event(1, Level = EventLevel.Informational, Keywords = Keywords.Pricing)]
    public void PriceComputed(string orderId, double cents) =>
        WriteEvent(1, orderId, cents);

    [Event(2, Level = EventLevel.Warning, Keywords = Keywords.Pricing)]
    public void PriceFallback(string orderId, string reason) =>
        WriteEvent(2, orderId, reason);

    public static class Keywords
    {
        public const EventKeywords Pricing = (EventKeywords)0x1;
        public const EventKeywords Inventory = (EventKeywords)0x2;
    }
}
```

A consumer subscribes to it like any other provider:

```bash
dotnet-trace collect --process-id 12345 \
  --providers 'Acme-Orders:0x1:Informational' \
  --duration 00:00:30
```

`EventSource` is the public API; `Meter` (.NET 6+) is the modern replacement for *counters*, but `EventSource` remains the right choice for **structured, filterable, schema'd events**. See [`dotnet-counters.md`](./dotnet-counters.md) for the `Meter` side.

## Programmatic collection (`Microsoft.Diagnostics.NETCore.Client`)

For sidecars, custom dump triggers, or in-test instrumentation, drive EventPipe from code instead of a CLI:

```csharp
using Microsoft.Diagnostics.NETCore.Client;
using Microsoft.Diagnostics.Tracing;
using Microsoft.Diagnostics.Tracing.EventPipe;

var client = new DiagnosticsClient(processId: 12345);

var providers = new[]
{
    new EventPipeProvider(
        "Microsoft-Windows-DotNETRuntime",
        EventLevel.Informational,
        keywords: 0x1F00001),
    new EventPipeProvider(
        "Acme-Orders",
        EventLevel.Verbose,
        keywords: 0x1)
};

using EventPipeSession session = client.StartEventPipeSession(providers, requestRundown: true);
using var source = new EventPipeEventSource(session.EventStream);

source.Dynamic.All += e =>
{
    if (e.ProviderName == "Acme-Orders" && e.EventName == "PriceFallback")
        Console.WriteLine($"{e.TimeStamp:O} {e.PayloadByName("reason")}");
};

source.Process(); // blocks; call session.Stop() from another thread to end.
```

`requestRundown: true` is critical for traces that will be analysed off-process — the runtime emits method/symbol info on session stop so PerfView can resolve frames.

## Diagnostic ports and startup tracing

By default the runtime listens on a *server-mode* diagnostic port — tools attach after the process is up. To capture events from **before `Main` returns** (cold-start, startup latency, first-request profiling), invert the relationship: the tool listens, the runtime dials in.

```bash
# Tool side: listen on a socket and only collect once the runtime connects.
dotnet-trace collect \
  --diagnostic-port /tmp/orders.sock \
  --providers Microsoft-Windows-DotNETRuntime:0x1F00001:5

# In another shell — start the app pointed at that port.
DOTNET_DiagnosticPorts=/tmp/orders.sock,suspend dotnet ./Orders.dll
```

`suspend` makes the runtime block in `Main` until the tool acknowledges the session — guarantees no events are missed. Drop the `,suspend` for "connect when ready, miss the very early events" semantics.

`DOTNET_DiagnosticPorts` is the only EventPipe knob the user usually sets. Multiple ports can be listed `;`-separated. In a container, this is how you push diagnostics onto the **outside** of the container without exec'ing in.

## Containers and Kubernetes

EventPipe needs the diagnostic socket to be reachable from the consumer. Three patterns:

1. **`kubectl exec` in-pod** — install `dotnet-trace` as a sidecar tool, attach by PID. Simple but requires the tool image and a writable filesystem.
2. **Shared `emptyDir` volume** — both app container and a `dotnet-monitor` sidecar mount the same `/tmp`; the sidecar finds the socket file. The mainstream production pattern.
3. **`DOTNET_DiagnosticPorts` to a sidecar** — the runtime dials out to the sidecar's listening socket. Best when the app container is read-only or distroless.

`dotnet-monitor` (the official sidecar product, not to be confused with `dotnet-counters monitor`) is the supported way to expose EventPipe over HTTP/gRPC for clustered scenarios — it speaks EventPipe IPC inside, OTel/Prometheus/REST outside.

## EventPipe vs ETW (decision matrix)

| Question | EventPipe | ETW |
|---|---|---|
| Cross-platform? | Yes | No (Windows only) |
| Needs admin? | No | Yes for kernel providers, sometimes for CLR |
| Kernel events (file I/O, context switch)? | No | Yes |
| Hardware counters (CPU pmu)? | No | Yes (`xperf -on PROC_THREAD+LOADER+PMC_PROFILE`) |
| System-wide capture (multi-process)? | No | Yes |
| In-proc consumer possible? | Yes (`EventPipeEventSource`) | Yes (`TraceEventSession` real-time) |
| .NET Framework support? | No | Yes |
| Output format | `.nettrace` | `.etl` |
| Default tool | `dotnet-trace` | PerfView / `logman` |

On Windows, EventPipe and ETW emit overlapping CLR events; you can use either, and the choice usually reduces to "do I need kernel events?" If yes, ETW. If no, EventPipe — same data, less ceremony.

## Senior-level gotchas

- **EventPipe sessions are per-process.** There is no system-wide listener. To monitor 50 pods you run 50 sessions (or use `dotnet-monitor` sidecars). Don't confuse this with ETW's machine-wide `xperf`.
- **Buffer pressure drops events silently.** Default circular buffer is 256 MB; under a high-event provider (`GCSampledObjectAllocationHigh`, verbose `DiagnosticSource`) the consumer can lag and lose events. The runtime exposes a `RundownRequested` event but no in-band "events lost" signal — measure event counts against expectations.
- **`requestRundown: true` is essential for off-process analysis.** Without it the trace lacks method-name metadata and PerfView shows raw IL token IDs. Always enable for `.nettrace` files saved to disk.
- **The diagnostic socket lives in `$TMPDIR`, not always `/tmp`.** Some container images (distroless, read-only-root) override `TMPDIR`. If `dotnet-trace ps` shows nothing, check `/proc/<pid>/environ` for `TMPDIR=` and bind-mount accordingly.
- **`DOTNET_DiagnosticPorts=...,suspend` blocks the process until a consumer connects.** If you bake this into a deployment manifest and the consumer never shows up, the pod hangs in `Main` and looks like a hang bug. Use `suspend` only when the tool is also part of the deployment.
- **EventPipe runs in-process.** A misbehaving provider (yours or a third-party `EventSource`) that allocates per event will inflate the very GC numbers you're trying to measure. Keep custom event payloads small (primitives + strings) and avoid hot-path emission > ~100k/s.
- **`Microsoft-Diagnostics-DiagnosticSource` does not forward by default.** You must pass a `FilterAndPayloadSpecs` argument naming the activities you want — otherwise the provider is enabled but emits nothing. The format is `[ActivityName1]/[Property1];[ActivityName2]/[Property2]`.
- **EventPipe disables itself on platforms it can't open the socket on.** Some seccomp/AppArmor profiles block `AF_UNIX` socket creation in `/tmp`; the runtime falls back to no-op silently. Test the diagnostic port works as part of image hardening, not in incident response.
- **`.nettrace` is a Bedrock binary stream, not a self-describing format.** You need either PerfView, Speedscope (via `dotnet-trace convert`), or `EventPipeEventSource` to read it. Don't try to grep it.
- **A custom `EventSource` is matched by *name*, not assembly.** Two libraries shipping `[EventSource(Name = "Foo")]` collide silently — only one wins per process. Namespace your providers (`Acme-Orders`, not `Orders`).
- **`EventLevel.Verbose` (5) on `Microsoft-Windows-DotNETRuntime` is firehose territory.** `0x1F00001` at level 5 captures every JIT compile, every contention, every GC tick. Use level 4 (`Informational`) for routine production captures and reserve verbose for short, targeted windows.
- **`dotnet-counters` and `dotnet-trace` cannot share a session.** Two concurrent sessions are allowed (default cap = 4) but they don't merge — running both at once doubles overhead. Pick one tool per investigation.
- **EventPipe overhead is roughly proportional to event volume, not provider count.** Enabling 10 quiet providers is cheaper than enabling one chatty provider. Tune by **keyword bitmask**, not by adding/removing providers wholesale.
