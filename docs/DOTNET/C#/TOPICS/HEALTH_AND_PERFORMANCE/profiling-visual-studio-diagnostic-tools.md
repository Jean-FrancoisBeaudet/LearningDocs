# Visual Studio Diagnostic Tools

_Targets .NET 10 / C# 14, Visual Studio 17.x. See also: [dotTrace](./profiling-dottrace.md), [dotMemory](./profiling-dotmemory.md), [PerfView](./profiling-perfview.md), [BenchmarkDotNet](./benchmarkdotnet.md), [dotnet-counters](./dotnet-counters.md), [async/await performance](./async-await-performance.md)._

The lowest-friction profiler — it ships with Visual Studio. Two distinct surfaces share the same engine:

- **Diagnostic Tools window** (auto-opened during F5 debug). Live mini-charts for CPU/Memory/Events plus inline PerfTips in the editor. Convenient, but every measurement includes debugger overhead.
- **Performance Profiler** (Alt+F2 or Debug → Performance Profiler). Runs *without* the debugger attached, uses the proper sampling/instrumentation engines. This is the surface to use when you want timings you can quote.

If your only goal is "show me roughly where the CPU goes in this scenario", VS Diagnostic Tools answers it without installing anything. For deeper questions — ETW-level GC analysis, retention paths across snapshots, async-correlated stacks — graduate to PerfView, dotMemory, or dotTrace.

## Tools available (Performance Profiler)

| Tool | What it does | Overhead | Notes |
|---|---|---|---|
| **CPU Usage** | Sampling profiler with caller/callee, hot path, flame graph | Low | The default first stop |
| **Memory Usage** | Snapshot + diff over the managed heap | Low (between snapshots) | "Paths to Root" replaces dotMemory's retention path |
| **.NET Allocations** | Records every allocation with stack | Very high (5–20×) | Synthetic workloads only |
| **.NET Async** | `Task` lifetimes, continuations, awaiter waits | Moderate | The right view for "this `await` never completed" |
| **.NET Counters** | Live `EventCounters` charts | Low | Same data as `dotnet-counters` but inline |
| **Database** | EF Core / ADO.NET query timeline | Low–moderate | Captures connection, command text, duration |
| **File I/O** | File reads/writes by path | Low | Useful for startup analysis |
| **Instrumentation** (legacy) | Method call counts | High | Enterprise SKU only; rarely needed since `.NET Allocations` covers most cases |
| **GPU Usage** | DirectX/OpenGL frame timing | Low | Game / WPF graphics — out of scope for typical service work |

The set is deliberately overlapping with PerfView and dotTrace; the value of the VS tools is **proximity** — three clicks from the source you're editing to a measurement of the method you just changed.

## CPU Usage

Sampling at 1 kHz by default. Output is the same shape as dotTrace sampling: call tree (top-down), hot path, caller/callee, and (since 17.8+) a flame graph.

```
Performance Profiler → CPU Usage → Start
  → exercise scenario in the running app
  → Stop Collection
  → Top Functions ranked by Self/Total %
```

The "Open Details" link shows the function source inline with sample-attributed hit counts in the gutter. That's the closest VS gets to PerfView's `Source View` and is the right level for "which line in this method is hot".

## Memory Usage

Two-step workflow:

1. Click **Take Snapshot**. VS forces a Gen2 collection and records the surviving heap.
2. Run the scenario. Click **Take Snapshot** again.
3. The snapshot row shows `Objects (Diff)` and `Heap Size (Diff)`. Click the diff to drill in.
4. **Paths to Root** on any type shows the chain of references holding instances alive.

Smaller feature surface than dotMemory — no allocation tracking, no inspections panel, no dump-file load — but the diff workflow is the same and the data is enough to find typical leaks.

## .NET Allocations

This is an **instrumented** tool (not sampled). Every `newobj` is intercepted and a stack trace is recorded. Useful when:

- You suspect a specific code path allocates excessively and want exact numbers.
- You need to attribute allocations smaller than PerfView's 100 KB sampling threshold.

Cost: 5–20× wall-clock slowdown, very large output files. Run on a synthetic, isolated workload — not full integration tests.

## .NET Async

Visualises each `Task`'s lifecycle: created → started → awaited → continuation enqueued → completed. The view is a horizontal "swimlane per Task" timeline. Highlights:

- **Tasks that never completed** — leaks, forgotten awaits, deadlocks.
- **Long awaiter waits** — your continuation was queued for X ms before resuming, often a sign of ThreadPool starvation (cross-reference [`thread-contention.md`](./thread-contention.md)).
- **`async void` chains** — they show up but lack continuation context, which makes the bug visible.

## Diagnostic Tools window during F5

Free with every debug session. Three panels:

- **PerfTips**: an inline `[Elapsed: 142 ms]` next to the line you just stepped over. Best at orienting you between breakpoints; **not** an authoritative timing — debugger overhead, PDB symbol lookup, and breakpoint serialization all inflate it.
- **Events** tab: every exception thrown, every breakpoint hit, every IntelliTrace event. The exception stream is the killer feature: spot first-chance exceptions you didn't know were happening.
- **CPU & Memory mini-graphs**: live, low-resolution. Useful for "is something happening?", not for measurement.

Treat any number from F5-debug as **directional only**. Quote nothing from this surface in a perf bug — re-run under Performance Profiler first.

## When to graduate

| Situation | Reach for |
|---|---|
| Need ETW-level CLR depth (GC stats, JIT events, contention sources) | PerfView |
| Need retention-path navigation across many types | dotMemory |
| Need timeline correlation across threads + IO + locks | dotTrace |
| Need Linux profiling | `dotnet-trace` + PerfView, or dotTrace for Linux |
| Need NativeAOT-friendly profiling | `dotnet-trace` / `perf` (Linux) |
| Need scriptable / CI integration | PerfView CLI, `dotnet-trace`, BenchmarkDotNet |

VS Diagnostic Tools covers the 80% case for in-IDE work. Beyond that, the dedicated tools are worth the install.

## Senior-level gotchas

- **Debugger overhead is significant and variable.** Numbers from the F5 Diagnostic Tools window are useful only as deltas within the same session. Never quote them across builds, machines, or to engineers outside the room.
- **Performance Profiler is the answer for actual timings.** Alt+F2, pick a tool, run without the debugger. The two surfaces look similar but produce different-quality data.
- **CPU sampling at 1 kHz is too coarse for hot loops.** A method taking 200 µs called 50 times will register but with high variance. For sub-millisecond regions, switch to BenchmarkDotNet — sampling is the wrong approximation.
- **`.NET Allocations` distorts timing**. Don't read the wall-clock from this tool — it's instrumented. Use it for *allocation counts* and *stacks* only; pair with `MemoryDiagnoser` for cleaner per-method numbers.
- **Memory snapshots take a Gen2 collection.** Same caveat as dotMemory — intrusive on a live service. Don't let "but it's just Visual Studio" make you think it's free.
- **`Paths to Root` truncates at long graphs.** A retention path 30+ frames deep collapses with "..." in the middle. Switch to dotMemory if you need the full chain.
- **The `.NET Async` tool relies on the Task instrumentation provider** which is sometimes disabled by hosting configs (e.g. `DOTNET_TieredCompilation=0` flips behavior; some AOT scenarios). If the swimlane view is empty, that's the cause.
- **Diagnostic sessions are saved as `.diagsession`** which PerfView can also open. Useful when a colleague captures in VS and you want to drill in with PerfView's grouping language.
- **SKU matters for some tools.** Legacy Instrumentation needs Enterprise; CPU, Memory, .NET Allocations, .NET Async all work in Community/Professional. If a tool greys out, the SKU is the most likely reason.
- **PerfTips elapsed time includes the debugger** — every step-over runs through the debugger's source notification path. Numbers can be 5–10× the real cost. They're a smell test, not a measurement.
- **VS does not have an equivalent of dotTrace's timeline view** — the "Events" tab is the closest analogue, but without the per-thread state lanes. If a request's gap is the question, this is not the tool.
