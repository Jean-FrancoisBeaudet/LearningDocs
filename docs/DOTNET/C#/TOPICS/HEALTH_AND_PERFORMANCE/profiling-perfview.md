# PerfView

_Targets .NET 10 / C# 14. See also: [ETW](./etw-event-tracing-for-windows.md), [EventPipe](./eventpipe.md), [dotnet-trace](./dotnet-trace.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [GC pressure](./gc-pressure.md), [dotMemory](./profiling-dotmemory.md), [dotTrace](./profiling-dottrace.md)._

Microsoft's free ETW / EventPipe analysis tool. Written by the CLR team for their own use, which is why it sees more about the runtime than any other tool: **every** GC event, every JIT compile, every allocation tick, every contention event, every exception, the full ThreadPool internal state. PerfView is the right answer when the question is about *what the runtime did*, and the right answer when GC pause analysis or allocation source attribution must be defensible.

It is also the most user-hostile of the major profilers — dense Win32-style GUI, terse error messages, and a stack-folding/grouping language you have to learn. Once you do, nothing matches its depth.

## Where it fits

| Strength | Weakness |
|---|---|
| Free, no license, ships standalone | Windows GUI; opens `.nettrace` from Linux but you analyse on Windows |
| Decodes every CLR provider faithfully | Steep learning curve — `IncPats`, `ExcPats`, `GroupPats`, fold patterns |
| Reads `.etl`, `.nettrace`, `.gcdump`, `.diagsession` | No timeline view (use dotTrace for that) |
| Scriptable via command-line collection | Heap walking UX is dated (use dotMemory for retention paths) |
| GC stats are the source of truth — same engine the runtime team ships | No async-aware call stack reconstruction |

Reach for PerfView when:

1. You suspect a GC problem and need a **quantitative** picture (pause distribution, induced collections, LOH allocation rate).
2. You need to attribute allocations to a stack — `MemoryDiagnoser` says "a lot of bytes", PerfView's `AllocationSampling` says "from this method".
3. You want to capture in production with bounded overhead and bounded file size.
4. You're on Linux and need to analyse what the runtime emitted via EventPipe (`dotnet-trace` outputs `.nettrace`, PerfView reads it).

## Collection

```cmd
:: Default: GC, exceptions, JIT, ThreadPool, contention, sampled CPU.
PerfView.exe collect /AcceptEULA /CircularMB=500 /Merge:true /MaxCollectSec=120 trace.etl

:: Just GC events (small file, low overhead — daily prod-safe baseline).
PerfView.exe collect /GCCollectOnly /CircularMB=200 /MaxCollectSec=300 gconly.etl

:: Add allocation sampling (one stack per ~100 KB allocated).
PerfView.exe collect /ClrEvents:GC,GCAllocationTicks /CircularMB=500 alloc.etl

:: From a .NET Core process on Linux: use dotnet-trace, then open in PerfView.
dotnet-trace collect --process-id 12345 \
  --providers Microsoft-Windows-DotNETRuntime:0x1:5 \
  --duration 00:00:60
```

`/CircularMB` matters: ETW writes to a ring buffer, then flushes the last N MB. A circular buffer is the only safe collection mode in production — it bounds disk usage and you never accidentally fill `C:`. `/Merge:true` is required for a portable file (otherwise symbol resolution breaks on a colleague's machine).

## CLR provider keywords (the bitmask)

The single most-used flag is `Microsoft-Windows-DotNETRuntime` and its keywords. The bitmask shape is `<ProviderGuid>:<Keywords>:<Level>`. Common keywords:

| Keyword | Hex | What it emits |
|---|---|---|
| GC | `0x1` | Allocations, generations, pause begin/end, heap stats |
| GCHandle | `0x2` | Handle creation/destruction |
| Loader | `0x8` | Assembly load events |
| Jit | `0x10` | JIT compile begin/end, tiering transitions |
| NGen / R2R | `0x20` | Pre-JIT image events |
| Exception | `0x8000` | Every managed exception thrown |
| Contention | `0x4000` | Monitor.Enter contention |
| ThreadPool | `0x10000` | Worker enqueue/dequeue, IO completion |
| GCSampledObjectAllocationHigh | `0x200000` | Allocation samples, ~1 in 100 KB |

Combine with `|` in the dotnet-trace syntax or sum in hex for ETW: `0x1 | 0x10 | 0x4000 = 0x4011`.

## Core views

| View | Question it answers |
|---|---|
| **GCStats** | Pause time distribution, gen-by-gen counts, induced GCs, LOH allocation rate, suspension/resumption phases |
| **JITStats** | What got JITted, when, and how long it took (relevant for cold-start / first-request latency) |
| **CPU Stacks** | Sampled call stacks (default 1 ms); read with grouping/folding to find hot paths |
| **Thread Time Stacks** | Wall-clock attribution including blocked time — closest PerfView gets to a "where did time go" view |
| **Any Stacks** | Attribute *any* event (allocations, exceptions, contention) to the stack that caused it |
| **Memory → GC Heap Dump** | Walk a forced heap dump; less navigable than dotMemory but free |
| **Events** | Raw ETW events filtered by provider — the escape hatch when nothing else helps |

The `GC Stats` HTML report is what to send to a backend team when arguing about a GC regression: it has pause percentiles, induced-vs-natural counts, server-vs-workstation breakdown, and per-gen allocation rates. It's defensible.

## Stack folding — the unavoidable skill

Raw stacks are 70+ frames deep and unreadable. PerfView's folding language collapses noise. The four parameters:

- **GroupPats** — collapse frames matching a pattern into a synthetic node. Default `[group module entries]` collapses by DLL.
- **FoldPats** — drop the matching frame and attribute its time to its caller. Use to remove `mscorlib`, `System.Private.CoreLib`, `kernel32` noise.
- **IncPats** — only stacks that contain this pattern. Use to filter to one request type.
- **ExcPats** — drop stacks containing this pattern. Use to remove unrelated background work.

Worked example for "where does the JSON serializer allocate?":

```
GroupPats: [Just my code]
FoldPats:  System.Private.CoreLib;System.Text.Json!*MoveNext
IncPats:   MyApp!*Controller*
```

Save these as a stack-view preset; you'll reuse them daily.

## Workflow: GC pause incident

A latency p99 spike showed up; counters point at GC.

1. **Collect a small `GCCollectOnly` trace** (`/CircularMB=200 /GCCollectOnly`). 50–200 KB output, almost zero overhead.
2. Open in PerfView → **GCStats**. Look for: Gen2 pause percentile (anything >100 ms is suspicious), induced collections (`Reason: Induced` = someone called `GC.Collect()` — find them), LOH allocations per second.
3. If pauses are Gen2 and natural, you have promotion pressure. Re-capture with `/ClrEvents:GC,GCAllocationTicks` and open **GC Heap Alloc Stacks** to attribute allocations to call stacks.
4. If pauses are induced, search the codebase for `GC.Collect` and middleware/library calls — Newtonsoft.Json's contract resolver historically did this; some metrics libraries still do.
5. Cross-check with [`gc-generations-gen0-gen1-gen2.md`](./gc-generations-gen0-gen1-gen2.md) for the promotion patterns to look for in code.

## Linux: PerfView reads what dotnet-trace writes

```bash
# On the Linux box.
dotnet-trace collect \
  --process-id 12345 \
  --providers Microsoft-Windows-DotNETRuntime:0x4011:5 \
  --duration 00:01:00 \
  --output /tmp/svc.nettrace

# Copy to a Windows machine.
scp box:/tmp/svc.nettrace .

# Open svc.nettrace in PerfView; same views work.
```

EventPipe is a subset of ETW — kernel events (file IO, hardware counters) are missing, but every CLR provider works.

## Senior-level gotchas

- **ETW collection requires admin.** `PerfView collect` will silently fall back to a smaller event set (or fail) under a non-elevated user. EventPipe (`dotnet-trace`) does not need admin and is the right choice on locked-down hosts.
- **Full call stacks on a busy server cause an event storm.** Default stack capture is 1 sample/ms per CPU — on a 64-core box that's 64 K stacks/sec. Use `/CircularMB` ruthlessly and prefer `/GCCollectOnly` for first-pass production captures.
- **Allocation sampling captures one stack per ~100 KB allocated**, not every allocation. Many tiny short-lived allocations vanish into the gap. To attribute small-allocation-heavy hot paths, supplement with [`benchmarkdotnet.md`](./benchmarkdotnet.md) `[MemoryDiagnoser]`.
- **`/Merge:true` is essential** before sharing an `.etl` file. Without it, the file references local PDBs and symbol servers — opens fine on your box, useless on anyone else's.
- **Symbol resolution is fragile.** Set `_NT_SYMBOL_PATH=srv*c:\symbols*https://msdl.microsoft.com/download/symbols` before opening. Without it, framework frames show as offsets and the trace becomes a riddle.
- **PerfView does not understand `await` continuations.** A request that yields and resumes on a ThreadPool thread appears as two unrelated stacks. For async-aware reconstruction use dotTrace timeline mode.
- **`Thread Time Stacks` requires `/ThreadTime` at collection** (which is the default in `collect` but not in some scripts). If the view is empty, recapture with the flag explicit.
- **Induced GCs come from somewhere.** "Induced" reason means *managed code* called `GC.Collect()` — it's not the runtime deciding. Most frequent culprits: tests, `MemoryDiagnoser`, legacy logging libraries, and "warmup" code that someone wrote in 2012. Search and remove.
- **GC Stats counts background-GC pause as one event with two short pauses.** The HTML report shows both; the percentile table shows the longer one. Don't double-count when reporting up.
- **`.etl` files don't survive being copied without `/Merge`** — even a zip-and-email roundtrip can break PDB lookups. Always `PerfView merge file.etl` before transport.
- **PerfView's heap snapshot view loads the whole graph into memory.** On large heaps (>4 GB) prefer dotMemory or `dotnet-gcdump` for navigation; PerfView for the events that surround the dump.
