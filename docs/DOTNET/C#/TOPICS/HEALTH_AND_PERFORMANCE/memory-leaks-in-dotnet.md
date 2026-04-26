# Memory leaks in .NET

_Targets .NET 10 / C# 14. See also: [GC pressure](./gc-pressure.md), [GC generations (gen0 / gen1 / gen2)](./gc-generations-gen0-gen1-gen2.md), [Large Object Heap (LOH)](./large-object-heap-loh.md), [LOH fragmentation](./loh-fragmentation.md), [Finalization and the finalizer queue](./finalization-and-finalizer-queue.md), [ArrayPool&lt;T&gt;](./arraypool-of-t.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [dotMemory](./profiling-dotmemory.md), [PerfView](./profiling-perfview.md)._

A **memory leak** in .NET is a *rooting* problem.

> Objects the application no longer needs remain reachable from a GC root, so the collector cannot reclaim them. Heap (especially Gen2 + LOH) grows monotonically with uptime; full collections do not free anything because the objects are still alive by the rules of the GC.

This is what people *mean* by "leak" in a managed runtime — there are no `free()` mistakes; instead something keeps a reference. The signature is unique and distinguishes it from two adjacent problems:

| Problem | Heap shape | Root |
|---|---|---|
| **Memory leak** | Heap and Gen2 grow monotonically; full GC frees little | Something is rooting objects |
| **GC pressure** | Heap stable; high Gen0 churn; high `% time in gc` | Allocation rate — see [GC pressure](./gc-pressure.md) |
| **High working set** | Heap large but stable | App genuinely needs that much memory |

The fix sets do not overlap. A leak fix is "find the root and unroot it"; pressure is "allocate less"; working set is "use less data or scale out".

## Quantitative signals

A service is leaking when, over hours of representative load:

- **`gc-heap-size` slope is positive** beyond warm-up.
- **`gen-2-size` and `loh-size` rise without coming back down** after Gen2 collections.
- **`gen-2-gc-count` rises** but freed bytes per collection trends to zero.
- **Working-set RSS tracks `gc-heap-size`** — the leak is on the managed heap, not unmanaged.
- **Eventually**: `OutOfMemoryException`, container OOM-killer, or pod restart loops.

A short rise in heap during warm-up (caches, JIT, pooled handlers populating) is normal. The diagnostic test is: at *steady-state load*, is the trend over hours flat or upward?

## Measurement recipe

1. **Establish the trend** — hours, not minutes:
   ```bash
   dotnet-counters monitor System.Runtime --process-id <pid> \
       --counters gc-heap-size,gen-2-size,loh-size,working-set
   ```
   Plot it. If the slope is zero after warm-up, there is no leak — investigate elsewhere.
2. **Two snapshots** — before and after the leak window:
   ```bash
   dotnet-dump collect --process-id <pid> -o snap-1.dmp
   # ... wait an hour under load ...
   dotnet-dump collect --process-id <pid> -o snap-2.dmp
   ```
3. **Diff what grew**:
   ```bash
   dotnet-dump analyze snap-2.dmp
   > !dumpheap -stat -gen 2
   > !dumpheap -stat -min 85000   # LOH
   > !finalizequeue                # is the finalizer thread keeping up?
   ```
   The type at the top — and especially the *delta* vs `snap-1` — is the suspect.
4. **Find what roots the suspect**:
   ```bash
   > !dumpheap -mt <method-table-of-suspect> -short
   > !gcroot <object-address>
   ```
   `!gcroot` walks back to a root: a static field, a thread-local, a strong handle, a finalizer queue. That root is your bug.
5. **Visual diff** — when the heap is dense:
   - **dotMemory**: open both snapshots, "Compare" → "New objects". The "Retained size by class" sort makes the answer obvious.
   - **PerfView**: *Heap Snapshot → Diff* of two GC heap dumps.

## Root causes (descending real-world frequency)

- **Static collections used as caches without eviction.** `private static readonly Dictionary<string, T> _cache = new();` Every distinct key adds a permanent entry. Use `MemoryCache` with `SizeLimit` and per-entry `Size`, or `IMemoryCache` with TTLs.
- **Event handler subscriptions never unsubscribed.** When `publisher.Event += subscriber.OnEvent;`, the publisher holds a reference to *the subscriber*. If the publisher outlives the subscriber, the subscriber is rooted forever — including everything it captures. Always pair `+=` with a `-=` in `Dispose`.
- **`IMemoryCache` / `MemoryCache` with no `SizeLimit`.** It will grow until the process is killed. Configure both the cache (`SizeLimit`) and every entry (`SetSize(...)`).
- **Captured closures rooting outer scope.** A lambda registered as a callback (timer, event, message subscription) captures `this` of the registering object. The host of the callback now roots the registering service. Common in DI: a transient registers a callback on a singleton, and the transient is rooted forever.
- **`HttpClient` per request without disposal.** Each instance holds a handler graph, sockets, and timer references. The `IHttpClientFactory` story exists in part for this reason.
- **`async` state machines parked forever.** A method that awaits a `Task` that never completes (no timeout, no cancellation) leaves the state machine on the heap, rooting all the locals. Plumb `CancellationToken` and per-call timeouts.
- **Background `Task` references held in a static list.** `_tasks.Add(Task.Run(...))` without removal — the static rooted the `Task`, the `Task` rooted its state machine.
- **DI scope mistakes.** A *singleton* holds a reference to a *scoped* service it received via constructor. The scoped instance is now alive for the singleton's lifetime — and so is everything it rooted. The `ASP0000` analyzer catches a subset; the rest you find via `!gcroot`.
- **Improper `IDisposable` with finalizer on managed-only state.** A finalizer promotes the object an extra generation (Gen2 minimum) and reschedules anything it references. If the type holds nothing unmanaged, the finalizer is wrong — remove it. See [finalization and the finalizer queue](./finalization-and-finalizer-queue.md).
- **`ThreadLocal<T>` without `Dispose`.** Each value is rooted by the thread it was created on; on the thread pool, that's effectively forever.
- **EF Core change-tracker accumulation.** A long-running scope (`IServiceScope` held across operations, common in console workers) tracks every entity ever queried. Use `AsNoTracking()` for reads, scope per logical operation, or call `ChangeTracker.Clear()` between batches.
- **`IDisposable` not disposed.** `using`/`await using` is the only correct pattern; manual disposal often misses an exception path. Roslyn analyzer `CA2000` flags many cases.
- **Pinned objects.** A long-lived pin on a regular SOH array prevents compaction, which fragments the heap and forces premature promotions. See [pinned objects](./pinned-objects.md).

## Diagnostic flow

```
heap rises in dotnet-counters?
    ├─ no  → not a leak; check GC pressure / working set
    └─ yes → take two dumps T+30min, T+2h
              ├─ !dumpheap -stat -gen 2 → top type X grew by N MB
              │  └─ !dumpheap -mt <X-MT> -short → pick a sample
              │     └─ !gcroot <addr> → root chain
              │        ├─ static field ........ static cache / event publisher
              │        ├─ thread local ........ thread-bound state
              │        ├─ AsyncLocal .......... captured async context
              │        ├─ finalizer queue ..... finalizable object referencing X
              │        └─ pinned ............. pinning preventing compaction
              └─ if LOH grew → !dumpheap -stat -min 85000
                 └─ usually buffer mismanagement → ArrayPool / RecyclableMemoryStream
```

## Worked example — leaking event subscription

```csharp
// Before: every NotificationListener created here is rooted forever
// because EventBus is a singleton and we never unsubscribe.
public sealed class NotificationListener
{
    private readonly byte[] _buffer = new byte[64 * 1024];

    public NotificationListener(IEventBus bus)
    {
        bus.OnMessage += OnMessage;     // bus holds delegate → holds this → holds _buffer
    }

    private void OnMessage(object? s, MessageEventArgs e) { /* ... */ }
}

// After: explicit unsubscription tied to disposal.
public sealed class NotificationListener : IDisposable
{
    private readonly IEventBus _bus;
    private readonly byte[] _buffer = new byte[64 * 1024];

    public NotificationListener(IEventBus bus)
    {
        _bus = bus;
        _bus.OnMessage += OnMessage;
    }

    private void OnMessage(object? s, MessageEventArgs e) { /* ... */ }

    public void Dispose() => _bus.OnMessage -= OnMessage;
}
```

The pattern catches almost half the event-leak cases. The other half: anonymous lambdas you can't `-=`. Solution: store the delegate in a field and subtract that exact instance, or use a weak-event abstraction.

## Worked example — bounded cache instead of static dictionary

```csharp
// Before: leaks one entry per unique key, forever.
public static class FormatterCache
{
    private static readonly Dictionary<Type, JsonSerializerOptions> _byType = new();
    public static JsonSerializerOptions For(Type t) =>
        _byType.TryGetValue(t, out var o)
            ? o
            : (_byType[t] = BuildFor(t));   // not even thread-safe; just a leak demo
}

// After: thread-safe, size-bounded, evicts under pressure.
public sealed class FormatterCache(IMemoryCache cache)
{
    public JsonSerializerOptions For(Type t) =>
        cache.GetOrCreate(t, e =>
        {
            e.Size = 1;                                    // every entry costs 1 unit
            e.SlidingExpiration = TimeSpan.FromMinutes(15);
            return BuildFor(t);
        })!;
}

// Program.cs
builder.Services.AddMemoryCache(o => o.SizeLimit = 1024); // hard ceiling
```

`SizeLimit` + per-entry `Size` is non-optional with `IMemoryCache` if you don't want a leak. `SizeLimit = null` (default) means "unbounded".

## Production canary

The cheapest detection in CI is the synthetic stress run that asserts on heap slope:

```csharp
[Fact]
public async Task ServiceDoesNotLeakOverOneHour()
{
    using var host = await StartHostAsync();
    GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
    var before = GC.GetTotalMemory(forceFullCollection: true);

    await GenerateLoadAsync(host, durationMinutes: 60);

    GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();
    var after = GC.GetTotalMemory(forceFullCollection: true);

    var growthMb = (after - before) / 1024.0 / 1024.0;
    Assert.InRange(growthMb, 0, 50); // tolerance for caches / JIT
}
```

It will not catch every leak but will catch the obvious ones (events, static caches, list-of-tasks) that produce dozens of MB per hour.

**Senior-level gotchas:**
- **Most "memory leaks" are bounded caches that aren't bounded.** Configure `SizeLimit` and per-entry `Size`, or use `IMemoryCache.Compact`. Unbounded `Dictionary<,>` with user-controlled keys is a textbook DoS vector too.
- **`WeakReference<T>` is not a substitute for unsubscription.** It works for caches that should not extend lifetime, but events still root by default — the canonical fix is `-=`, not weak references everywhere.
- **`AsNoTracking` does not reset the change tracker for entities you already loaded.** Apply it on the query, or call `ChangeTracker.Clear()` to reset.
- **The finalizer queue is itself a leak vector.** A backed-up finalizer thread (because one finalizer blocks) keeps every queued object — and everything they reference — alive across collections. `!finalizequeue` shows the depth; if it's nonzero, find the slow finalizer.
- **Static + scoped is undetectable from a single dump.** A singleton holding a scoped service rooted at startup looks indistinguishable from a legitimate cache. The diff between two dumps is the test: did it grow?
- **`GC.GetTotalMemory(true)` forces a full collection** — useful in tests, never in production.
- **`GCMemoryInfo` (`.NET 7+`) gives you the GC's view**: `MemoryLoadBytes`, `HighMemoryLoadThresholdBytes`, fragmented bytes. More accurate than `Process.WorkingSet64` for managed-heap diagnosis.
- **`ConditionalWeakTable<TKey, TValue>`** is the right tool for "extension state on third-party objects" — it doesn't prevent the key from being collected and removes the entry when it is.
- **Dumps from containers** require the matching .NET runtime version and `dotnet-dump analyze` running on a compatible host (or `WinDbg` with the right SOS). Symbol resolution often fails silently — `loadby sos coreclr` and check that types resolve before believing the output.
- **`!gcroot` can be slow** on large heaps (gigabytes) and may report multiple roots. The first one printed is enough to identify the bug; you don't need them all.
- **Process-private dictionaries shared across `AppDomain`s** — irrelevant on .NET (no AppDomains) but a real concern when porting from .NET Framework.
- **A sudden jump in heap is rarely a leak** — it's usually a load spike or a cache being populated. Real leaks have *slope*. Don't diagnose from a single noisy minute.
