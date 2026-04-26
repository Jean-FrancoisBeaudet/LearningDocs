# ETW (Event Tracing for Windows)

_Targets .NET 10 / C# 14, Windows 10 / Server 2019+. See also: [EventPipe](./eventpipe.md), [PerfView](./profiling-perfview.md), [dotnet-trace](./dotnet-trace.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [GC pressure](./gc-pressure.md), [BenchmarkDotNet](./benchmarkdotnet.md)._

ETW is Windows' **kernel-level, system-wide tracing infrastructure**. It predates .NET, runs in the kernel, and instruments everything from disk I/O to process scheduling to TCP connections. Every Windows-native diagnostic — Performance Monitor, Event Viewer, Windows Performance Recorder, PerfView, Visual Studio Diagnostic Tools, the Windows Performance Toolkit, BenchmarkDotNet's `[EtwProfiler]` — is a consumer of ETW. The CLR ships its own ETW provider (`Microsoft-Windows-DotNETRuntime`) so managed events flow into the same trace as the kernel events around them.

ETW is **the only choice** when the question crosses the managed/native boundary: "is this latency in my code or in the kernel?", "is the GC stalling because of a page fault?", "what was the disk doing during that pause?". For pure managed-event capture on Windows, [EventPipe](./eventpipe.md) is lighter, no-admin, and emits the same CLR events. ETW's value is **breadth**, not just CLR coverage.

## Where it fits

| Strength | Weakness |
|---|---|
| Kernel + user events in one timeline | Windows-only; no Linux/macOS equivalent |
| Sees file I/O, network, page faults, context switches, DPC/ISR | Admin / `Performance Log Users` membership required for kernel providers |
| Hardware performance counters (PMU) — branch mispredicts, cache misses | `.etl` files balloon fast (multi-GB in minutes at high verbosity) |
| System-wide — capture across all processes simultaneously | Steep learning curve — provider GUIDs, keyword bitmasks, log mode flags |
| Native CLR support back to .NET Framework 4.0 | Symbol resolution depends on `_NT_SYMBOL_PATH` and a merged file |

Reach for ETW when:

1. The investigation crosses managed/native — disk slowness, network slowness, kernel scheduler issues correlated with GC.
2. You need **system-wide** capture: multiple .NET processes plus IIS plus SQL Server, all in one timeline.
3. You're on **.NET Framework** — EventPipe doesn't exist there, ETW is the only option.
4. You need PMU events (instructions retired, cache misses) for micro-architectural analysis.

## Architecture

```
                   Kernel
                     │
   ┌─────────────────┼─────────────────┐
   │ Kernel logger (NT Kernel Logger)  │
   │  - PROC_THREAD                    │
   │  - DISK_IO                        │
   │  - NETWORK_TCPIP                  │
   │  - CSWITCH / DPC / ISR            │
   │  - PMC_PROFILE (hardware counters)│
   └─────────────────┬─────────────────┘
                     │
   ┌─────────────────┴─────────────────┐
   │       ETW dispatcher              │ ◄── controllers (logman, xperf, PerfView)
   └─────────────────┬─────────────────┘
                     │
   ┌─────────────────┴─────────────────┐
   │       User-mode providers         │
   │  Microsoft-Windows-DotNETRuntime  │
   │  Microsoft-AspNetCore-...         │
   │  Your EventSource                 │
   └───────────────────────────────────┘
                     │
                     ▼
                 .etl file
```

- **Providers** (kernel and user) emit events. Each has a **GUID** and exposes **keyword bitmasks** (which event categories to enable) and **levels** (`1`=Critical → `5`=Verbose).
- **Sessions** are configured by a *controller* (`logman`, `xperf`, PerfView). The session attaches a list of `(provider, keywords, level)` tuples and a buffer/log policy.
- **`.etl` log files** are the on-disk format. After capture, they must be **merged** so they include the symbol/image rundown — without merge, files are non-portable.
- **Consumers** read `.etl` files (PerfView, WPA, `tracerpt`, `TraceEvent` library) or attach in **real-time mode** to a live session.

## Provider model

The CLR provider GUID is `{e13c0d23-ccbc-4e12-931b-d9cc2eee27e4}`, exposed by name as `Microsoft-Windows-DotNETRuntime`. Keywords match those in [EventPipe](./eventpipe.md) — the bitmask is a portable concept across both transports.

Common keywords (full table in [`profiling-perfview.md`](./profiling-perfview.md)):

| Keyword | Hex | Events |
|---|---|---|
| GC | `0x1` | Allocations, generations, pause begin/end, suspension/resumption |
| Loader | `0x8` | Assembly loads (essential for symbol resolution) |
| Jit | `0x10` | JIT compile begin/end, tiering, code-versioning |
| NGen / R2R | `0x20` | Pre-JIT image events |
| Contention | `0x4000` | `Monitor.Enter` contention |
| Exception | `0x8000` | Every managed exception thrown |
| ThreadPool | `0x10000` | Worker enqueue/dequeue, IO completion |
| GCSampledObjectAllocationHigh | `0x200000` | One allocation stack per ~100 KB allocated |

Kernel keywords are different — provided as flag names rather than hex via `xperf`/`logman`:

```
PROC_THREAD     - Process and thread create/exit
LOADER          - Image loads (DLL/EXE)
DISK_IO         - File-system disk I/O
FILE_IO         - File-system operations (open/close/read/write)
FILE_IO_INIT    - File-system request initiation timestamps
NETWORK_TCPIP   - TCP/UDP send/receive
CSWITCH         - Context switches (the big one — ~100 K events/sec on a busy box)
DPC, ISR        - Deferred procedure calls and interrupt service routines
PMC_PROFILE     - Hardware performance counters (Intel/AMD PMU)
```

## Capture: PerfView (recommended)

```cmd
:: Standard CPU + GC + JIT + ThreadPool + sampled stacks, 2 minutes, 500 MB ring.
PerfView.exe collect /AcceptEULA /CircularMB=500 /Merge:true /MaxCollectSec=120 trace.etl

:: GC-only — small, prod-safe baseline.
PerfView.exe collect /GCCollectOnly /CircularMB=200 /MaxCollectSec=300 gc.etl

:: Allocation-attribution capture.
PerfView.exe collect /ClrEvents:GC,GCAllocationTicks /CircularMB=500 alloc.etl

:: Custom user provider plus CLR.
PerfView.exe collect /OnlyProviders="*Acme-Orders:0x1:5,*Microsoft-Windows-DotNETRuntime:0x4011:5" /CircularMB=200 acme.etl
```

`/CircularMB` makes the buffer a ring — capture stops only when triggered, but disk usage is bounded. `/Merge:true` is non-negotiable for a portable file.

## Capture: `logman` (no GUI required)

```cmd
:: Create the session (does not start it).
logman create trace clrtrace -ow -o C:\traces\clr.etl ^
  -p "Microsoft-Windows-DotNETRuntime" 0x4011 0x5 ^
  -nb 16 256 -bs 1024 -mode Circular -max 500 -ets

:: Start, wait while reproducing, stop.
logman start clrtrace -ets
:: ... reproduce the issue ...
logman stop clrtrace -ets

:: Merge for portability.
xperf -merge C:\traces\clr.etl C:\traces\clr.merged.etl

:: Inspect from the command line.
tracerpt C:\traces\clr.merged.etl -o C:\traces\clr.xml -of XML
```

`-mode Circular` plus `-max 500` (MB) is the production-safe pattern. `-ets` is "Event Trace Session" — runs without persisting controller config to disk.

## Capture: kernel + CLR together (`xperf`)

```cmd
:: Two sessions: one kernel, one user.
xperf -on PROC_THREAD+LOADER+DISK_IO+FILE_IO+CSWITCH ^
      -stackwalk CSwitch+ReadyThread ^
      -f C:\traces\kernel.etl

xperf -start clr -on Microsoft-Windows-DotNETRuntime:0x4011:5 -f C:\traces\clr.etl

:: ... reproduce ...

xperf -stop clr -stop -d C:\traces\merged.etl
```

`xperf -d` runs the merge for you. The merged file shows kernel and managed events on a single timeline — the only way to answer "did this GC pause overlap a disk write?".

## Programmatic reading: `Microsoft.Diagnostics.Tracing.TraceEvent`

```csharp
using Microsoft.Diagnostics.Tracing;
using Microsoft.Diagnostics.Tracing.Etlx;
using Microsoft.Diagnostics.Tracing.Parsers;

using var log = TraceLog.OpenOrConvert("trace.etl");

var clr = new ClrTraceEventParser(log.Events.GetSource());

clr.GCStart += e =>
    Console.WriteLine($"GC #{e.Count} reason={e.Reason} gen={e.Depth} at {e.TimeStampRelativeMSec:F2} ms");

clr.ContentionStart += e =>
    Console.WriteLine($"contention thr={e.ThreadID} at {e.TimeStampRelativeMSec:F2} ms");

log.Events.GetSource().Process();
```

`TraceEvent` is the same library PerfView is built on. Use it for custom regression detectors (parse a CI-collected `.etl`, fail if Gen2 pause p99 > target).

## When to choose ETW over EventPipe

| Need | Pick |
|---|---|
| .NET Framework | ETW |
| Linux / macOS / containers | EventPipe (no choice) |
| Kernel I/O correlation | ETW |
| PMU / hardware counters | ETW |
| System-wide capture (multiple processes) | ETW |
| No admin available | EventPipe |
| Identical-CLR coverage on Windows | Either; EventPipe simpler |
| Custom in-proc consumer | Either; EventPipe simpler |

## Senior-level gotchas

- **Kernel providers require admin.** Specifically the user must be in `Performance Log Users` (or Administrators). On a hardened server with no admin, you're stuck with EventPipe / `dotnet-trace`. Don't try to "elevate just for this trace" without ops sign-off.
- **`.etl` files are not portable without `/Merge`.** A merged file embeds image/PDB info; an unmerged file references local symbol caches. Email an unmerged `.etl` and the recipient sees offsets, not method names. Always `xperf -merge` before transport.
- **Symbol resolution is the silent failure mode.** Set `_NT_SYMBOL_PATH=srv*c:\symbols*https://msdl.microsoft.com/download/symbols` before opening. Without it, `coreclr!??` everywhere.
- **Stack-walking is per-event-class — and OFF by default for most.** `xperf -stackwalk CSwitch+ReadyThread+DiskRead` enables stack capture for those events. Forgetting it means a trace with rich timing but no callers.
- **CSwitch is a firehose.** A 64-core box can emit 1 M context-switch events/sec. Without `/CircularMB` and a tight time window, the file fills the disk in seconds. Use sparingly and prefer real-time consumers when possible.
- **PMU (`PMC_PROFILE`) is hardware-counter-driven and serialises with the OS.** It needs `bcdedit /set DisableDynamicTick yes` and a reboot on some Windows versions. Plan ahead; this isn't an "ad hoc on a prod box" tool.
- **The CLR provider keyword `0x4000` (Contention) excludes spin-waits.** Only contention that goes to kernel-mode wait counts. A hot spin loop competing for a `lock` produces no events. Pair with kernel `READY_THREAD` to see the full picture.
- **Multiple controllers can collide.** Only one session per provider with the same name; a second `logman start` for `clrtrace` fails silently. List active sessions with `logman query -ets` before starting new ones.
- **`tracerpt` is fine for sanity checks, useless for analysis.** XML/CSV dumps of millions of events are unreadable. Use PerfView or WPA. Reach for `tracerpt` only to confirm a session captured *something*.
- **EventSource events flow into ETW automatically on Windows.** Same provider/keyword model — your custom `EventSource` shows up in PerfView under its `Name`. No extra registration needed (no manifest registration as in classic ETW).
- **ETW does not stop on process exit.** A session created with `-ets` outlives the controller. If the controlling app crashes, the session keeps running and burning disk. Always wrap controller logic in try/finally and call `logman stop ... -ets` on shutdown.
- **`/Merge` and rundown matter most for short-lived processes.** The CLR emits a JIT/method rundown when its session ends; if a session captures across multiple process lifetimes, only those that ended *during* the session are decodable. For a startup investigation, ensure the process is stopped before the trace is.
- **Sysinternals' `xperf`/`wpa` is the modern UX.** WPA (Windows Performance Analyzer) reads the same `.etl` PerfView reads but offers a timeline view PerfView lacks. Use both: WPA for "when did it happen", PerfView for "what stack caused it".
- **PerfView's `EtwProfiler` in BenchmarkDotNet** captures a per-benchmark `.etl` automatically — use it when the benchmark itself reveals nothing and you need to see what the runtime did. See [`benchmarkdotnet.md`](./benchmarkdotnet.md).
