# Parallel LINQ (PLINQ)

_Targets .NET 10 / C# 14. See also: [Parallel class](./parallel-class.md), [Thread/ThreadPool/Task/TaskFactory](./thread-threadpool-task-taskfactory.md), [Task continuation, cancellation, and multiple tasks](./task-continuation-cancellation-and-multiple-tasks.md)._

PLINQ (`System.Linq.ParallelEnumerable`) is LINQ-to-Objects executed across multiple threads. You turn any `IEnumerable<T>` into a `ParallelQuery<T>` with `.AsParallel()` and most LINQ operators — `Where`, `Select`, `GroupBy`, `Aggregate` — run in parallel partitions.

It is specifically for **CPU-bound, side-effect-free, order-agnostic** queries over in-memory data. Used correctly, it's a one-line speedup on large transforms. Used badly, it's slower than sequential LINQ and harder to debug.

## The basic shape

```csharp
int[] primes = numbers
    .AsParallel()
    .Where(IsPrime)              // runs on N worker partitions
    .ToArray();
```

`AsParallel` returns a `ParallelQuery<T>`; subsequent operators dispatch to parallel overloads. The query doesn't execute until a terminal operator (`ToArray`, `ToList`, `First`, `Count`, `foreach`, `ForAll`).

### The three configuration knobs

```csharp
var result = source
    .AsParallel()
    .WithDegreeOfParallelism(8)
    .WithCancellation(ct)
    .WithMergeOptions(ParallelMergeOptions.NotBuffered)
    .Select(Transform)
    .ToList();
```

| Option | Meaning |
|---|---|
| `WithDegreeOfParallelism(n)` | Cap workers. Default = `Environment.ProcessorCount` (max 512). |
| `WithCancellation(ct)` | Plumbs a token — partition workers throw OCE at checkpoints. |
| `WithMergeOptions(...)` | How results are re-serialized: `AutoBuffered`, `FullyBuffered`, `NotBuffered`. |
| `AsOrdered()` / `AsUnordered()` | Preserve / drop original sequence order. |
| `WithExecutionMode(...)` | `Default` (PLINQ may refuse to parallelize) vs `ForceParallelism`. |

## Ordering — the expensive default

`AsParallel()` does **not** preserve input order by default. `AsOrdered()` tells PLINQ to reassemble results in the original sequence's order:

```csharp
var orderedResults = source
    .AsParallel()
    .AsOrdered()
    .Select(Transform)
    .ToArray();
```

Cost: the merge stage has to buffer and sort. For many queries that negates the parallelism win. Only use `AsOrdered` when a downstream step depends on position, and consider `AsUnordered` mid-query to drop the ordering constraint once you no longer need it:

```csharp
source.AsParallel().AsOrdered()
    .Where(Keep)       // keeps order
    .AsUnordered()     // drops order from here on
    .Select(Heavy)
    .ToArray();
```

## `ForAll` — parallel foreach without merging

```csharp
source.AsParallel()
      .ForAll(item => Write(item));
```

`ForAll` runs the action on each partition's thread with **no merge step** — faster than a `foreach` over the query because it skips the final serialization. Use it when the action is the sink (e.g., writing to a thread-safe collection or sending to a channel) and you don't need a result.

## Merge modes

| Mode | Behavior | Use |
|---|---|---|
| `AutoBuffered` (default) | Workers fill small buffers, consumer pulls them. | General purpose — good latency/throughput balance. |
| `FullyBuffered` | All workers finish; consumer gets everything at once. | When you call `ToArray` / `ToList` anyway — can be faster because workers don't wait on the consumer. |
| `NotBuffered` | Every produced element is handed off immediately. | Streaming consumers that want each element ASAP (rare). |

For queries that terminate in `ToList`/`ToArray` over CPU-bound data, `FullyBuffered` is often the fastest.

## Aggregation

```csharp
long total = source
    .AsParallel()
    .Aggregate(
        seed:         () => 0L,
        updateAccumulatorFunc: (localSum, item) => localSum + item.Bytes,
        combineAccumulatorsFunc: (a, b) => a + b,
        resultSelector: x => x);
```

The three-argument-plus-combiner form is the parallel sweet spot: each partition keeps a private accumulator (no contention), then a single combiner reduces them at the end. This replaces the `Parallel.For` + `Interlocked` pattern for pure reductions and is often cleaner.

The single-argument `Aggregate(func)` overload is **not** safe for parallel use — `func` is invoked concurrently on shared state. Use the 4-arg form.

## When PLINQ helps

A rough decision table — all of these must be true for PLINQ to beat sequential LINQ:

1. **Source size is large.** Below ~1000 items, partition overhead dominates.
2. **Per-element work is non-trivial.** A CPU-bound `Select` over image decoding, parsing, hashing, crypto. A `Select(x => x + 1)` will always be slower in parallel.
3. **Operators are pure.** No shared mutable state, no I/O, no UI, no event raising.
4. **Ordering doesn't matter** (or matters only at the end, where `AsOrdered` is acceptable).
5. **You're CPU-bound.** Parallelizing async I/O via PLINQ does not add real concurrency — use `Parallel.ForEachAsync` or `Task.WhenAll` instead.

## When PLINQ hurts

- **Short operator chains** with cheap bodies: partitioning overhead > savings.
- **Operators with cross-element dependencies**: `Distinct`, `OrderBy`, `GroupBy`, `Zip` — these force buffering/synchronization that may be slower than sequential.
- **Stateful selectors or lambdas capturing mutable locals**: undefined behavior at best, race at worst.
- **I/O-bound work** (`Select(async ...)` does NOT exist for PLINQ — use async fan-out tools).
- **Queries followed by lots of small LINQ ops**: partitioning and re-merging per operator adds up.

## Measuring, not guessing

Use `BenchmarkDotNet` with a `[MemoryDiagnoser]` to compare `.AsParallel()` vs sequential for your specific workload, source size, and core count. A common outcome: PLINQ helps starting around 10k–100k items on genuinely CPU-heavy operations, and hurts below 1k items or under lightweight transforms.

```csharp
[Benchmark(Baseline = true)]
public int Sequential() => _data.Select(Hash).Sum();

[Benchmark]
public int Parallel() => _data.AsParallel().Select(Hash).Sum();
```

## PLINQ vs `Parallel.ForEach`

| Use PLINQ when | Use `Parallel.ForEach` when |
|---|---|
| The pipeline has multiple LINQ operators. | You just want a parallel `foreach` with a body. |
| You need an aggregated result (`Sum`, `Count`, `Aggregate`). | You're mutating external state with per-thread locals. |
| Composition with LINQ syntax is clearer. | You need `state.Break()` / `state.Stop()`. |
| Large data pipelines in reporting / ETL-style code. | Unknown source size where Partitioner options matter. |

Both sit on the same TPL substrate; the choice is ergonomic, not performance.

## Exceptions

```csharp
try
{
    var results = source.AsParallel().Select(Risky).ToList();
}
catch (AggregateException ex)
{
    foreach (var inner in ex.Flatten().InnerExceptions) { ... }
}
```

Partition workers collect exceptions and PLINQ rethrows them as an `AggregateException` at the terminal operator. Unlike `await`, there's no unwrapping to the first inner — always handle the aggregate. `.Flatten()` collapses nested aggregates (common when operators compose aggregates internally).

**Senior-level gotchas:**
- PLINQ ordering cost is real and often silent. A benchmark that shows "PLINQ is 3× slower than sequential" usually has an `OrderBy`, `Distinct`, or `AsOrdered` hiding in the pipeline.
- `.AsParallel().First()` with `ParallelExecutionMode.Default` may actually refuse to parallelize and fall back to sequential. The planner is conservative for operators where parallelism rarely wins. Force with `WithExecutionMode(ParallelExecutionMode.ForceParallelism)` if you've measured a win.
- `ForAll` has no implicit ordering and **no error aggregation until the enumeration finishes**. If you need predictable ordering or immediate failure, use a normal `foreach` over the query.
- PLINQ captures `TaskScheduler.Default`, not `TaskScheduler.Current`. That's actually correct (don't run data-parallel work on a UI dispatcher), but it means inside the lambdas, any `ConfigureAwait(true)` is meaningless — there's no captured context.
- `.AsParallel()` on an `IEnumerable<T>` that isn't indexable (a random `yield` iterator) uses chunk partitioning — every worker competes for a lock to grab the next chunk. For very hot iterator sources, materialize into an array first: `source.ToArray().AsParallel()`.
- PLINQ does nothing for `IQueryable<T>` — EF Core queries are already sent to the database. Calling `.AsParallel()` on a `DbSet<T>` runs sequentially and wastes effort.
- Nested PLINQ (`source.AsParallel().Select(x => x.Children.AsParallel().Sum())`) is almost always a mistake — you multiply the worker count and starve the pool. Collapse or flatten.
- Forgetting `WithCancellation(ct)` means an `ct.Cancel()` from the outside has no effect on a running PLINQ query. Cancellation is opt-in for PLINQ, unlike `Parallel.For*` which takes it via `ParallelOptions`.
