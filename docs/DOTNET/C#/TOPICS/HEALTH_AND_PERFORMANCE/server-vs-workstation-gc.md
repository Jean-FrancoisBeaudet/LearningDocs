# Server vs Workstation GC

_Targets .NET 10 / C# 14. See also: [How GC works](../../MEMORY_MANAGEMENT/how-gc-works.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [DATAS](./datas-dynamically-adapting-to-application-sizes.md), [GC pressure](./gc-pressure.md)._

The runtime ships **two GC flavors**: **Workstation** (one heap, one GC thread) and **Server** (one heap *per logical core* — give or take, with DATAS — collected by N parallel GC threads). They share the same generational/tracing/compacting algorithm and the same Background GC machinery; the difference is entirely about how parallelism, working set, and pause profile trade off.

You **pick once at startup**. The mode is a process-level decision baked from `runtimeconfig.json` / env / csproj — there is no API to switch at runtime.

## Comparison at a glance

| Dimension | Workstation | Server |
|---|---|---|
| GC threads | 1 | ~1 per logical core (or DATAS-managed) |
| Heaps | 1 | N (one per GC thread) |
| Throughput on multi-core | Lower | Higher (parallel mark + compact) |
| Pause time goal | Short, tolerates throughput hit | Tolerates longer stop-the-world for bigger throughput payoff |
| Working-set cost | Lower | Higher — each heap has its own ephemeral segment, allocation budget, free lists |
| Background GC | Yes | Yes (was historically called "Concurrent GC") |
| Default in | Console / desktop / WPF | ASP.NET Core, gRPC services, hosted services |
| Behaviour with 1 logical core | Optimal | Pathological — see below |

The throughput edge of Server GC comes from parallel work *during* a collection. The cost is per-heap memory: each heap reserves and grows its own ephemeral segment, so a process with 16 heaps will have a baseline working set noticeably larger than the same process with 1 heap.

## Choosing

**Server is the right default for**:
- ASP.NET Core (it's already on by default).
- gRPC / Kestrel-fronted services.
- Worker services consuming Kafka / Service Bus at high message rates.
- Anything multi-core, allocation-heavy, latency-sensitive at p99.

**Workstation is right for**:
- Single-core or 1-vCPU containers.
- Memory-constrained sidecars and tools (test runners, CLIs, batch helpers).
- Desktop UI apps (WPF / WinForms / MAUI) where the Server-GC pause profile would jank rendering.
- Test processes — low memory baseline matters more than throughput.

## Configuring the mode

Three places, in increasing precedence — env vars win.

**csproj** (msbuild property):
```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

**runtimeconfig.json** (after publish):
```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.HeapCount": 4,
      "System.GC.HeapHardLimitPercent": 75
    }
  }
}
```

**Environment variables** (override, useful for k8s):
```
DOTNET_gcServer=1
DOTNET_gcConcurrent=1
DOTNET_GCHeapCount=4
DOTNET_GCHeapHardLimit=2147483648
DOTNET_GCHeapHardLimitPercent=75
DOTNET_GCConserveMemory=5
DOTNET_GCDynamicAdaptationMode=1
```

Confirm at runtime:
```csharp
Console.WriteLine($"IsServerGC={GCSettings.IsServerGC}");
Console.WriteLine($"Latency  ={GCSettings.LatencyMode}");
Console.WriteLine($"HeapCount={GC.GetGCMemoryInfo().GenerationInfo.Length /* not heap count - see below */}");
```

For the *actual* heap count, check `GC.GetGCMemoryInfo().HeapCountInfo` (.NET 7+) or read `gc-heap-count` from `dotnet-counters` — `GenerationInfo` describes generations, not heaps.

## The 1-vCPU container trap

This is the single most common Server-GC misconfiguration in production.

ASP.NET Core templates default to Server GC. When deployed in a Kubernetes pod with `resources.limits.cpu: "1"` (or a Docker container with `--cpus=1`), the runtime sees the cgroup CPU quota and *should* now (.NET 6+) limit GC heap count accordingly — but the behavior has been buggy across versions and depends on whether the cgroup limit is set as a quota vs. a request, and whether `DOTNET_GCDynamicAdaptationMode` is enabled.

When it goes wrong, you get:
- **Several GC heaps** (one per *host* logical core, ignoring the limit).
- **No actual parallelism** (you have one CPU; threads serialize).
- **Working set inflated** by N ephemeral segments worth of headroom.
- **Worse pauses** because the "parallel" mark/compact phases are stop-the-world and now run sequentially on one core.

Fixes, in order of preference:
1. **Pin heap count** explicitly: `DOTNET_GCHeapCount=1`. Keeps Server-GC scheduling but with one heap.
2. **Enable DATAS** on .NET 8+: `DOTNET_GCDynamicAdaptationMode=1`. The runtime watches CPU usage and shrinks/grows heap count over time. This is the preferred long-term answer for variable-load services. See [`./datas-dynamically-adapting-to-application-sizes.md`](./datas-dynamically-adapting-to-application-sizes.md).
3. **Switch to Workstation**: `DOTNET_gcServer=0`. Simplest, lowest memory; smaller throughput.

Always validate with `dotnet-counters` (`gc-heap-count`, `% time in gc`, `gc-heap-size`) and a load test, not assumption.

## DATAS — when N is not fixed

**DATAS (Dynamically Adapting To Application Sizes)**, .NET 8+, is the answer to "Server GC's per-heap memory cost is wasteful when load is variable." Instead of a static N heaps decided at startup, the runtime starts small and grows the heap count when CPU/throughput justifies it, and shrinks when the app is idle.

- Off by default in .NET 8 (opt-in via `DOTNET_GCDynamicAdaptationMode=1`).
- On by default for some configurations in .NET 9+ (verify per release).
- Most useful in: Kubernetes / autoscaled environments where pods see varying QPS; multi-tenant hosts; idle-then-burst workloads.
- Less useful when: load is steady-state high (a fixed `GCHeapCount` is more predictable) or pods are very short-lived.

DATAS does not replace `GCHeapHardLimit` / cgroup detection; pair them.

## Background GC

Orthogonal to Workstation/Server. **On by default in both**, controlled by `<ConcurrentGarbageCollection>` / `DOTNET_gcConcurrent`.

- BGC runs the Gen2 mark/sweep concurrently with user threads, with two short pauses to synchronize.
- Disabling BGC is **almost never the right answer**. It brings full stop-the-world Gen2 pauses back. The only legitimate scenario is a memory-extremely-constrained tool where BGC's bookkeeping overhead is unaffordable — and even then, measure first.

## Tuning knobs

| Knob | Default | Effect |
|---|---|---|
| `GCHeapCount` | Auto (≈ logical cores or DATAS-driven) | Fixed heap count — useful in containers |
| `GCHeapHardLimit` | None | Bytes ceiling on managed heap |
| `GCHeapHardLimitPercent` | None | Percent of cgroup memory ceiling |
| `GCNoAffinitize` | false | Set true to let the OS schedule GC threads freely (multi-NUMA) |
| `GCConserveMemory` | 0 | 0–9 dial; higher = smaller heap, more frequent collections |
| `GCDynamicAdaptationMode` | 0 (off in 8) | 1 = enable DATAS |
| `GCRetainVM` | false | Set true to keep released memory committed for reuse (lower latency, higher RSS) |
| `LatencyMode` | `Interactive` | `SustainedLowLatency` to suppress Gen2 in a critical window |

`GCConserveMemory` is the most under-used knob. On memory-tight services, `5` or `7` measurably reduces working set with a small throughput tax.

## Observability checklist

Whenever a service goes through a load profile change, verify:

1. `GCSettings.IsServerGC` — confirms intended mode.
2. `dotnet-counters monitor System.Runtime` — `gc-heap-count`, `gc-heap-size`, `% time in gc`, `alloc-rate`.
3. `GC.GetGCMemoryInfo().HeapCountInfo` — actual runtime heap count (.NET 7+).
4. Pod memory limit vs `gc-heap-size` — should leave headroom for thread stacks, native heap, JIT cache, etc.
5. p99 latency under load — the actual SLO. If GC mode change improved counters but not latency, it wasn't the bottleneck.

**Senior-level gotchas:**
- **Local-dev RSS vs prod RSS aren't comparable.** A dev box runs Workstation by default in some host configurations; prod is Server. Don't escalate "memory regression" until you've confirmed both sides are running the same GC.
- **Server GC + 1 vCPU = bad time.** Always pin `GCHeapCount=1` (or use Workstation) in single-CPU containers.
- **`gc-heap-count` is per-process, not per-machine.** When co-locating multiple services on a node, total heap memory = sum across processes. Underprovisioning RAM is a common cause of OOMKill loops.
- **"Concurrent GC" ≠ "Server GC".** Concurrent (now BGC) is about whether Gen2 can run alongside user threads. Server is about how many heaps. They're independent.
- **Container CPU detection has been buggy.** .NET 6 added cgroup v2 support but quirks remain across .NET versions and host kernels. If `gc-heap-count` looks wrong vs. CPU limit, set it explicitly.
- **`GCRetainVM=1` lowers tail latency** by keeping pages committed to avoid OS reallocation cost — at the price of higher RSS. A reasonable choice for steady-state services with strict pause SLOs and slack memory.
- **`GCNoAffinitize=1` matters on multi-NUMA hardware.** By default GC threads pin to cores; on NUMA boxes pinning sometimes hurts. Almost never relevant in cloud-VM deployments.
- **DATAS interactions:** with DATAS on, a fixed `GCHeapCount` setting is overridden by runtime adaptation. If you want strict pinning, leave DATAS off.
- **Workstation can have higher pause time on large heaps.** "Workstation = short pauses" is true at small heap sizes but stops being true past a few hundred MB live set, since one thread does all compacting. Server GC's parallelism wins above that scale.
- **The choice you make at startup persists for process lifetime.** "We'll switch to Server later" is not a thing. Plan for restarts.
