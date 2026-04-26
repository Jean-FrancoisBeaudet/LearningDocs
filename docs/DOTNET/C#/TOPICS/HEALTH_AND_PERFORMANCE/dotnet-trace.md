# dotnet-trace

_Targets .NET 10 / C# 14, `dotnet-trace` 9.x. See also: [EventPipe](./eventpipe.md), [ETW](./etw-event-tracing-for-windows.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [PerfView](./profiling-perfview.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [GC pressure](./gc-pressure.md), [async/await performance](./async-await-performance.md), [BenchmarkDotNet](./benchmarkdotnet.md)._

`dotnet-trace` is the **cross-platform CLI for capturing managed traces** over [EventPipe](./eventpipe.md). It produces `.nettrace` files that open in PerfView, Visual Studio, or (after conversion) Speedscope. On Linux/macOS it is the *only* in-runtime trace tool. On Windows it does the same job as ETW for CLR-only investigations, with no admin and no `.etl`-file pain.

It is **not** a profiler with a UI, **not** a load tool, and **not** a replacement for ETW when kernel events are needed. Its job is exactly: enumerate processes, attach to one, enable a list of providers, write events to disk, detach.

## Where it fits

| Strength | Weakness |
|---|---|
| Cross-platform; works in any container with a writable `/tmp` | No GUI — analyse in PerfView/Speedscope/VS afterwards |
| No admin; attaches via diagnostic IPC | Per-process — no system-wide capture |
| Standard provider/keyword vocabulary; portable knowledge from ETW | Cannot capture kernel events (use [ETW](./etw-event-tracing-for-windows.md) on Windows) |
| Built-in `--profile` shortcuts for common scenarios | Default buffer is **not** circular — long captures fill disk |
| Speedscope / Chromium trace export for sharing | Symbol resolution depends on the trace having rundown events |

Reach for `dotnet-trace` when:

1. You want a CPU sample / GC trace on Linux or in a container.
2. The investigation is purely managed (CPU stacks, GC events, exceptions, contention, custom `EventSource`).
3. You need to share the trace with someone who has PerfView or Speedscope.
4. The host doesn't allow admin/elevation and `xperf`/PerfView aren't options.

## Install and inventory

```bash
# Install once.
dotnet tool install -g dotnet-trace

# List all .NET processes the current user can attach to.
dotnet-trace ps
#   12345  Orders.Api          /app/Orders.Api
#   12350  Pricing.Worker      /app/Pricing.Worker

# List the well-known providers and their typical keyword bitmasks.
dotnet-trace list-profiles
```

`dotnet-trace ps` enumerates by walking `$TMPDIR/dotnet-diagnostic-*` socket files (Linux/macOS) or named pipes (Windows). If a process is missing, the diagnostic IPC isn't reachable — usually a `TMPDIR` mismatch or a hardened seccomp profile.

## Built-in profiles

```bash
# CPU sampling at ~1 ms (default), 30 s.
dotnet-trace collect -p 12345 --duration 00:00:30
# Equivalent to: --profile cpu-sampling

# GC verbose — every allocation tick, every pause, induced GCs.
dotnet-trace collect -p 12345 --profile gc-verbose --duration 00:00:30

# GC summary only — small file, prod-safe.
dotnet-trace collect -p 12345 --profile gc-collect --duration 00:01:00

# Database — System.Data.* / Microsoft.EntityFrameworkCore.* events.
dotnet-trace collect -p 12345 --profile database --duration 00:01:00
```

| Profile | Providers enabled | Use for |
|---|---|---|
| `cpu-sampling` | SampleProfiler + ThreadTime + CLR/Loader/Jit/Stop | CPU hot path, "where is time going?" |
| `gc-verbose` | DotNETRuntime keywords `0x1F00001` at level Verbose | GC pause analysis, allocation attribution |
| `gc-collect` | DotNETRuntime keyword `0x1` at level Informational | Lightweight GC summary; safe baseline |
| `database` | `Microsoft-Diagnostics-DiagnosticSource` filtered to ADO.NET / EF Core activities | Slow-query / N+1 / connection-pool issues |

`--profile` is shorthand for a known provider tuple. Mix with `--providers` to add to it (the union is enabled).

## Custom providers

Provider syntax is `<Name>:<KeywordsHex>:<Level>[:<Args>]` — identical to PerfView/EventPipe. Multiple providers are comma-separated.

```bash
# CPU sampling + custom EventSource at verbose, 60 s.
dotnet-trace collect -p 12345 \
  --profile cpu-sampling \
  --providers 'Acme-Orders:0x1:5,Acme-Pricing:0xFF:4' \
  --duration 00:01:00

# Granular CLR — allocations + JIT + ThreadPool, no profile.
dotnet-trace collect -p 12345 \
  --providers 'Microsoft-Windows-DotNETRuntime:0x1F00001:5' \
  --duration 00:00:30

# DiagnosticSource → ETW bridge: name the activities and properties to forward.
dotnet-trace collect -p 12345 \
  --providers 'Microsoft-Diagnostics-DiagnosticSource:0x4:5:FilterAndPayloadSpecs="HttpHandlerDiagnosticListener/System.Net.Http.HttpRequestOut.Start@Activity1Start:-Request.RequestUri"' \
  --duration 00:01:00
```

`Level` accepts numeric (1–5) or names (`Critical`, `Error`, `Warning`, `Informational`, `Verbose`). The `Args` slot after the level is provider-specific — for `DiagnosticSource` it's `FilterAndPayloadSpecs` (which activities to forward and which payload properties to include).

The full keyword table is in [`profiling-perfview.md`](./profiling-perfview.md); use the same bitmask in `dotnet-trace`.

## CLR-events shorthand

```bash
dotnet-trace collect -p 12345 \
  --clrevents gc+gchandle+jit+contention+exception \
  --clreventlevel verbose \
  --duration 00:00:30
```

`--clrevents` accepts `+`-separated mnemonics: `gc`, `gchandle`, `loader`, `jit`, `ngen`, `startenumeration`, `endenumeration`, `security`, `appdomainresourcemanagement`, `jittracing`, `interop`, `contention`, `exception`, `threading`. Easier than computing keyword bitmasks for common cases.

## Output formats and conversion

```bash
# Default: .nettrace (Bedrock binary; PerfView, VS, dotnet-trace).
dotnet-trace collect -p 12345 --output trace.nettrace

# Speedscope JSON (https://www.speedscope.app — drag and drop).
dotnet-trace collect -p 12345 --format Speedscope --output trace.speedscope.json

# Chromium trace JSON (chrome://tracing or https://ui.perfetto.dev).
dotnet-trace convert trace.nettrace --format Chromium --output trace.chromium.json
```

PerfView / Visual Studio / `dotnet-trace`'s own `convert` are the canonical readers. **Speedscope is the easiest way to share a CPU trace with a colleague who doesn't have PerfView** — single HTML page, no install.

## Startup tracing

The default mode attaches to a running process — events from before attach are lost. To capture from `Main`, invert the connection: the tool listens, the runtime dials in.

```bash
# Tool listens on a socket (or pipe).
dotnet-trace collect \
  --diagnostic-port /tmp/orders.sock,suspend \
  --providers 'Microsoft-Windows-DotNETRuntime:0x1F00001:5' \
  --output startup.nettrace

# Runtime connects on startup and blocks until the tool acknowledges.
DOTNET_DiagnosticPorts=/tmp/orders.sock,suspend dotnet ./Orders.dll
```

`,suspend` makes the runtime block in `Main` until the tool is ready — guarantees zero missed events. Drop it for "best effort, don't hang on tooling absence". See [`eventpipe.md`](./eventpipe.md) for the diagnostic-port model in depth.

## Sizing and overhead

```bash
# Buffer size in MB (default 256). Larger = fewer dropped events under load.
dotnet-trace collect -p 12345 --buffersize 1024 --duration 00:01:00

# Stop after the trace file reaches a size (cap output file).
dotnet-trace collect -p 12345 --max-size-mb 500 --output trace.nettrace
```

Defaults are **not circular** — `dotnet-trace` accumulates and writes a single growing file. A cpu-sampling trace at default buffer can be safely run for a minute or two on a busy production process. A `gc-verbose` trace on a high-allocation workload generates 100+ MB/min; bound it with `--max-size-mb` or `--duration`.

## Production capture flow

A representative flow for "p99 latency spike at 14:30 — capture the next occurrence":

```bash
# 1. List candidate PIDs in the pod.
dotnet-trace ps

# 2. Capture 60 s starting now, with CPU + GC + contention.
dotnet-trace collect -p 12345 \
  --profile cpu-sampling \
  --providers 'Microsoft-Windows-DotNETRuntime:0x14000:4' \
  --duration 00:01:00 \
  --output /trace/spike.nettrace

# 3. Copy out for analysis.
kubectl cp orders-7df-abc12:/trace/spike.nettrace ./spike.nettrace

# 4. Open in PerfView (Windows) or convert for Speedscope.
dotnet-trace convert spike.nettrace --format Speedscope
```

For repeated triage, store this as a sidecar script. For automated capture-on-trigger, use **`dotnet-monitor`** — it watches counters and starts/stops `dotnet-trace`-equivalent sessions on rules.

## dotnet-trace vs PerfView vs ETW

| Question | dotnet-trace | PerfView | ETW |
|---|---|---|---|
| Cross-platform? | Yes | Read-only on Linux | No |
| Captures kernel events? | No | Yes (Windows) | Yes |
| Needs admin? | No | Yes for kernel | Yes for kernel |
| GUI? | No | Yes (Windows) | Through PerfView/WPA |
| File format | `.nettrace` | `.etl` (writes), `.nettrace` (reads) | `.etl` |
| CLR coverage | Full | Full | Full |
| Best for | Linux, containers, CLR-only | Production Windows GC analysis | Kernel + managed correlation |

Functional summary: `.nettrace` collected on Linux opens in PerfView on Windows. Same data; different transport.

## Senior-level gotchas

- **Default capture is not circular.** A 1-hour `dotnet-trace collect` on a busy process can fill the disk. Always pass `--duration` *or* `--max-size-mb`. There is no `/CircularMB` equivalent — `dotnet-trace` records-then-stops, it doesn't ring-buffer.
- **`dotnet-trace ps` shows no processes when `TMPDIR` differs.** In hardened/distroless containers, `TMPDIR` may be `/var/tmp` or `/run/dotnet`. Check `/proc/<pid>/environ` and either set `TMPDIR=...` for the tool or pass `--diagnostic-port <socket>` directly.
- **The trace must include a method/JIT rundown to be analysable.** For attach-mode captures, the runtime emits rundown automatically when the session stops — *if the process is still running*. If the target crashes mid-capture, the trace lacks rundown and PerfView shows IL token offsets. There is no recovery; recapture.
- **Don't enable `GCSampledObjectAllocationHigh` (`0x200000`) on a high-throughput service without thinking.** Sample rate is one stack per ~100 KB allocated, but in-process the sample emit itself allocates. On a 5 GB/sec allocator you generate 50 K stack samples/sec — buffer overflow and dropped events. Prefer `gc-collect` first to assess allocation rate, then narrow.
- **`--profile cpu-sampling` is per-CPU sampling, not per-thread.** A process with 100 mostly-blocked threads on a 4-core box gets ~4000 samples/sec total — adequate. A burning-hot 4-thread process on the same box also gets ~4000 samples/sec — adequate. But a single thread saturating one core on a 64-core box still gets ~1000 samples/sec for that thread, which is fine; just remember the unit is core-seconds, not thread-seconds.
- **The runtime drops events when the consumer is slow.** `dotnet-trace` writes to disk synchronously; on slow disks (network FS, EBS gp2) the in-process buffer fills and events drop. Symptom is suspicious gaps in the timeline. Mitigations: bigger `--buffersize`, faster output disk (local SSD, tmpfs), or shorter capture windows.
- **`--providers` parsing is fragile.** Quoting matters in shells. The `FilterAndPayloadSpecs` argument requires `=` and `;`, both of which need escaping in bash — use single quotes for the entire `--providers` value. Confirm with `dotnet-trace collect ... --providers "..." --print-config-only` (newer versions) before a real run.
- **Multiple concurrent sessions on the same process work but cost CPU.** The runtime supports up to 4 concurrent EventPipe sessions; each one has its own buffer and serialiser. Running `dotnet-counters` and `dotnet-trace` simultaneously is supported but doubles ingestion overhead. Prefer one tool per investigation.
- **`--clrevents jit` adds significant overhead during cold start.** Every method JIT emits two events. On a startup-tracing run this is exactly what you want; on steady-state it produces volume without value (most methods are already JITted). Drop `jit` from steady-state captures.
- **Speedscope is great for CPU traces, useless for GC traces.** `dotnet-trace convert --format Speedscope` only converts CPU sampling profiles. GC events are dropped silently from the conversion. For GC analysis, send the `.nettrace` to PerfView.
- **`dotnet-trace` on Windows runs even when an ETW session is also active.** EventPipe and ETW are independent transports. There's no cross-talk, but you double the in-process emission cost. Pick one transport per investigation on Windows.
- **`--diagnostic-port` listens on a Unix socket, not a TCP port** despite the "port" name. Cannot be reached across a network without a relay (`socat`, `dotnet-monitor`).
- **Remember the difference between `.nettrace` and `.diagsession`.** Visual Studio sometimes wraps a `.nettrace` in a `.diagsession` for the IDE's own session viewer. The inner `.nettrace` is the portable artefact — extract before sharing externally.
