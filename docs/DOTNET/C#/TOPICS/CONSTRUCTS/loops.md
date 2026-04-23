# Loops: `while`, `do-while`, `for`, `foreach`

_Targets .NET 10 / C# 14. See also: [Break, continue, return](./break-continue-return.md), [Async streams & `await foreach`](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md), [Collections](../COLLECTIONS/array-list-dictionary-hashset-stack-queue.md), [Span & Memory](../MEMORY_MANAGEMENT/span-and-memory.md)._

Four loop shapes that compile to wildly different IL. `foreach` in particular is a **pattern-based** construct that the compiler lowers — not a runtime feature — and that's where most allocation surprises live.

## `while` and `do-while`

Condition-gated loops with no loop-carried state owned by the construct:

```csharp
while (queue.TryDequeue(out var job))
    Process(job);

do
{
    line = reader.ReadLine();
    if (line is null) break;
    Handle(line);
} while (true);
```

Differences:
- `while` checks the condition **before** the first iteration → may run zero times.
- `do-while` runs the body **at least once** → the condition is checked after each iteration.

`while (true) { … break; }` is the honest way to write an infinite loop with internal exits — clearer than `for (;;)`.

## `for`

```csharp
for (int i = 0; i < arr.Length; i++)
    arr[i] *= 2;
```

Use `for` when you need the index for more than just iteration — random access, reverse iteration, stride, paired indices. For `int[]` and `List<T>` accessed by index, the JIT **eliminates bounds checks** on `for (int i = 0; i < arr.Length; i++)` because it can prove `i` is in range. Subtle rewrites — `length` cached in a local, `<=` vs `<`, reverse iteration — can foil this. The pattern above is the canonical one.

```csharp
// Reverse iteration — unchecked-for-bounds elim sometimes; measure.
for (int i = arr.Length - 1; i >= 0; i--)
    Process(arr[i]);

// Stride
for (int i = 0; i < data.Length; i += blockSize)
    ProcessBlock(data.Slice(i, Math.Min(blockSize, data.Length - i)));
```

## `foreach` — the pattern-based loop

`foreach` does **not** require `IEnumerable`/`IEnumerable<T>`. The compiler looks for this shape:

1. A `GetEnumerator()` method (instance **or** extension, C# 9+).
2. Returning something with `MoveNext()` that returns `bool`.
3. And a `Current` property (any type).

That's it. If the enumerator type also implements `IDisposable` (or has a pattern-based `Dispose()`), `foreach` wraps the loop in `try/finally` and calls `Dispose()`.

This duck-typing is why `foreach` can iterate `Span<T>` (whose enumerator is a `ref struct` — can't implement interfaces pre–C# 13) and why `IAsyncEnumerable<T>` + `await foreach` work via `GetAsyncEnumerator()`.

### Compiler lowering, roughly

```csharp
foreach (var x in list) Use(x);

// ≈ compiles to:
var e = list.GetEnumerator();
try
{
    while (e.MoveNext())
    {
        var x = e.Current;
        Use(x);
    }
}
finally
{
    ((IDisposable)e).Dispose();   // only if the enumerator is disposable
}
```

### Struct enumerators = zero allocation

```csharp
var list = new List<int> { 1, 2, 3 };
foreach (var x in list) …      // uses `List<T>.Enumerator` struct — on the stack
```

`List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, `Stack<T>`, `Queue<T>`, and most BCL collections expose a **struct** enumerator named `Enumerator`. The `foreach` variable is a local, and `MoveNext()`/`Current` are direct struct calls. No heap allocation, no interface dispatch.

Cast the same collection to `IEnumerable<T>` and you lose all of that:

```csharp
IEnumerable<int> e = list;
foreach (var x in e) …          // boxes the struct enumerator → heap allocation
```

The compiler uses **the static type of the expression** (`list` vs `IEnumerable<T>`) to decide. So on hot paths: keep the concrete type (or `var`), and resist the urge to "program to the interface" through a `foreach` header.

### `foreach` over `Span<T>` / `ReadOnlySpan<T>`

```csharp
Span<byte> buffer = stackalloc byte[256];
foreach (ref byte b in buffer)       // C# 7.3+: ref iteration
    b = 0;
```

`Span<T>`'s enumerator is a `ref struct` — can only live on the stack. `foreach` sees it via pattern matching. Use `foreach (ref var x in span)` to mutate elements in place, or `foreach (ref readonly var x in span)` when you only read.

### `foreach` on collection expressions and ranges

```csharp
foreach (var x in (ReadOnlySpan<int>)[1, 2, 3, 4]) …    // collection expression, zero heap

foreach (var x in arr[1..^1]) …                          // iterate slice — allocates a copy for T[]
foreach (var x in arr.AsSpan()[1..^1]) …                 // iterate a span slice — zero alloc
```

`T[]` slicing with `a[1..^1]` **allocates a new array**. `Span<T>` slicing doesn't — it's a view.

### `await foreach` — the async form

```csharp
await foreach (var line in ReadLinesAsync(path, ct).WithCancellation(ct).ConfigureAwait(false))
    Handle(line);
```

Pattern-based on `GetAsyncEnumerator(CancellationToken)`, which returns something with `MoveNextAsync()` and a sync `Current`. See [async streams](../ASYNC_PROGRAMMING/async-streams-and-await-foreach.md).

## Iterating with mutation — the forbidden move

```csharp
foreach (var x in list)
    if (Predicate(x)) list.Remove(x);     // 💥 InvalidOperationException
```

Most BCL collections bump an internal `_version` on mutation; each `MoveNext()` checks it. The canonical fixes:

```csharp
// 1) Materialize first:
foreach (var x in list.ToArray())
    if (Predicate(x)) list.Remove(x);

// 2) Use `RemoveAll` (List<T>) — single O(n) pass:
list.RemoveAll(Predicate);

// 3) Reverse index loop (stable against RemoveAt):
for (int i = list.Count - 1; i >= 0; i--)
    if (Predicate(list[i])) list.RemoveAt(i);
```

`Dictionary<K,V>` throws even on **value** mutation via the enumerator's key. For in-place updates, collect keys first or use `CollectionsMarshal.GetValueRefOrNullRef` outside the loop.

## Iterators — `yield return` / `yield break`

```csharp
public static IEnumerable<int> Even(IEnumerable<int> src)
{
    foreach (var x in src)
    {
        if (x < 0) yield break;           // stop enumerating
        if (x % 2 == 0) yield return x;   // produce a value, suspend
    }
}
```

The compiler turns the method into a state machine that implements both `IEnumerable<T>` and `IEnumerator<T>`. Execution is **deferred** — nothing runs until the caller starts iterating, and **each `foreach` over the result restarts from the top** (the enumerator is single-pass, but the enumerable can spin up new ones).

Caveats:
- Cannot have `try/catch` around a `yield return` in older C# (the rule has relaxed over the years — in modern C# a `try/catch` is allowed if no `yield` is **inside** the catch).
- No `ref`/`out` params on iterators. Argument validation executes **lazily** — use a wrapper method to eager-check:

    ```csharp
    public static IEnumerable<T> Take<T>(IEnumerable<T> src, int n)
    {
        ArgumentNullException.ThrowIfNull(src);
        if (n < 0) throw new ArgumentOutOfRangeException(nameof(n));
        return TakeIterator(src, n);

        static IEnumerable<T> TakeIterator(IEnumerable<T> src, int n)
        {
            foreach (var x in src) { if (n-- <= 0) yield break; yield return x; }
        }
    }
    ```

## Loop choice cheat-sheet

| Situation | Pick |
|---|---|
| Index or stride needed | `for` |
| Reverse iteration | `for` (downward) |
| Walk a collection, produce nothing | `foreach` on the concrete type |
| Transform → new collection | LINQ `Select` + `ToArray`/`ToList`, not a loop |
| Condition-driven, unknown count | `while` |
| Must run once, then maybe again | `do-while` |
| Deferred projection | iterator method with `yield return` |
| `async` iteration | `await foreach` |

## Senior-level gotchas

- **Typing a collection as `IEnumerable<T>` before `foreach` boxes the struct enumerator.** On hot paths, keep the concrete type or pass `ReadOnlySpan<T>`.
- **`foreach` is pattern-based, not interface-based** — that's how `Span<T>` iterates without boxing and how extension `GetEnumerator()` lets you `foreach` over types you don't own.
- **`for (int i = 0; i < arr.Length; i++)` is the form the JIT optimizes best** — caching `arr.Length` in a local can actually *hurt* because it defeats the JIT's proof that `i` is in range.
- **Mutating a collection while enumerating throws `InvalidOperationException`** via the `_version` check. `ConcurrentDictionary` is the exception — its enumerator is a live snapshot and tolerates concurrent writes.
- **Array slicing (`arr[1..^1]`) allocates**; span slicing (`arr.AsSpan()[1..^1]`) does not. On hot paths, slice spans.
- **Iterator argument validation is lazy** — bad arguments surface on first `MoveNext()`, not at call time. Wrap with an eager-throwing outer method.
- **`foreach (ref var x in span)`** mutates in place; `foreach (var x in span)` gives you a copy. The distinction matters for large structs.
- **`IEnumerable<T>` vs `IEnumerator<T>`**: never cache an enumerator across threads or `await` boundaries — they're single-consumer, stateful objects.
- **`await foreach` without a `CancellationToken`** blocks shutdown. Always pass the token via `.WithCancellation(ct)` or as a parameter to the iterator.
