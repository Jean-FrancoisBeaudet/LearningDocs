# dotnet-dump

_Targets .NET 10 / C# 14, `dotnet-dump` 9.x. See also: [EventPipe](./eventpipe.md), [dotnet-trace](./dotnet-trace.md), [dotnet-counters](./dotnet-counters.md), [PerfView](./profiling-perfview.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [memory leaks in .NET](./memory-leaks-in-dotnet.md), [GC pressure](./gc-pressure.md), [finalization and the finalizer queue](./finalization-and-finalizer-queue.md), [thread contention](./thread-contention.md)._

`dotnet-dump` is the **cross-platform CLI for capturing and post-mortem-analysing process memory dumps** of a .NET application. It collects a snapshot — full or heap-only — of the process's virtual memory, threads, and CLR state, and ships an interactive REPL (the SOS commands) for walking that snapshot offline. On Linux, where Windows-native tools (WinDbg, ProcDump) don't apply, it is the canonical way to triage a hang, an OOM, or a heap leak.

It is **not** a profiler, **not** a counter monitor, and **not** something you run continuously. A dump is a one-shot, expensive artefact. Capture it, analyse it, archive it. For live observability use [`dotnet-counters`](./dotnet-counters.md); for event tracing use [`dotnet-trace`](./dotnet-trace.md).

## Where it fits

| Strength | Weakness |
|---|---|
| Cross-platform; only managed dump tool on Linux | Capture suspends the process for 1–10s on multi-GB heaps |
| Full SOS REPL: `clrstack`, `dumpheap`, `gcroot`, `eeheap` | UX is text-only; retention paths take patience |
| Captures threads, locks, exception state — perfect for hangs | Multi-GB dumps are expensive to copy/store |
| No admin in most cases; uses diagnostic IPC | Cannot dump a crashed process unless OS already collected one |
| Works in containers (with shared `/tmp`) | Not interchangeable with WinDbg `.dmp` for post-mortem on Windows (different format on Linux) |

Reach for `dotnet-dump` when:

1. The process is **hung** — every other tool is for live data and a hung process emits no events.
2. You suspect a **managed memory leak** and need to walk the live heap to find what's rooting the offenders.
3. The pod is in `CrashLoopBackOff` and you need a snapshot before the OOM kill.
4. You're on Linux and there is no native equivalent.

## Install and inventory

```bash
dotnet tool install -g dotnet-dump

# Find candidate PIDs (same diagnostic-IPC mechanism as the rest of the suite).
dotnet-dump ps
#   12345  Orders.Api          /app/Orders.Api
```

`dotnet-dump ps` is identical to `dotnet-counters ps` / `dotnet-trace ps` — they all walk the diagnostic socket directory.

## Capturing a dump

```bash
# Default: --type Full (complete process memory; large file, full visibility).
dotnet-dump collect -p 12345 --output /diag/orders.dmp

# Heap-only — managed heap + handles + threads, no native modules.
dotnet-dump collect -p 12345 --type Heap --output /diag/orders.heap.dmp

# Mini — just thread state + stacks; tiny file, useful for hang triage.
dotnet-dump collect -p 12345 --type Mini --output /diag/orders.mini.dmp

# Triage — mini + selected runtime info.
dotnet-dump collect -p 12345 --type Triage --output /diag/orders.triage.dmp
```

| Type | Includes | Typical size | Use for |
|---|---|---|---|
| `Full` (default) | Everything: native + managed heap + threads + modules | 1× working set (multi-GB easily) | Hard cases; first-pass on Linux |
| `Heap` | Managed heap, handles, threads, GC info (no native heap) | ~managed heap size | Memory leaks, retention |
| `Mini` | Thread stacks + module list | < 100 MB | Hangs, deadlocks |
| `Triage` | Mini + runtime version + key state | < 100 MB | Quick triage when sharing externally (PII-sensitive) |

The capture **suspends the process** while the runtime walks its state. On a 4 GB heap that's typically 1–3 seconds; on 32 GB it's noticeable. Plan accordingly: dump on a canary pod, dump during a maintenance window, or accept that the SLO will see a blip.

## Analysing a dump

```bash
dotnet-dump analyze /diag/orders.dmp

# Inside the REPL, the SOS commands are available without prefix.
> clrthreads
> setthread 14
> clrstack -all
> dumpheap -stat
> dumpheap -mt 00007f8a8c1234 -short
> gcroot 00007f8a90abcdef
> eeheap -gc
> dumpasync
> threadpool
> exit
```

The `analyze` command spins up a host process, loads the dump, attaches the SOS extension, and drops you into a REPL. Commands are the standard SOS set — knowledge transfers from WinDbg and `lldb`+SOS sessions.

## Triage flow: hung process

A pod is unresponsive; counters show no progress; nothing in logs.

```bash
dotnet-dump collect -p 1 --type Mini --output /diag/hang.dmp
dotnet-dump analyze /diag/hang.dmp
```

```
> clrthreads
   ID OSID ThreadOBJ State GC Mode    Lock Count APT Exception
   ...
> ~*e !clrstack            # WinDbg-style: every thread, managed stack
> threadpool                # is the threadpool starved?
> dumpasync                 # async state machines waiting (.NET 5+)
> syncblk                   # sync blocks held; -1 in the SyncBlock column = lock contention
```

Patterns to look for:

- **Many threads in `Monitor.Wait` / `WaitOne` / `Task.Wait` / `SemaphoreSlim.Wait`** → classic deadlock or sync-over-async. Find the lock owner via `syncblk` and `gcroot` the held object.
- **Threadpool exhausted** (`threadpool` shows max workers, all busy) → blocking sync work on async paths. See [thread contention](./thread-contention.md).
- **One thread in GC, all others suspended** → a long Gen2 pause. Capture a [`dotnet-trace --profile gc-verbose`](./dotnet-trace.md) instead — a dump tells you *that* GC is happening; a trace tells you *why*.
- **`dumpasync` shows thousands of pending state machines** → unawaited tasks accumulating, possibly with a stuck consumer. See [async/await performance](./async-await-performance.md).

## Triage flow: managed memory leak

Working set climbs steadily; counters confirm `gen-2-gc-count` and `gc-heap-size` rising; OOM expected.

```bash
dotnet-dump collect -p 1 --type Heap --output /diag/leak.dmp
dotnet-dump analyze /diag/leak.dmp
```

```
> dumpheap -stat              # one row per type, sorted by total bytes (last column)
              MT    Count    TotalSize Class Name
00007f88c1234     1023341    245601984 System.Byte[]
00007f88c5678      812000    155264000 Acme.Orders.OrderEntity
...

> dumpheap -mt 00007f88c5678 -short    # addresses of every OrderEntity instance
00007f8a90abcdef
00007f8a90abcdf0
...

> gcroot 00007f8a90abcdef     # what's keeping it alive?
HandleTable:
  ...
-> 00007f8a8a... System.Diagnostics.Tracing.EventListener
   -> 00007f8a8b... Acme.Orders.OrderTelemetry
      -> 00007f8a8c... System.Collections.Generic.List<Acme.Orders.OrderEntity>
         -> 00007f8a90abcdef Acme.Orders.OrderEntity
```

The `gcroot` chain is the answer: a static `EventListener` is holding a `List<OrderEntity>` indefinitely. The fix lives in code; the dump just told you *who's holding the bag*.

For deeper analysis on small heaps, **dotMemory** has the better retention-path UX. `dotnet-dump` is the right tool when dotMemory isn't available — i.e. on a Linux server, in a container, or when the org doesn't license JetBrains. See [dotMemory](./profiling-dotmemory.md).

## Compared to `dotnet-gcdump`

```bash
# dotnet-gcdump: managed-heap-only graph, much smaller, opens in PerfView / VS.
dotnet tool install -g dotnet-gcdump
dotnet-gcdump collect -p 12345 --output /diag/orders.gcdump
```

| Tool | Captures | File size | Analyser | Triggers GC? |
|---|---|---|---|---|
| `dotnet-dump --type Full` | Everything | Largest (~ working set) | `dotnet-dump analyze`, WinDbg, dotMemory (Windows) | No |
| `dotnet-dump --type Heap` | Managed heap + threads | Medium | `dotnet-dump analyze` | No |
| `dotnet-gcdump` | Managed object graph only | Smallest (~ heap × 0.1) | PerfView, VS, dotMemory | **Yes** (forced Gen2) |

`dotnet-gcdump` is the right first move for "what's on the heap?" — it's smaller, opens in PerfView's heap views, and works from Linux. Use `dotnet-dump --type Heap` when you also need thread stacks (hung process + leak), or when you need the full SOS surface for navigation.

Note `dotnet-gcdump` triggers a forced Gen2 collection before snapshotting — the heap you see is the *post-collection* state, which is what you usually want (no `Gen0` garbage). But the GC itself is observable in the process and adds latency.

## Containers and Kubernetes

```bash
# Capture inside the pod.
kubectl exec -it orders-7df-abc12 -- \
  dotnet-dump collect -p 1 --output /tmp/orders.dmp

# Copy out before the pod restarts.
kubectl cp orders-7df-abc12:/tmp/orders.dmp ./orders.dmp

# Analyse locally (host needs same .NET runtime version as the captured process).
dotnet-dump analyze ./orders.dmp
```

**Key constraints:**

- Filesystem space — a `Full` dump is sized to the working set. Pods with `emptyDir` of 1 GB cannot hold a 4 GB dump. Mount a sized PVC or stream out via `kubectl cp` while writing (not always reliable).
- Runtime version — `dotnet-dump analyze` needs SOS for the *target's* runtime version. The tool downloads it automatically on first use; ensure the analyser host has internet access or pre-seed the cache.
- Restart racing — capture the dump *before* the orchestrator kills the pod for OOM/liveness. For OOM-prone services, deploy a `dotnet-monitor` sidecar with a memory-threshold dump trigger.

## Production safety

| Risk | Mitigation |
|---|---|
| Suspends the process | Run on a canary pod or during a maintenance window; a 4 GB dump on a 16-core box typically pauses 1–3 s |
| Large file → disk pressure | Use `--type Heap` or `--type Mini` first; full dumps only when needed |
| Sensitive data in dump | A full dump contains *every byte of memory* — including secrets, tokens, PII. Treat dumps as production data, encrypt at rest, redact before sharing |
| OOM during capture | Capture writes the dump to disk via the diagnostic IPC; the writing process is the target itself plus a helper. Dump destinations should not be the same volume as application data |
| Production pod can't run dotnet-dump | Pre-bake the tool into the image, or use `dotnet-monitor` sidecar |

## Senior-level gotchas

- **Capturing suspends the process. Period.** No "low-overhead dump" mode exists. Do not script it as a routine collection. The runtime must walk thread state and (for non-Mini) the heap with stop-the-world semantics. On large heaps this is multiple seconds.
- **`dotnet-gcdump` *triggers a Gen2*.** It calls `GC.Collect()` first to give a clean post-collection picture. The collection itself is a managed-side event with full pause cost. Don't confuse "the gcdump is small" with "gcdump is free".
- **Cross-platform dumps are not interchangeable.** A Linux `Full` dump is an `ELF coredump`; a Windows `Full` dump is a Microsoft minidump. `dotnet-dump analyze` reads both, but WinDbg only reads the Windows format and lldb only the Linux one.
- **The analyser needs the matching runtime SOS.** Capture on .NET 8.0.5; analyse on a host with .NET 8.0.0; SOS may be slightly out-of-step and some commands misbehave. The tool downloads correct SOS on first analyse, but in air-gapped environments pre-stage the matching runtime.
- **`dumpheap -stat` sorts by total bytes by default — including dead objects until next GC.** Run `dumpheap -live -stat` for live-only counts (slower but accurate). Inflated `Byte[]` counts often vanish under `-live`.
- **`gcroot` may report no roots when the object is actually rooted.** The most common reason is **rooting via async state machine boxed on the heap** — `gcroot` doesn't always traverse certain compiler-generated structures cleanly. If a leak's root chain is empty but the object survives Gen2, recapture and try `dumpasync` and `dumpheap -mt <AsyncStateMachineBox>` to find async-rooted graphs.
- **`dumpasync` is the killer command for async leaks.** It walks the heap for `IAsyncStateMachine` boxes, prints what each is waiting on, and shows the chain of pending continuations. Often more useful than `clrstack` for `await`-heavy services.
- **Static fields are roots — but so are `EventHandler` subscriptions.** A common leak pattern is `globalEventSource.Changed += target.Handler` without unsubscribe — `target` is now rooted from the source's invocation list. `gcroot` shows the chain via `Action`/`MulticastDelegate` instances; learn to recognise it.
- **`dotnet-dump` cannot capture a crashed process.** It needs the process to be alive and IPC-responsive. For crashed processes use OS-level core-dump capture (`coredumpctl` on Linux, Windows Error Reporting / `procdump -e` on Windows) and then load with `dotnet-dump analyze`.
- **An OOM-killed pod produces no dump.** The OOM killer SIGKILLs the process — no diagnostic IPC, no graceful shutdown. To dump *before* OOM, configure a `dotnet-monitor` rule on `gc-heap-size > threshold`, or set process limits low enough that you hit a managed `OutOfMemoryException` first.
- **Mini-dumps cannot inspect the managed heap.** No `dumpheap`. Useful only for thread/stack analysis. Pick the right `--type` upfront — recapturing a multi-GB Full dump after realising Mini was too small is expensive.
- **Symbol resolution for native frames depends on `_NT_SYMBOL_PATH` (Windows) or having the matching `.so` files (Linux).** A managed-only investigation usually doesn't need them, but if you find yourself in `coreclr!` frames, set the symbol path before opening.
- **The tool's REPL is single-threaded and synchronous.** A `dumpheap` on a 30 GB heap can take minutes and there's no progress indicator. Patience or, better, narrower commands (`-mt <table>`, `-min <bytes>`).
- **Treat a dump as confidential.** A Full dump contains every secret in process memory at capture time — JWTs, DB connection strings, recently-decrypted payloads. Encrypt at rest, restrict access, and shred when the investigation is done. Dumps that "leaked the JWT signing key" because they were attached to a Slack message are a real and recurring incident.
- **For continuous heap visibility, prefer counters first.** [`dotnet-counters`](./dotnet-counters.md) `gc-heap-size`, `alloc-rate`, `gen-2-gc-count` will tell you *whether* there's a leak. The dump tells you *what's leaking*. Don't dump first if a counter run hasn't confirmed the symptom.
