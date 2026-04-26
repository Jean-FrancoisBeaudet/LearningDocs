# BenchmarkDotNet

_Targets .NET 10 / C# 14, BenchmarkDotNet 0.14+. See also: [.NET performance anti-patterns](./dotnet-performance-anti-patterns.md), [Allocations and boxing](./allocations-and-boxing.md), [Low-allocation patterns in modern .NET](./low-allocation-patterns-modern-dotnet.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [Performance regression tracking](./performance-regression-tracking.md), [Performance baselines and release gates](./performance-baselines-and-release-gates.md), [Span&lt;T&gt;](./span-of-t.md)._

The de facto micro-benchmark harness for .NET. Its actual job is **statistical rigor** — isolating the hot loop, controlling JIT warm-up, running enough iterations to get a confidence interval, and silently defending you against the dozen common micro-benchmark mistakes (dead-code elimination, constant folding, cold JIT, GC interference, boxing inside the loop, ambient state contamination). Writing a `Stopwatch` loop and trusting the number is a beginner mistake; BDN is the grown-up version.

It is **not** a load tool, **not** a profiler, and **not** a substitute for end-to-end measurement. Its outputs are per-method, in-process numbers — useful for choosing between two implementations, useless for proving that a change improves p99 latency under real traffic.

## Anatomy of a benchmark

```csharp
using System.IO.Hashing;
using System.Security.Cryptography;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class HashBench
{
    private byte[] _data = null!;

    [Params(64, 4096, 65536)]
    public int Size { get; set; }

    [GlobalSetup]
    public void Setup() => _data = new byte[Size];

    [Benchmark(Baseline = true)]
    public int Sha256() => SHA256.HashData(_data).Length;

    [Benchmark]
    public int XxHash3() => (int)XxHash3.HashToUInt64(_data);
}

public static class Program
{
    public static void Main(string[] args) =>
        BenchmarkRunner.Run<HashBench>(args: args);
}
```

Drive it from a **Release-mode console app** — BDN refuses to run a Debug build and prints a clear error if you try. The runner spawns one child process per `[Job]` so the host process's JIT history doesn't pollute the measurements.

## What BDN actually controls for

| Concern | BDN's response |
|---|---|
| Cold JIT | Warmup iterations until per-iteration time stabilizes |
| Tiered compilation | Tier-1 promotion happens during warmup; defaults to disabled in benchmark jobs |
| Variable iteration count | Pilot stage tunes `InvocationsPerIteration` so each iteration takes ~250 ms |
| GC interference | Forces collection between iterations when `[MemoryDiagnoser]` is on |
| Dead-code elimination | Return value is consumed via `Consumer.Consume` |
| In-process noise | Each job is a separate process with a fresh CLR |
| Statistical confidence | Outliers detected and reported; `Mean ± Error (99.9% CI)` |
| Environment drift | Header in every report records OS, CPU, .NET version, AVX availability |

Read the **environment header** at the top of every results table; cross-machine comparison is meaningless without it.

## Diagnosers

A diagnoser is an opt-in instrument that adds columns or files to the report.

| Diagnoser | What it adds |
|---|---|
| `[MemoryDiagnoser]` | Gen0/1/2 collection counts, allocated bytes per op |
| `[ThreadingDiagnoser]` | Lock contentions, completed work items |
| `[ExceptionDiagnoser]` | Exceptions thrown per op (silent perf killer) |
| `[EtwProfiler]` (Windows) | Captures an `.etl` for each benchmark — open in PerfView |
| `[EventPipeProfiler(EventPipeProfile.GcVerbose)]` | Cross-platform alternative to ETW |
| `[DisassemblyDiagnoser(maxDepth: 3)]` | Emits the JITted asm — the closest thing to "look at the actual codegen" |
| `[InliningDiagnoser]` | Reports which methods got inlined |
| `[TailCallDiagnoser]` | Detects tail-call optimization in hot paths |

`MemoryDiagnoser` is essentially free — turn it on by default. `DisassemblyDiagnoser` is for the moments where a perf delta has no obvious source code explanation.

## Multi-runtime / parameter sweeps

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
[SimpleJob(RuntimeMoniker.Net90)]
[SimpleJob(RuntimeMoniker.Net90, Jit.NoTiered, baseline: false)] // tier-1-only
public class SerializerBench
{
    [ParamsSource(nameof(Payloads))]
    public Payload Input { get; set; } = null!;

    public static IEnumerable<Payload> Payloads() => new[]
    {
        Payload.Small, Payload.Medium, Payload.Large
    };

    [Benchmark(Baseline = true)] public byte[] SystemTextJson() => /* ... */;
    [Benchmark]                  public byte[] MessagePack()    => /* ... */;
}
```

The result is a matrix: every `(Runtime, Input, Method)` combination is one row. BDN handles the cartesian product; you don't write loops.

## Reading the output

```
| Method   | Size  | Mean      | Error    | StdDev   | Ratio | Allocated | Gen0   |
|--------- |------ |----------:|---------:|---------:|------:|----------:|-------:|
| Sha256   | 4096  | 12.34 us  | 0.18 us  | 0.16 us  | 1.00  |     32 B  |      - |
| XxHash3  | 4096  |  0.87 us  | 0.02 us  | 0.02 us  | 0.07  |      0 B  |      - |
```

Columns:

- **Mean** — average per invocation.
- **Error** — half-width of the 99.9% confidence interval. Always quote `Mean ± Error`.
- **StdDev** — sample standard deviation. Big StdDev = unstable benchmark; investigate before trusting Mean.
- **Median** — robust to outliers; pair with Mean for skewed distributions.
- **Ratio** — vs the `[Baseline = true]` method, *within the same parameter combination*. Without a baseline, the column doesn't appear.
- **Allocated** — bytes allocated per op, requires `MemoryDiagnoser`.
- **Gen0/1/2** — collections per 1000 ops, requires `MemoryDiagnoser`.

## CI integration

```bash
# Filter to a specific class, JSON output, store as artifact.
dotnet run -c Release --project bench -- \
  --filter '*HashBench*' \
  --exporters json \
  --artifacts ./bench-out
```

Cross-reference [`performance-baselines-and-release-gates.md`](./performance-baselines-and-release-gates.md) for the full gate flow:

1. Store JSON results as a build artifact.
2. On PR, diff against the baseline JSON from `main`.
3. Fail the PR if any benchmark regresses by more than a configured tolerance (e.g. Ratio > 1.05 *and* delta > 3× StdDev — the StdDev clause keeps noise from triggering false alarms).

A CI failure that isn't reproducible on a developer's machine usually means the CI runner is shared or has variable CPU governance — pin the runner or accept a wider tolerance.

## Anti-patterns BDN catches (or doesn't)

```csharp
// BAD — return value discarded, JIT eliminates the entire body.
[Benchmark] public void DoWork() { var x = 1 + 1; }

// GOOD — return the result, BDN's Consumer prevents elimination.
[Benchmark] public int DoWork() => 1 + 1;

// BAD — Console.WriteLine dwarfs the work and adds wildly variable IO.
[Benchmark] public int DoWork() { Console.WriteLine("hi"); return 1; }

// BAD — cross-iteration state mutation; results depend on order.
[Benchmark] public int Add() => _list.Count; // _list grows in another benchmark

// GOOD — reset per-iteration with [IterationSetup] only when you need to,
// understanding that it disables BDN's amortization (see gotchas).

// BAD — measuring constant folding.
[Benchmark] public int Const() => Math.Abs(-42);

// GOOD — use [Arguments] or [Params] to defeat folding.
[Benchmark] [Arguments(-42)] public int Abs(int x) => Math.Abs(x);
```

## Senior-level gotchas

- **Debug mode is rejected by design.** If you somehow trick it into running (e.g. via `[DryJob]` for testing the harness itself), the numbers are nonsense. Always Release.
- **`[GlobalSetup]` runs once per `[Params]` value combination**, not once per benchmark method. If your setup is expensive and shared across methods, that's the right level. If you need fresh state per iteration, use `[IterationSetup]` — but note that it **disables BDN's invocation-batching**, often dropping accuracy for sub-microsecond methods. The tradeoff is real.
- **`[MemoryDiagnoser]` slightly perturbs timing** by forcing `GC.WaitForPendingFinalizers()` between iterations to get clean per-op allocation numbers. If you need both ultra-precise timing and allocation data, run two passes — one without and one with the diagnoser.
- **Tiered JIT is disabled in BDN jobs by default**, which is realistic for steady-state but pessimistic for cold-start. To benchmark first-call latency, use `Job.Default.WithRuntime(...).WithEnvironmentVariable("DOTNET_TieredCompilation", "1")` and a cold-start specific harness.
- **Cross-machine numbers are not comparable** — even with the same .NET version, a different CPU, turbo state, or kernel can swing results 2–10×. CI gates should compare against a baseline captured on the **same** runner.
- **Pinned CPU governance matters.** On laptops and dynamic-frequency cloud VMs, results drift by 20–30% across runs. Disable Turbo Boost and pin to "High Performance" power plan when accuracy matters.
- **`BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args)`** is the right entry point for projects with multiple benchmark classes — it lets the user pick at the command line via `--filter`.
- **`[Benchmark]` returning `Task` is supported** and BDN will await it, but the overhead of `async` machinery is part of what's measured. For pure async-overhead studies use `ValueTask` returns and compare against a sync baseline; see [`async-await-performance.md`](./async-await-performance.md).
- **`[Params]` with reference types** uses the type's `ToString()` for the report column. Override it (or use a `record`) so the table is readable.
- **The `Allocated` column counts the benchmarked method's allocations only** — allocations in `[GlobalSetup]` are excluded. If a "zero allocation" claim depends on something the setup pre-built, say so explicitly.
- **`[ParamsAllValues]` on an enum runs every enum member** — including `None`/`Default` which often takes a degenerate fast path and skews the table. Filter to the values you actually want.
- **Micro-benchmarks lie about cache behavior.** A tight loop that hits L1 in BDN may hit L3 or memory in production where the surrounding workload pollutes the cache. Always pair a micro-benchmark win with an end-to-end measurement before claiming the change matters in prod.
- **Don't benchmark `try`/`catch` cost without throwing.** The cost of `try`/`catch` *without an exception* is near zero on modern .NET. The cost when an exception is thrown is dominated by stack-walk and metadata lookup, not the catch frame. Design the benchmark to reflect the actual production frequency of throws — typically very rare.
- **`DisassemblyDiagnoser` is the truth-teller** when two algorithmically-identical methods benchmark differently. Look at the asm, find the missing inline or the unexpected boxing, and the mystery resolves.
