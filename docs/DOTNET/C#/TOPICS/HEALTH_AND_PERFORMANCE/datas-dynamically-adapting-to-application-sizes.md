# DATAS (Dynamically Adapting To Application Sizes)

_Targets .NET 10 / C# 14. See also: [Server vs Workstation GC](./server-vs-workstation-gc.md), [GC generations (gen0 / gen1 / gen2)](./gc-generations-gen0-gen1-gen2.md), [GC pressure](./gc-pressure.md), [Heap allocation patterns](./heap-allocation-patterns.md)._

DATAS is a Server-GC mode (introduced in .NET 8) where the runtime **adjusts heap count and ephemeral segment sizing in response to live CPU and throughput**, instead of pinning one heap per logical core at startup. It exists to fix Server GC's dirty secret: the per-heap memory tax that wrecks autoscaled and multi-tenant deployments. With one heap per logical core, an idle 32-vCPU pod carries 32 ephemeral segments worth of headroom; with DATAS, the runtime collapses to fewer heaps when load is low and grows back when traffic picks up.

It does not replace cgroup-aware sizing. It does not toggle Background GC. It does not exist under Workstation GC. And it is not a silver bullet — for some workloads, fixed `GCHeapCount` remains the right answer.

## What problem DATAS solves

Server GC's design assumes you want maximum throughput by default: parallel collection across N heaps where N = logical cores. The cost is per-heap working-set headroom that compounds badly in:

- **Kubernetes pods with bursty QPS** — most of the day idle, brief spikes. The static N-heap layout reserves memory it doesn't use 90 % of the time.
- **Multi-tenant hosts** (App Service multi-app, Functions Premium, container density on a node) — every co-located process pays the same per-heap tax. RAM is the bottleneck, not throughput.
- **Idle-then-burst workers** (cron-style, queue drain) — a worker that processes 30 seconds of work then sleeps for 5 minutes still holds 16 heaps' worth of segments.

Pre-DATAS, the answers were **(a) pin `GCHeapCount=1` or `=2`** (lose throughput) or **(b) switch to Workstation GC** (lose throughput more, gain memory). DATAS is the third answer: stay on Server GC, let the runtime size for current load.

## How it works (conceptually)

The runtime samples CPU usage, allocation rate, and pause-time metrics over a window of seconds, and decides whether to:

- **Grow** the active heap count (up to `GCHeapCount` or logical-core count, whichever is set) when allocation pressure or CPU justifies the parallelism win.
- **Shrink** the active heap count and reclaim the ephemeral segments back to the OS when the app is idle.

The cadence is on the order of seconds, not milliseconds — DATAS won't track sub-second spikes, and it won't fix a service whose load profile is "every request is a burst". For services with steady-state high load, fixing the heap count gives more predictable counters (RSS doesn't hunt around) at the cost of accepting the worst-case memory footprint.

## Version timeline

| .NET version | Default | How to enable |
|---|---|---|
| .NET 8 | Off | `DOTNET_GCDynamicAdaptationMode=1` env var, `<GCDynamicAdaptationMode>1</GCDynamicAdaptationMode>` MSBuild, or `System.GC.DynamicAdaptationMode` runtimeconfig |
| .NET 9 | On by default for Server GC in many configurations | Inverse — set the value to `0` to opt out |
| .NET 10 | Continued tuning; default-on remains for Server GC | Same toggle |

Verify the live setting at runtime — defaults shift between releases:

```csharp
Console.WriteLine($"IsServerGC = {GCSettings.IsServerGC}");
foreach (var (key, value) in GC.GetConfigurationVariables())
    if (key.Contains("Adaptation", StringComparison.OrdinalIgnoreCase))
        Console.WriteLine($"{key} = {value}");
```

`GC.GetConfigurationVariables()` (.NET 8+) returns the runtime's view of GC config — use it in production diagnostics rather than trusting the documented defaults.

## Configuration

DATAS is a **process-level** decision, set at startup. There is no API to toggle it on a running process.

**MSBuild (csproj)**:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <GCDynamicAdaptationMode>1</GCDynamicAdaptationMode>
</PropertyGroup>
```

**runtimeconfig.json** (after publish):

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.DynamicAdaptationMode": 1,
      "System.GC.HeapHardLimitPercent": 75
    }
  }
}
```

**Environment variable** (highest precedence — useful in Kubernetes manifests where you don't want to rebuild the image):

```
DOTNET_gcServer=1
DOTNET_GCDynamicAdaptationMode=1
DOTNET_GCHeapHardLimitPercent=75
```

## Interaction with other GC knobs

| Other knob | Interaction |
|---|---|
| `GCHeapCount` | **DATAS overrides it.** Setting both is a config smell — DATAS wins, the explicit value is misleading. Pick one. |
| `GCHeapHardLimit` / `GCHeapHardLimitPercent` | **Still respected.** DATAS controls heap *count*; hard limits cap heap *size*. Pair them in containers. |
| Workstation GC | **No effect.** DATAS is Server-GC-only — silently no-op under Workstation. |
| `GCConserveMemory` (0–9) | **Compatible.** Higher values bias the runtime toward smaller heaps; DATAS tunes the count dynamically on top. |
| `GCRetainVM` | **Compatible.** Keeps released memory committed for reuse — DATAS reclaims pages in the OS sense, `GCRetainVM` keeps them committed as a latency optimization. |
| Background GC | **Independent.** BGC governs whether Gen2 runs concurrently; DATAS governs how many Server-GC heaps participate. Both on by default. |

A typical container deployment that wants memory discipline:

```
DOTNET_gcServer=1
DOTNET_GCDynamicAdaptationMode=1
DOTNET_GCHeapHardLimitPercent=80   # cap inside the cgroup memory limit
DOTNET_GCConserveMemory=5          # bias toward smaller heap
```

## Workloads where DATAS helps

- **Autoscaled web services** in Kubernetes / App Service with variable QPS — the heap count tracks load.
- **Multi-tenant hosts** running many low-traffic apps on the same node — each app's idle footprint shrinks.
- **Worker services** with cron-style or queue-drain workloads — the worker collapses heaps during the idle window.
- **Functions Premium / dedicated plans** — the per-instance memory budget can host more app instances when DATAS reclaims idle heap space.

## Workloads where DATAS is neutral or harmful

- **Steady-state high-load services** — load is constant, so the dynamic adaptation has nothing to track. A fixed `GCHeapCount` gives more predictable RSS curves, easier capacity planning, and no transient counter swings.
- **Very short-lived processes** (CLI tools, build-step processes, tests) — the process exits before DATAS has a window to adapt. The cost of the adaptation logic, while small, is wasted.
- **Latency-critical p99 SLOs** — heap count changes can cause transient pauses or RSS bumps that show up as outliers. Some teams pin `GCHeapCount` and accept the worst-case memory cost rather than tolerate the variability.
- **Single-vCPU containers** — already a degenerate case for Server GC. Pin `GCHeapCount=1` (see [server-vs-workstation-gc.md](./server-vs-workstation-gc.md)) or use Workstation; DATAS doesn't fix the underlying mismatch.

## Observability

DATAS makes some counters **time-varying** that previously were constants. Update dashboards and alerts accordingly.

| Signal | Where | Note |
|---|---|---|
| `gc-heap-count` | `dotnet-counters monitor System.Runtime` | **Will change at runtime under DATAS.** Don't alert on absolute value — alert on out-of-bound trends or rapid oscillation |
| Live heap count | `GC.GetGCMemoryInfo().HeapCountInfo` (.NET 7+) | Authoritative per-process value at the moment of call |
| `gc-heap-size` | `dotnet-counters` | Total managed heap; DATAS adjusts the constituent ephemeral segments |
| `% time in gc` | `dotnet-counters` | If DATAS is shrinking aggressively, expect more frequent (smaller) collections |
| Configuration view | `GC.GetConfigurationVariables()` | Runtime's actual mode — survives env-var typos that silently disabled the feature |

```csharp
// Snapshot for diagnostics endpoint
var info = GC.GetGCMemoryInfo();
return new
{
    IsServerGC = GCSettings.IsServerGC,
    LatencyMode = GCSettings.LatencyMode.ToString(),
    LiveHeapCount = info.HeapCountInfo,
    HeapSizeBytes = info.HeapSizeBytes,
    Generation = info.Generation,
    PauseTimePercentage = info.PauseTimePercentage,
};
```

## Senior-level gotchas

- **DATAS doesn't replace cgroup-aware heap sizing.** Pair with `GCHeapHardLimitPercent` for a hard cap inside the container memory limit. Without it, DATAS still respects per-heap segment sizing but doesn't know the cgroup ceiling.
- **Setting `GCHeapCount` *and* enabling DATAS is a config smell.** DATAS wins; the explicit value is misleading and confuses anyone debugging six months later. Decide and document — one or the other.
- **DATAS adjustment cadence is seconds, not milliseconds.** A request burst that arrives in 200 ms will be served by whatever heap count was active at burst start. For sub-second burst handling, fixed `GCHeapCount` sized for peak is more predictable.
- **`gc-heap-count` is no longer a constant** — alerts that previously checked `gc-heap-count == 16` will fire spuriously on idle. Migrate alert logic to "out of bounds" or "oscillating beyond N changes per minute".
- **Background GC and DATAS are independent.** Disabling BGC ("concurrent GC") is almost always wrong regardless of DATAS — it brings full stop-the-world Gen2 pauses back. Don't conflate the two flags.
- **DATAS interacts with `GCNoAffinitize`.** On NUMA hardware, the GC normally affinitizes threads to cores; DATAS scaling can churn affinity if the heap count moves rapidly. On dense multi-NUMA boxes, set `GCNoAffinitize=1` and let the OS schedule.
- **The decision belongs at deployment-config level, not in code.** A library can't know whether the host service wants DATAS — that's an ops decision tied to the deployment shape (autoscaled vs steady-state). Keep `GCDynamicAdaptationMode` out of csproj for libraries; set it in the host project or env.
- **DATAS cannot detect "this is a CLI tool".** A short-lived process inherits whatever the host config sets. For tools that wrap a runtime startup hot path (build steps, test runners), explicitly disable DATAS via env var if you've measured a regression.
- **The runtime's heap-count adjustment is observable as small RSS dips.** Expect ragged-looking RSS graphs in steady-state — they're the runtime returning pages to the OS, not a leak. `GCRetainVM=1` smooths the graph at the cost of higher steady-state RSS.
- **DATAS doesn't help if the bottleneck is allocation rate, not heap topology.** A service allocating 5 GB/s will still GC frequently regardless of heap count — fix the allocations first ([gc-pressure.md](./gc-pressure.md), [heap-allocation-patterns.md](./heap-allocation-patterns.md)). DATAS is about *idle* memory, not active throughput.
- **Process restart resets the adaptation state.** Fresh process starts with a small heap count and grows. Cold-start latency profile differs from warm-state — load tests should run for at least one DATAS adaptation window (typically a minute) before measuring p99.
- **Default behavior changes between .NET versions.** Don't assume your .NET 8 service behaves the same after a .NET 10 upgrade — confirm with `GC.GetConfigurationVariables()` in production after the upgrade rolls out, and re-baseline RSS curves in dashboards.
