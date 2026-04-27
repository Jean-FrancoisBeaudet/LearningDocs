# `Semaphore`, `Mutex`, and `Monitor`

_Targets .NET 10 / C# 14. See also: [`lock` & `SemaphoreSlim`](./lock-statement-and-semaphoreslim.md), [MRE & ARE](./manualresetevent-and-autoresetevent.md), [`volatile` & `Interlocked`](./volatile-and-interlocked.md), [`using` & `lock`](../CONSTRUCTS/using-and-lock.md)._

The three "classic" synchronization primitives. `Monitor` is the engine behind `lock` and the only one that exposes **condition variables** via `Wait`/`Pulse`. `Mutex` is a kernel-mode mutual-exclusion handle, the right tool for **cross-process** coordination. `Semaphore` is its counted cousin. All three are heavier than their in-process Slim variants — reach for them only when you need their specific superpower.

## `Monitor` — the engine behind `lock`

Every reference type has a hidden monitor in its object header. `lock (x)` is sugar for `Monitor.Enter(x, ref taken)` + `Monitor.Exit(x)` in a `try/finally`. Use `Monitor` directly only when you need:
- A **timeout** on acquisition.
- A **condition variable** (`Wait`/`Pulse`/`PulseAll`).

### Timeout

```csharp
if (!Monitor.TryEnter(_gate, TimeSpan.FromMilliseconds(50)))
    throw new TimeoutException();

try     { Mutate(); }
finally { Monitor.Exit(_gate); }
```

Surface the timeout as a metric — it's usually a sign of contention worth investigating.

### `Wait` / `Pulse` — condition variables

`Monitor.Wait` releases the lock atomically, blocks until pulsed, and reacquires the lock before returning. This is the only correct primitive for "wait for a condition while holding the lock":

```csharp
public sealed class BoundedBuffer<T>
{
    private readonly Queue<T> _items = new();
    private readonly object _gate = new();
    private readonly int _capacity;

    public BoundedBuffer(int capacity) => _capacity = capacity;

    public void Put(T item)
    {
        lock (_gate)
        {
            while (_items.Count == _capacity)
                Monitor.Wait(_gate);   // releases _gate, waits, reacquires

            _items.Enqueue(item);
            Monitor.Pulse(_gate);      // wake one waiter (Take side)
        }
    }

    public T Take()
    {
        lock (_gate)
        {
            while (_items.Count == 0)
                Monitor.Wait(_gate);

            var item = _items.Dequeue();
            Monitor.Pulse(_gate);
            return item;
        }
    }
}
```

Two non-negotiable rules:
- **Always use `while`, never `if`.** Spurious wakeups, multiple producers, and `PulseAll` mean the condition can be false even after `Wait` returns.
- **You must hold the lock when calling `Wait`, `Pulse`, or `PulseAll`.** Calling them outside the `lock` throws `SynchronizationLockException`.

`PulseAll` wakes every waiter — useful when the condition change is broadcast-shaped (e.g., "queue closed"). Overusing it produces a thundering herd: all waiters wake, all reacquire the lock, all but one go back to sleep.

In modern code, `Channel<T>` replaces 90% of hand-rolled `Wait`/`Pulse` patterns.

> Note: `Monitor.Wait`/`Pulse` only work on objects locked via the legacy `Monitor` path. The new `System.Threading.Lock` (.NET 9+) does **not** support condition variables — if you need `Wait`/`Pulse`, use a plain `object _gate = new();` with `lock`.

## `Mutex` — cross-process mutual exclusion

Kernel-mode, **named**, thread-affine, reentrant.

```csharp
using var mtx = new Mutex(initiallyOwned: true,
                          name: @"Global\Acme.MyApp.SingleInstance",
                          createdNew: out var first);

if (!first)
{
    Console.Error.WriteLine("Another instance is running.");
    return;
}

try     { RunApplication(); }
finally { mtx.ReleaseMutex(); }
```

Properties:
- **Cross-process** when given a name. Anonymous `Mutex` instances are in-process and are basically a `Monitor` with extra cost.
- **Reentrant**: the same thread can `WaitOne` repeatedly; release once per acquire.
- **Thread-affine**: only the acquiring thread may release. After `await`, you're on a different thread — `Mutex` is *not* async-safe. There is no `MutexAsync`.
- **Abandoned mutex**: if the owning thread exits without `ReleaseMutex()`, the next acquirer's `WaitOne` throws `AbandonedMutexException`. The mutex *is* acquired (the exception's `Mutex` property is set), but the shared state may be corrupt — handle it, validate state, log it, and proceed with care:

  ```csharp
  try     { mtx.WaitOne(); }
  catch (AbandonedMutexException) { /* recover invariants, then continue */ }
  ```

Cost: a `Mutex.WaitOne` is at least a system call. It is **measurably slower** than `lock` — only use it when you need cross-process semantics or the abandoned-mutex contract.

## `Semaphore` — kernel counting semaphore

Same shape as `Mutex` — kernel-backed, can be **named**, cross-process. Counted permits; nothing about thread affinity (any thread may release a permit, just like `SemaphoreSlim`).

```csharp
using var sem = new Semaphore(
    initialCount: 4,
    maximumCount: 4,
    name: @"Global\Acme.RateLimit");

sem.WaitOne();
try     { CallExternalApi(); }
finally { sem.Release(); }
```

Use when:
- Multiple processes share a resource (rate limit, license slots).
- Interop with native code expecting a `HANDLE`.

In a single process, `SemaphoreSlim` wins on every dimension (cheaper, async-aware, cancellation). `Semaphore` (capital S) **has no `WaitAsync`** — bridging it requires `RegisterWaitForSingleObject`, which is awkward.

## Worked example — single-instance Windows desktop app

```csharp
const string Name = @"Global\Acme.MyApp.{guid}";
using var mtx = new Mutex(true, Name, out var firstInstance);

if (!firstInstance)
{
    NotifyExistingInstance();   // e.g., named pipe to bring window to front
    return;
}

try     { Application.Run(new MainForm()); }
finally { mtx.ReleaseMutex(); }
```

The `Global\` prefix makes the mutex visible to all sessions on the machine; without it the mutex is per-session, which is *usually* what you want for a desktop app (one-per-user). Choose deliberately.

## Comparison

| | `Monitor` (`lock`) | `Mutex` | `Semaphore` (kernel) | `SemaphoreSlim` |
|---|---|---|---|---|
| Scope | In-process | **Cross-process** (named) | **Cross-process** (named) | In-process |
| Reentrant | Yes | Yes | n/a (counting) | No |
| Thread-affine | Yes | **Yes** | No | No |
| Async-aware | No | No | No | **Yes** |
| Cost (uncontended) | ~20 ns | μs (syscall) | μs (syscall) | ~50 ns |
| Condition variable | **Yes** (`Wait`/`Pulse`) | No | No | No |
| Abandoned-handle behavior | n/a (in-process) | `AbandonedMutexException` | n/a | n/a |
| Disposable | n/a (header) | Yes (handle) | Yes (handle) | Yes |

## Senior-level gotchas

- **`Monitor.Wait`/`Pulse` must be called while holding the lock.** Outside the `lock` they throw `SynchronizationLockException`. Inside the `lock` but on the wrong object (a different lock target) is a silent bug.
- **`Monitor.Wait` always uses `while`, not `if`.** Spurious wakeups exist. The pattern is `while (!cond) Monitor.Wait(gate);`.
- **`Monitor.PulseAll` is a thundering-herd amplifier.** Pulse only when the condition meaningfully changed. Prefer `Pulse` for single-slot conditions.
- **The new `System.Threading.Lock` does not support `Wait`/`Pulse`.** If you need condition variables, keep the lock target a plain `object`.
- **`AbandonedMutexException`** means "you got the mutex, but the previous owner died without releasing it." The exception's `Mutex` property is set; you now own it. Validate state and decide whether to proceed or fail loudly — never silently swallow.
- **`Mutex` is thread-affine.** A thread that `WaitOne`s must be the same thread that calls `ReleaseMutex`. After `await`, the continuation is almost certainly on a different thread — never use `Mutex` across `await`.
- **`Semaphore` (kernel) throws `SemaphoreFullException`** if you `Release` more times than `maximumCount` allows. Pass an honest `maximumCount` and audit your `Release`s.
- **Named-handle name collisions are silent across applications.** Pick a name that includes a vendor and a stable GUID/version. `Global\Foo` is a recipe for cross-app interference.
- **The `Global\` prefix matters on Windows.** Without it, names live in the user's session namespace, and another logon on the same machine has its own. With it, you must consider permissions; sandboxed apps may be denied.
- **`Mutex`/`Semaphore` are not portably released on process crash.** Windows releases mutexes when the owning thread/process terminates (this is what triggers `AbandonedMutexException`); on Linux, cleanup is best-effort. Don't rely on crash-cleanup for critical invariants.
- **`Monitor.Enter(obj)` without `ref taken`** is the legacy overload — if `Enter` throws after acquiring (rare under thread abort or async exceptions), the `Exit` in `finally` runs without the lock and throws too. Always use `lock` or `Monitor.Enter(obj, ref taken)`.
