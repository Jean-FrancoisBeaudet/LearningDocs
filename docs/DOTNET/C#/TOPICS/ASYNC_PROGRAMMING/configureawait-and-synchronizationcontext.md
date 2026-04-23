# ConfigureAwait and SynchronizationContext

_Targets .NET 10 / C# 14. See also: [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [ValueTask](./valuetask.md)._

`SynchronizationContext` is the abstraction that models "a thing that knows how to post work back to a specific thread or queue." `await`, by default, captures that context and resumes on it. `ConfigureAwait(false)` opts out of the capture. Together they explain most async deadlocks and most of the advice in async style guides.

## What `await` actually does

At the state-machine level, `await task` is roughly:

```csharp
// Pseudo-code for: var x = await task;
var awaiter = task.GetAwaiter();
if (!awaiter.IsCompleted)
{
    // 1. Capture the RESUMPTION context.
    var ctx = SynchronizationContext.Current;   // or TaskScheduler.Current if null
    awaiter.OnCompleted(() =>
    {
        if (ctx is not null) ctx.Post(ResumeStateMachine, null);
        else                 ResumeStateMachine(null);
    });
    return;   // unwind
}
var x = awaiter.GetResult();
```

`ConfigureAwait(false)` returns a different awaiter whose `OnCompleted` skips the capture and resumes on whatever thread finished the task (typically a thread-pool thread).

## Which hosts install a SynchronizationContext?

| Host | `SynchronizationContext.Current` on entry |
|---|---|
| WinForms | `WindowsFormsSynchronizationContext` → pumps messages on the UI thread |
| WPF | `DispatcherSynchronizationContext` → the UI dispatcher |
| MAUI | Platform-specific UI context |
| **ASP.NET (classic, .NET Framework)** | `AspNetSynchronizationContext` → enforces single-thread request affinity |
| **ASP.NET Core** | **`null`** — no context is installed |
| Console / Worker Service / gRPC | **`null`** |
| xUnit `[Fact]` | **`null`** (xUnit provides a `MaxConcurrencySyncContext` for `[Collection]`s but no UI affinity) |

This is the punchline: in modern ASP.NET Core and console services, `await` resumes on the thread pool regardless of `ConfigureAwait`. The keyword does (almost) nothing there.

## Why the sync-over-async deadlock happens

```csharp
// UI button click handler:
void OnClick(object? s, EventArgs e)
{
    var result = FetchAsync().Result;   // ← blocks the UI thread
    label.Text = result;
}

async Task<string> FetchAsync()
{
    var data = await _http.GetStringAsync("/api/x");   // captures UI context
    return data.Trim();                                 // wants to run on UI thread
}
```

Trace: `.Result` blocks the UI thread. When `GetStringAsync` completes, its continuation tries to `Post` back to the captured UI context — which is waiting on `.Result`. Both sides wait forever.

Two ways to fix it (pick one):

```csharp
// A. Don't block. Make the caller async.
async void OnClick(object? s, EventArgs e) => label.Text = await FetchAsync();

// B. In the library, opt out of the capture so there's nothing to post back to.
var data = await _http.GetStringAsync("/api/x").ConfigureAwait(false);
```

`A` is the real fix. `B` is defense-in-depth for library authors who cannot control how callers invoke them. Even on ASP.NET Core where there is no context, library authors still often add `ConfigureAwait(false)` for portability — your library might be consumed by a WPF app tomorrow.

## `ConfigureAwait(false)` — is it still needed in .NET Core?

Short answer: **only in libraries**, and even then the benefit is marginal in Core-only codebases. It is not needed in application code running on ASP.NET Core, console, workers, or tests.

Slightly longer answer:
- **Application code on Core:** no context, nothing to capture, no deadlock risk. Don't pepper your code with it.
- **Library code (NuGet packages, shared assemblies):** add it. Your consumer may still be a WPF app, a .NET Framework legacy service, or a Xamarin app that *does* have a context.
- **UI code (WinForms, WPF, MAUI):** do **not** add it to methods that touch UI controls after the await — you'll bounce off the UI thread and then try to touch controls from a non-UI thread. `ConfigureAwait(false)` is for methods that don't need the UI context on resume.

Static analyzer **CA2007** flags library code missing `ConfigureAwait`. Configure it off by default in app projects, on by default in library projects.

## `ConfigureAwaitOptions` (.NET 8+)

The old `ConfigureAwait(bool)` became overloaded with more knobs. On `Task` / `Task<T>`:

```csharp
await task.ConfigureAwait(
    ConfigureAwaitOptions.ContinueOnCapturedContext   // same as ConfigureAwait(true)
  | ConfigureAwaitOptions.SuppressThrowing            // don't rethrow on await
  | ConfigureAwaitOptions.ForceYielding);             // always yield, even if completed
```

- `SuppressThrowing` — await completes normally even if the task faulted. Rarely what you want; useful to drain a task you already logged for.
- `ForceYielding` — equivalent to `await Task.Yield()` followed by `await task`. Handy to break fairness issues where a synchronously-completing task would starve other work.
- `None` is equivalent to `ConfigureAwait(false)`.

Only `Task`/`Task<T>` got this overload. `ValueTask` still has `ConfigureAwait(bool)` only.

## `async void` and sync contexts

`async void` posts its exceptions to the captured `SynchronizationContext` — in a UI app that surfaces them on the dispatcher and typically crashes the process unless the host catches them. In an ASP.NET Core request pipeline (no context), an `async void` exception crashes the process via unhandled exception. Either way: never write `async void` except for event handlers.

## Cross-refs for interview answers

- "Why does `.Result` deadlock in WPF?" → captured `SynchronizationContext` + blocked resumption thread.
- "When do I need `ConfigureAwait(false)`?" → library code targeting any TFM that might run under a sync context.
- "What changed in ASP.NET Core vs classic?" → no `AspNetSynchronizationContext`, so no request-thread affinity, no deadlock from `.Result`, and `HttpContext.Current` is gone (use DI instead).

**Senior-level gotchas:**
- `ConfigureAwait(false)` is **pointwise**: it affects the single `await` it's applied to, not subsequent ones. A method is only "context-free" if every await on its hot path has it.
- `ValueTask.ConfigureAwait(false)` exists; it is **not** the default — `ValueTask` captures context just like `Task`.
- Inside a `using`/`finally` block after a `ConfigureAwait(false)` await, `Dispose` / `DisposeAsync` runs on a pool thread. If your disposable is thread-affine (e.g., a `DbContext` in some legacy wrappers), this manifests as "works in dev, crashes in WPF."
- A `TaskScheduler` can also cause a capture-style resumption: if `SynchronizationContext.Current` is `null` but `TaskScheduler.Current` is non-default (e.g., you're inside a `StartNew` with a custom scheduler), the awaiter captures *that*. `ConfigureAwait(false)` suppresses both.
- `await foreach` has its own `ConfigureAwait` on the `IAsyncEnumerable<T>` (via `WithCancellation(ct).ConfigureAwait(false)`) — the inner `MoveNextAsync` awaits do **not** inherit the outer method's configuration. See [async-streams-and-await-foreach.md](./async-streams-and-await-foreach.md).
- `.NET 8`'s `ConfigureAwait(ConfigureAwaitOptions.None)` is not the same numeric value as `ConfigureAwait(false)` in IL, but they behave identically — don't write tests that assert on the enum value.
