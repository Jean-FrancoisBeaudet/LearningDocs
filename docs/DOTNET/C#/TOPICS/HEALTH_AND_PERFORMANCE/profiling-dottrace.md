# dotTrace

_Targets .NET 10 / C# 14. See also: [dotMemory](./profiling-dotmemory.md), [PerfView](./profiling-perfview.md), [Visual Studio Diagnostic Tools](./profiling-visual-studio-diagnostic-tools.md), [dotnet-trace](./dotnet-trace.md), [async/await performance](./async-await-performance.md), [Thread contention](./thread-contention.md), [Synchronization bottlenecks](./synchronization-bottlenecks.md)._

JetBrains' CPU and timeline profiler. The differentiator versus PerfView is the **timeline view** — per-thread state lanes (running / waiting / GC / UI / lock / IO) drawn on a horizontal axis, with the call stack at any cursor position synchronised to the events around it. That's the right tool for "where did this 200 ms request actually spend its time?", because the answer is rarely "in CPU" — it's lock contention, async continuation latency, or a downstream HTTP call.

## Profiling modes

| Mode | What it captures | Overhead | When to pick it |
|---|---|---|---|
| **Sampling** | Stack snapshot every ~1 ms | Low (~5%) | First look, long-running services, prod-like environments |
| **Tracing** | Enter/leave for every method | High (often 10–50×) | Need exact call counts; hot/cold path proven by sampling first |
| **Line-by-line** | Source-level attribution | Very high | Last-mile inside one tight method |
| **Timeline** | Sampling + ETW/EventPipe events (threads, GC, locks, IO, HTTP) | Moderate (~10–20%) | Latency / responsiveness investigations — the default for service work |

Tracing changes JIT inlining decisions because the hooks make every callsite non-inlineable. Numbers from tracing are useful for *ratios*, never for absolute timings. Sampling is what you cite in a perf bug.

## The timeline view — the killer feature

A typical investigation on a slow request:

1. Filter to the request's thread (or the relevant `await`-completion threads — see below).
2. Find the gap. The lane will show the thread state during that gap: blue (running), grey (waiting on sync object), pink (GC pause), orange (IO).
3. Hover the gap → dotTrace shows the blocking event (e.g. `Monitor.Wait`, `HttpClient.SendAsync`, `WaitForSingleObject`).
4. Click → the call stack at the moment of the block.
5. Cross-reference with the **Subsystems** view to confirm the category (System.Net, EF Core, Custom code).

This is the closest thing to a "why was nothing happening?" answer that any .NET tool offers. PerfView's `Thread Time Stacks` gives the same data but without the timeline alignment, which is half the value.

## Subsystems view

Aggregates time by logical layer:

- **Custom code** — your assemblies.
- **System.Net** — HttpClient, sockets.
- **EF Core / Database** — query execution time.
- **Reflection / JSON / XML** — serialization frameworks.
- **GC / Synchronization / Native** — runtime overhead.

A first-glance subsystem split tells you whether the bottleneck is your code, a framework, IO, or the runtime. Don't optimise your `for` loop if 70% of the time is in `System.Text.Json`.

## Async profiling

dotTrace correlates `await` continuations through the state-machine box, so a request that resumes on a different ThreadPool thread still appears as one logical flow. Plain ETW stack samples lose this — you see "ThreadPool worker → continuation → MoveNext" with no indication of *which* logical request the continuation belongs to.

This matters for the most common .NET latency shape: the request-handler thread issued an async DB call, the ThreadPool was saturated, and the continuation queued for 80 ms before resuming. dotTrace shows that gap explicitly; raw stack profiles do not. See [`async-await-performance.md`](./async-await-performance.md) and [`thread-contention.md`](./thread-contention.md).

## Headless / CI use

```bash
# Attach to a running PID, sample for 60 seconds, save snapshot.
dotTrace.exe attach 12345 ^
  --profiling-type=Sampling ^
  --timeout=60s ^
  --save-to=C:\traces\workload.dtt

# Or launch a .NET Core process with profiling enabled.
dotTrace.exe start-net-core ^
  --profiling-type=Timeline ^
  --save-to=C:\traces\soak.dtt ^
  -- C:\app\MyService.dll --config soak
```

Snapshots open in the GUI on any developer's machine. For soak runs on Linux there's `dotTrace` for Linux (same CLI shape) and the snapshots are cross-platform.

## Reading the snapshot — what to actually click

- **Call Tree** (top-down): "where does my entry point spend time?"
- **Hot Spots** (bottom-up): "which leaf method dominates regardless of caller?"
- **Plain List**: time per method without any tree relationship — useful for grouped analysis.
- **Threads tab**: thread states aggregated by name; spot ThreadPool starvation as bursts of "queued / waiting" on workers.
- **Events** (timeline mode only): GC events, contention events, file IO, web requests.

Default to Hot Spots for triage and Call Tree for deep dives.

## Comparing snapshots

`Compare snapshots` diffs by method. Useful for confirming a perf fix: take a snapshot on `main`, take another on the fix branch under the same workload, look for the green/red columns. Like dotMemory, comparison only works *within the same machine and roughly the same build* — it diffs methods by mangled signature, so renames defeat it.

## Senior-level gotchas

- **Sampling at 1 ms misses sub-millisecond methods entirely.** A method that takes 200 µs but is called 1000 times will show up; one that takes 200 µs called 5 times will not. Switch to tracing only after sampling proves the hot region.
- **Tracing distorts inlining.** A method that's normally inlined (`AggressiveInlining`) gets a real stack frame under tracing, which both inflates its time and changes register pressure in callers. Treat tracing-mode timings as relative.
- **Timeline mode on Linux uses EventPipe**, which has a smaller event vocabulary than ETW. Some categories that work on Windows (e.g. file IO breakdowns) are partial or missing on Linux. Check the snapshot's environment header to confirm what was captured.
- **NativeAOT support is limited** — the profiler relies on JIT metadata for some features. You'll get sampling stacks but lose accurate inlining info, and `await` correlation can break for trimmed state machines.
- **Snapshots embed thread names and command lines.** A snapshot from a prod box may contain hostnames, container IDs, or environment-derived secrets in process arguments. Sanitize before sharing externally.
- **GUI memory consumption explodes for large snapshots.** A 1 GB `.dtt` can need 6–8 GB of RAM to navigate. Use `--timeout` aggressively or filter to a single thread before exporting.
- **The "Self time" column is wall-clock, not CPU.** A method that did `await Task.Delay(100)` will show 100 ms of self-time in timeline mode. Sampling mode reports CPU time instead. Read the column header before drawing conclusions.
- **Profiler attachment can fail silently on locked-down hosts** (no admin, profiling API disabled). The CLI returns 0 but the snapshot is empty. Always confirm the snapshot has events before walking away from the box.
- **Don't use Tracing in CI for regression detection.** The numbers aren't stable enough across machines and JIT versions. Use Sampling for trend tracking and reserve Tracing for human-in-the-loop drill-downs. For machine-readable regression gates use [`benchmarkdotnet.md`](./benchmarkdotnet.md) instead.
- **`ConfigureAwait(false)` in app code does not change what dotTrace shows** — the timeline still reconstructs the logical flow. Don't add `ConfigureAwait` "for the profiler"; add it (or don't) for the runtime semantics described in [`async-await-performance.md`](./async-await-performance.md).
