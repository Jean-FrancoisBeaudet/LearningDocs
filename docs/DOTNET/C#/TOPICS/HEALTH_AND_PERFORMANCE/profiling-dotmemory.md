# dotMemory

_Targets .NET 10 / C# 14. See also: [dotTrace](./profiling-dottrace.md), [PerfView](./profiling-perfview.md), [Visual Studio Diagnostic Tools](./profiling-visual-studio-diagnostic-tools.md), [Memory leaks in .NET](./memory-leaks-in-dotnet.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [LOH fragmentation](./loh-fragmentation.md), [Finalization and the finalizer queue](./finalization-and-finalizer-queue.md), [dotnet-dump](./dotnet-dump.md)._

JetBrains' managed-memory profiler. Its core workflow is **snapshot, exercise, snapshot, diff** over a live process or a memory dump. The strength is the retention-path UI: given any object instance, dotMemory walks back through the live graph to the GC root holding it, with the field names along the way. That's what you want when "the heap keeps growing" and you need to point at a specific static, event, or cache responsible for the growth.

## Where it fits versus alternatives

| Tool | Strength | Weakness |
|---|---|---|
| **dotMemory** | Retention paths, snapshot diff, live attach, dump load | Windows GUI; paid (free tier = JetBrains account / 30-day) |
| **PerfView** | ETW depth, GC stats, allocation tick provider, scriptable, free | Heap walk UX is rough; no diff workflow |
| **VS Diagnostic Tools — Memory Usage** | Built-in, free, similar diff model | Smaller feature set; weaker retention navigation |
| **dotnet-dump + SOS** | Free, Linux-friendly, post-mortem | No GUI; `!gcroot` is one address at a time |

Reach for dotMemory when the question is **"what is rooting these instances?"** and the answer needs to fit in one screenshot you can share with the team.

## Attach modes

- **Standalone GUI**: launch dotMemory, attach to a running PID, or "Profile new process".
- **Rider / ReSharper plugin**: same engine, integrated panel.
- **Command-line** (`dotMemory.exe` on Windows, `dotmemory` on macOS/Linux): the CI-friendly surface.
- **Load existing dump**: open a `.dmp` from `dotnet-dump collect` or Task Manager's "Create Dump File". You lose live tracking but you get the same retention UI offline.

```bash
# Headless capture: attach, take three snapshots one minute apart, save and exit.
dotMemory.exe attach 12345 ^
  --temp-dir=C:\dotmem ^
  --save-to-file=C:\dotmem\workload.dmw ^
  --trigger-on-activation ^
  --trigger-timer=60
```

The `.dmw` is dotMemory's snapshot bundle and is reopened in the GUI later. For one-off captures use `dotMemory.exe get-snapshot <pid> --save-to-file=...`.

## Snapshot mechanics

A snapshot:

1. **Suspends the target** briefly via the profiling API.
2. **Forces a blocking Gen2 GC** so all dead objects are collected first.
3. **Walks the entire managed heap**, recording type, size, generation, and reference graph.
4. **Resumes the target.**

What you get: every live managed object, generation/LOH/POH placement, every reference field, and the set of GC roots (statics, finalizer queue, stack pinning, handles, sync blocks). Native memory and unmanaged buffers held by `SafeHandle`/`IntPtr` are **not** captured — those are PerfView / WinDbg territory.

Because step 2 is a blocking Gen2, snapshots are intrusive. Don't take them on a live SLO-bound endpoint without a window.

## The four core analyses

| View | Question it answers |
|---|---|
| **Group by → Dominators** | "What single object would free the most memory if it disappeared?" |
| **Retention path** (right-click any instance) | "Why is *this* still alive? Walk me to the root." |
| **Snapshot diff** (`Compare with…`) | "What new objects survived between A and B?" |
| **Inspections** | Pre-baked detectors: duplicate strings, sparse arrays, large objects, finalizable instances, event-handler leaks |

The Inspections panel is where you start when you have *no* hypothesis yet — it surfaces the usual leak shapes without you having to know which type to look at.

## Workflow: classic event-handler leak

1. **Warm up** the process (let JIT settle, caches fill).
2. **Snapshot A** — baseline.
3. **Exercise the suspect scenario N times** (e.g. 100 page navigations, 1000 imports). N matters: noise floor is around 50 instances of common BCL types per snapshot, so make the leak count visibly larger.
4. **Wait for Gen2 settle** — issue an idle period or call `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();` from a debug endpoint.
5. **Snapshot B**.
6. **Compare A → B**, sort the diff by `New objects`, filter out framework noise, find your domain types at the top.
7. **Right-click the suspect type → Open retention path**. Look for `EventHandler` chains, `static Dictionary<,>`, `ConditionalWeakTable`, or `ConcurrentDictionary` keyed by something request-scoped.
8. **Fix and re-run.** A fixed leak shows zero (or small constant) new instances after the same N iterations.

## Allocation tracking mode

Different from snapshots: dotMemory hooks the runtime allocation events and records **a stack trace per allocation**. Overhead is significant (often 2–5× wall-clock), so it's a development-time tool. Use it when retention paths can't tell you *who keeps allocating* — for example, an LOH growth driven by a third-party serializer.

Toggle "Collect allocation data" on attach. The result is a flame-style "Allocations" tab where you can sort callers by total bytes / objects.

## Reading dumps without the original process

```bash
dotnet-dump collect --process-id 12345 --output app.dmp
# then in dotMemory: File → Open → app.dmp
```

dotMemory will attempt to resolve symbols from the .NET runtime metadata embedded in the dump, but third-party PDBs need to be reachable (set `_NT_SYMBOL_PATH` or use the dotMemory "Symbol servers" setting). Without symbols you still get type names — they live in the metadata — but stack frames in allocation data become offsets.

## Senior-level gotchas

- **Snapshots force a full Gen2 collection.** You will pause the process, often for hundreds of ms on a multi-GB heap. Never trigger one inside a request handler endpoint and call it "diagnostics".
- **Finalizable types look rooted by the finalizer queue.** That's not a leak per se — it just means the object is waiting for finalization. Cross-reference with [`finalization-and-finalizer-queue.md`](./finalization-and-finalizer-queue.md). If retention path stops at "F-Reachable Queue", the next snapshot after a forced finalizer pump will clear them.
- **`WeakReference` and `ConditionalWeakTable` graphs do not appear in dominators**, because by definition they don't keep targets alive. If you suspect a leak through one of these, check it manually — the type still shows up in instance counts, just without a retention edge.
- **Comparing snapshots from different process runs is meaningless.** Object addresses, JIT codegen, and root layout all change. The diff feature only does what you want within a single attach session.
- **Interior pointers** (e.g. spans of strings, ref-fields, `fixed` blocks) can produce surprising "boxed value type" entries in the dominator tree. They're usually a UI artifact, not a real allocation — confirm with the inspections panel before chasing them.
- **String duplicates are usually a sign of missing interning**, not a leak. Inspections will flag them. The fix is `string.Intern` (rarely) or `FrozenDictionary`/`SearchValues` for keys, or upstream caching — not "deduplicating" with `HashSet<string>` reactively.
- **`ArrayPool<T>` rentals look like LOH leaks** if you forgot to return them. The retention path lands on the static `ArrayPool` bucket — that's the tell. Same shape applies to `RecyclableMemoryStream`.
- **Don't profile Release self-contained NativeAOT builds with dotMemory** — the profiling API surface it relies on isn't available. Use `dotnet-gcdump` or PerfView with EventPipe instead.
- **dotMemory's "Used memory" includes overhead** (bookkeeping, profiling agent allocations). Compare against `GC.GetTotalMemory(false)` from a `/diag` endpoint when you need a number to put in a ticket.
- **Snapshots can be huge.** A 4 GB heap with deep graphs can produce a 6–8 GB `.dmw`. Plan disk space; the GUI also needs ~2× the file size in working memory to navigate freely.
