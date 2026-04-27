# Thread synchronization — decision tree

_Targets .NET 10 / C# 14. Index into: [`lock` & `SemaphoreSlim`](./lock-statement-and-semaphoreslim.md) · [`Semaphore`, `Mutex`, `Monitor`](./semaphore-mutex-monitor.md) · [`ManualResetEvent` & `AutoResetEvent`](./manualresetevent-and-autoresetevent.md) · [`volatile` & `Interlocked`](./volatile-and-interlocked.md)._

Pick the cheapest primitive that actually satisfies the requirement. Asking the questions in this order keeps you from over-reaching for a kernel object when a CAS would do, or for a `lock` when an `await` will be involved.

1. **Boundary** — in-process, cross-process, cross-machine.
2. **What you're coordinating** — a single value, a critical section, a signal/event, a pipeline, an N-way rendezvous, a one-time init.
3. **Sync or async** — does the critical section span an `await`?
4. **Modifiers** — reentrancy, condition wait, timeout, cancellation, fairness, reader/writer split.

## Master tree

```
What scope are you coordinating across?
│
├── Cross-machine (different hosts)
│   └── Distributed lock (Redis Redlock, ZooKeeper, Azure Blob lease,
│       SQL row lock, Cosmos DB pessimistic concurrency, etcd)
│       — none of the in-box primitives below cross hosts.
│
├── Cross-process (same machine)
│   ├── Mutual exclusion (one process at a time) ──────► named `Mutex`
│   │       (handle `AbandonedMutexException`; thread-affine; sync-only)
│   ├── Bounded concurrency (N processes at a time) ──► named `Semaphore`
│   │       (kernel; no `WaitAsync`; `SemaphoreFullException` on over-release)
│   ├── Signal "go" to other processes ────────────────► named `EventWaitHandle`
│   │       (`EventResetMode.ManualReset` for broadcast, `AutoReset` for one)
│   └── Single-instance app guard ─────────────────────► named `Mutex`
│           (use `Global\<Vendor>.<App>.<Guid>`; pick session vs global deliberately)
│
└── In-process (single OS process)
    │
    ├── Coordinating a SINGLE value (one int/long/ref, no compound invariant)
    │   │
    │   ├── Plain read/write of a flag, single writer ──► `volatile` field
    │   │       (acquire/release per access; no compound atomicity)
    │   ├── 64-bit read/write (`long`/`double`) ────────► `Volatile.Read/Write(ref x)`
    │   │       (`volatile long` is rejected by the compiler)
    │   ├── Increment / add / counter ──────────────────► `Interlocked.Increment` / `Add`
    │   │       (use the value it RETURNS, not a follow-up read)
    │   ├── Atomic swap of a reference / value ─────────► `Interlocked.Exchange`
    │   ├── Conditional update ("set if equals") ───────► `Interlocked.CompareExchange`
    │   │       (foundation of every lock-free structure; add backoff under contention)
    │   ├── Lazy single-instance assignment ────────────► `Interlocked.CompareExchange(ref x, new, null)`
    │   │       (cheap construction; if expensive, use `Lazy<T>` / `LazyInitializer`)
    │   └── Need a memory fence with no read/write ─────► `Interlocked.MemoryBarrier()`
    │           (`MemoryBarrierProcessWide()` for RCU-style code on .NET 6+)
    │
    ├── Coordinating COMPOUND STATE (a critical section guarding multiple fields)
    │   │
    │   ├── Does the critical section cross an `await`?
    │   │   │
    │   │   ├── YES (async) ─► `SemaphoreSlim(1, 1)` + `WaitAsync(ct)` in `try/finally`
    │   │   │      ├── Bounded concurrency (N permits) ► `SemaphoreSlim(N, N)`
    │   │   │      ├── Need timeout ───────────────────► `WaitAsync(TimeSpan, ct)` (returns bool)
    │   │   │      └── Reentrant ──────────────────────► RESTRUCTURE — there is no async reentrant lock in BCL
    │   │   │              (third-party: `Nito.AsyncEx.AsyncReaderWriterLock`,
    │   │   │               `AsyncLock`; or track ownership via `AsyncLocal<bool>`)
    │   │   │
    │   │   └── NO (sync)
    │   │       │
    │   │       ├── .NET 9+ and you control the field type ► `System.Threading.Lock`
    │   │       │       (~2–3× faster uncontended; type-safe; reentrant; thread-affine.
    │   │       │        Does NOT support `Wait`/`Pulse`.)
    │   │       │
    │   │       ├── Need a timeout on acquisition ───────► `Monitor.TryEnter(obj, TimeSpan)` in try/finally
    │   │       │
    │   │       ├── Need a condition variable ("wait
    │   │       │   while holding the lock until X") ────► `Monitor.Wait` / `Monitor.Pulse[All]`
    │   │       │       (lock target MUST be a plain `object`, not `Lock`;
    │   │       │        always `while (!cond) Monitor.Wait(gate)` — never `if`)
    │   │       │
    │   │       ├── Many readers, rare writer, profiled
    │   │       │   read-side as the bottleneck ────────► `ReaderWriterLockSlim`
    │   │       │       (benchmark first — `lock` often wins because it has zero
    │   │       │        bookkeeping; supports recursion only with `LockRecursionPolicy.SupportsRecursion`,
    │   │       │        which has its own performance hit)
    │   │       │
    │   │       └── Otherwise (default) ────────────────► `lock (obj)` over `Monitor`
    │   │               (reentrant; thread-affine; no cancellation; fastest path
    │   │                pre-.NET 9; on .NET 9+ prefer the `Lock` type above)
    │   │
    │   └── Multiple processes / machines? Go back up to the cross-process branch.
    │
    ├── SIGNALING — "thread A tells thread B to proceed" (no shared mutable state to guard)
    │   │
    │   ├── Sync waiters
    │   │   ├── Broadcast: open a gate, ALL waiters pass ► `ManualResetEvent` (kernel)
    │   │   │                                            or `ManualResetEventSlim` (in-process, spins first)
    │   │   ├── One-at-a-time turnstile (single waiter) ► `AutoResetEvent`
    │   │   │       (signals are NOT counted — `Set; Set` before any wait releases ONE)
    │   │   └── Counting "N permits available" semantics ► `Semaphore` / `SemaphoreSlim`
    │   │           (use `SemaphoreSlim(0, 1)` as an "auto-reset event slim" — there is none in BCL)
    │   │
    │   ├── Async waiters
    │   │   ├── One-shot completion ("the thing is done") ► `TaskCompletionSource` (with
    │   │   │       `TaskCreationOptions.RunContinuationsAsynchronously` — non-negotiable)
    │   │   ├── Broadcast gate (resettable) ──────────────► roll your own `AsyncManualResetEvent`
    │   │   │       on TCS (sample in the MRE/ARE note), or take `Nito.AsyncEx`
    │   │   ├── One-at-a-time turnstile ──────────────────► `SemaphoreSlim(0, 1)` + `WaitAsync`
    │   │   └── Cancellation as a signal ─────────────────► `CancellationTokenSource` +
    │   │           `token.Register(...)` or `await token.WaitHandle.WaitOneAsync()` patterns
    │   │
    │   └── Cross-process signaling ─────────────────────► named `EventWaitHandle`
    │
    ├── PIPELINE — producer(s) push items to consumer(s)
    │   │
    │   ├── Async, bounded backpressure, multi-producer/multi-consumer ► `Channel<T>`
    │   │       (default first choice — replaces 90% of hand-rolled MRE/ARE + queue)
    │   ├── Sync, blocking, bounded ────────────────────────────────────► `BlockingCollection<T>`
    │   │       (legacy but fine; `CompleteAdding` + `GetConsumingEnumerable`)
    │   ├── Lock-free FIFO/LIFO without bounds ────────────────────────► `ConcurrentQueue<T>` /
    │   │                                                                  `ConcurrentStack<T>`
    │   │       (no signaling — pair with `SemaphoreSlim` if consumers must wait)
    │   ├── Concurrent dictionary ────────────────────────────────────► `ConcurrentDictionary<K,V>`
    │   └── Bag-of-objects (no order) ────────────────────────────────► `ConcurrentBag<T>`
    │           (thread-local segments; only beats the queue when producer == consumer)
    │
    ├── RENDEZVOUS — wait for N things to happen
    │   │
    │   ├── Tasks, async ─────────────────► `Task.WhenAll(tasks)` / `Task.WhenAny(tasks)`
    │   ├── Signals from non-task code,
    │   │   known count ─────────────────► `CountdownEvent`
    │   │       (`Signal()` past zero throws; `Reset(N)` to reuse)
    │   ├── Phase synchronization for
    │   │   parallel algorithms ─────────► `Barrier`
    │   │       (each phase: workers `SignalAndWait`, then start next phase together)
    │   └── Wait on multiple `WaitHandle`s ► `WaitHandle.WaitAny` / `WaitAll`
    │           (`WaitAll` requires MTA on Windows; not for STA UI threads)
    │
    └── ONE-TIME INITIALIZATION
        │
        ├── Expensive construction, want thread-safe lazy ► `Lazy<T>` (default mode is
        │       `ExecutionAndPublication` — exactly-once construction, exceptions cached)
        ├── Cheap construction, fine to discard losers ──► `Interlocked.CompareExchange`
        ├── Init a field-of-T inside a class lazily ─────► `LazyInitializer.EnsureInitialized`
        └── Static field init at first type access ──────► plain `static readonly` field
                (CLR guarantees thread-safe `beforefieldinit` semantics)
```

## Modifier sub-trees (apply after picking from the master tree)

### Need cancellation?

| Primitive | Cancellation support |
|---|---|
| `lock` / `Lock` / `Monitor.Enter` | None — use `Monitor.TryEnter(timeout)` and check the token between attempts |
| `Monitor.TryEnter(TimeSpan)` | Timeout only |
| `SemaphoreSlim.WaitAsync(ct)` | **First-class** — throws `OperationCanceledException`, no permit consumed |
| `Mutex.WaitOne(timeout)` | Timeout only; no token overload |
| `ManualResetEvent.WaitOne(timeout)` | Timeout only |
| `ManualResetEventSlim.Wait(ct)` | **First-class** |
| `Task.WaitAsync(ct)` (.NET 6+) | Wraps any Task with cancellation |
| `CancellationToken.Register(...)` | Run code on cancel; no need to "wait" at all |

### Need reentrancy?

| Primitive | Reentrant? |
|---|---|
| `lock` / `Monitor` / `Lock` (.NET 9+) | Yes |
| `Mutex` | Yes (release once per acquire) |
| `SemaphoreSlim` / `Semaphore` | **No** — recursive `Wait` deadlocks silently |
| `ReaderWriterLockSlim` | Only with `LockRecursionPolicy.SupportsRecursion` (slower) |
| Async code | **No reentrant async lock in BCL** — restructure or use `Nito.AsyncEx.AsyncLock` |

### Need a condition wait ("release the lock while I sleep, reacquire when pulsed")?

Only `Monitor.Wait` / `Pulse` / `PulseAll` provides this — and the lock target **must be a plain `object`** (the new `System.Threading.Lock` does not support condition variables). In modern code, prefer `Channel<T>` or `TaskCompletionSource`.

### Need fairness (FIFO release order)?

None of the BCL primitives **contractually** guarantee FIFO. `SemaphoreSlim` is approximately FIFO in practice; `AutoResetEvent`, `Mutex`, kernel `Semaphore` are at the kernel scheduler's discretion. If you need fairness, use a `Channel<T>` or build a queue-of-`TaskCompletionSource`.

## Quick-lookup table

| Problem | Primitive |
|---|---|
| Atomic counter | `Interlocked.Increment` / `Add` |
| Atomic flag (single writer) | `volatile bool` |
| 64-bit field on any platform | `Volatile.Read/Write(ref long)` or `Interlocked.Read` |
| Atomic ref swap | `Interlocked.Exchange` |
| Lock-free CAS / push-on-list | `Interlocked.CompareExchange` |
| Critical section, sync code, .NET 9+ | `System.Threading.Lock` |
| Critical section, sync code, < .NET 9 | `lock (object)` |
| Critical section, async code | `SemaphoreSlim(1, 1)` + `WaitAsync` |
| Bounded concurrency on outbound calls | `SemaphoreSlim(N, N)` |
| "Wait until condition" inside a lock | `Monitor.Wait` / `Pulse` (with plain `object`) |
| Many readers, rare writers (profiled) | `ReaderWriterLockSlim` |
| Cross-process mutual exclusion | named `Mutex` |
| Cross-process counted slots | named `Semaphore` |
| Cross-process signal | named `EventWaitHandle` |
| Single-instance desktop app | named `Mutex` with `Global\` prefix |
| Cross-machine lock | distributed lock service (Redis, etcd, blob lease, …) |
| One-shot async completion | `TaskCompletionSource` (RunContinuationsAsynchronously) |
| Broadcast "ready" gate, sync | `ManualResetEvent[Slim]` |
| Single-waiter handoff, sync | `AutoResetEvent` |
| Single-waiter handoff, async | `SemaphoreSlim(0, 1)` + `WaitAsync` |
| Producer/consumer, async, bounded | `Channel<T>` |
| Producer/consumer, sync, blocking | `BlockingCollection<T>` |
| Concurrent map | `ConcurrentDictionary<K,V>` |
| Concurrent FIFO/LIFO | `ConcurrentQueue<T>` / `ConcurrentStack<T>` |
| Wait for N async tasks | `Task.WhenAll` / `WhenAny` |
| Wait for N non-task signals | `CountdownEvent` |
| Phase-synchronized parallel algo | `Barrier` |
| Thread-safe lazy init, expensive | `Lazy<T>` |
| Thread-safe lazy init, cheap | `Interlocked.CompareExchange` or `LazyInitializer` |
| Per-thread state | `ThreadLocal<T>` (or `[ThreadStatic]` for static fields) |
| Per-async-flow state | `AsyncLocal<T>` |

## Anti-patterns to recognize first

These are common "wrong tree" choices — recognizing them upstream saves a redesign:

- **`lock` around an `await`** — won't compile (CS1996); switch to `SemaphoreSlim`.
- **`SemaphoreSlim` recursively re-entered** — silent deadlock; restructure.
- **`Mutex` across `await`** — undefined behavior (thread-affine release on a continuation thread); use `SemaphoreSlim`.
- **`volatile` on a counter** — not atomic; use `Interlocked`.
- **`volatile long`** — does not compile; use `Volatile.Read/Write` or `Interlocked.Read`.
- **`AutoResetEvent` to count signals** — drops `Set`s when no waiter; use `Semaphore`/`SemaphoreSlim`/`Channel`.
- **`TaskCompletionSource` without `RunContinuationsAsynchronously`** — continuations run inline on the setter; classic deadlock.
- **`ReaderWriterLockSlim` chosen by reflex** — usually slower than `lock` until profiled. Default to `lock` and switch only with evidence.
- **`Mutex` for in-process exclusion** — kernel transition for nothing; use `lock`/`Lock`.
- **Locking on a `SemaphoreSlim` instance with `lock(...)`** — compiles, takes a `Monitor` on the semaphore object, completely unrelated to permits.
- **Double-checked locking with a non-`volatile` field** — broken on weak memory models (ARM64); use `volatile` reference, `Lazy<T>`, or `LazyInitializer`.
- **`Task.Result` / `.Wait()` inside library code** — not a sync primitive but worth flagging here: thread-pool starvation and SynchronizationContext deadlocks. Keep `async` end-to-end.
