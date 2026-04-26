# Performance baselines and release gates

_Targets .NET 10 / C# 14, BenchmarkDotNet 0.14+, k6 0.50+. See also: [BenchmarkDotNet](./benchmarkdotnet.md), [Load testing](./load-testing.md), [Performance regression tracking](./performance-regression-tracking.md), [dotnet-counters](./dotnet-counters.md), [.NET performance anti-patterns](./dotnet-performance-anti-patterns.md), [Allocations and boxing](./allocations-and-boxing.md), [GC pressure](./gc-pressure.md)._

A **baseline** is a frozen set of performance numbers (mean latency, allocations per op, p95/p99, throughput) for each named scenario, captured under controlled conditions and stored in version control. A **release gate** is a CI rule that compares a candidate build against the baseline and fails the pipeline when divergence exceeds a tolerance. Together they convert "perf is the team's responsibility" — which dilutes to nobody's responsibility — into "perf is a build-time check", which has a clear owner: whoever wrote the failing PR.

Without gates, performance erodes one merge at a time. Each individual PR adds 2% — a single regression that nobody would block — and a year later p99 is twice what it was. Baselines are how you make the 2% visible.

## Two kinds of baseline, two kinds of gate

| | Micro baseline | Macro baseline |
|---|---|---|
| Tool | [BenchmarkDotNet](./benchmarkdotnet.md) | [k6 / NBomber](./load-testing.md) |
| Unit | Per method, per parameter combo | Per scenario / endpoint |
| Metrics | Mean ns/op, Allocated bytes, Gen0/1/2 collections | p50/p95/p99 latency, RPS sustained, error rate |
| Noise floor | < 3% on a pinned runner | 5-15% even on dedicated infra |
| Cost | Seconds to minutes | 10-60 minutes per run |
| Where to gate | Every PR | Nightly / release branch |
| Tolerance shape | Ratio + StdDev clause | SLO threshold + delta-vs-baseline |

The two are gated **differently** because their noise floors differ. A 5% p99 swing on k6 is normal; a 5% mean swing in BDN on a pinned runner is a real change. Sharing one tolerance across both produces either false alarms (BDN tolerance too loose) or missed regressions (k6 tolerance too tight).

## Capturing a baseline

The baseline is a file, not a memory. Store it next to the code:

```
/perf
  /baselines
    bdn/HashBench-net90.json        # checked-in BDN JSON export
    bdn/SerializerBench-net90.json
    k6/checkout-p99.json            # exported summary { p99: 380, p95: 142, rps: 1023, ... }
  scripts/
    capture-baseline.sh             # run on the perf runner, copies results in
    diff-baseline.sh                # used by CI gates
```

Capture rules:

1. **Pinned runner**, identical hardware to the gating runner. CPU governance set to `performance` (Linux) or "High Performance" (Windows). Turbo behaviour pinned via BIOS or `cpupower frequency-set -g performance`.
2. **Locked dependencies** at capture — same `global.json`, same NuGet lock file, same base image as the gate.
3. **Captured on `main` after a merge**, never on a branch. The baseline reflects what's deployed, not what's proposed.
4. **At least 3 capture runs**, kept and averaged. A single noisy run becomes a baseline that punishes the next PR for no reason.
5. **Re-baseline only on intentional changes**: an explicit perf-improving commit, a major dependency upgrade, a runtime upgrade (.NET 9 → .NET 10). Drift-baselining ("the number got worse, update the baseline") is the single fastest way to lose the value of the system.

```bash
# capture-baseline.sh — run on the perf runner.
set -euo pipefail
dotnet run -c Release --project bench -- \
  --filter '*' \
  --exporters json \
  --artifacts ./bdn-out

# Copy each JSON into the repo's baseline folder.
cp bdn-out/results/*report-full-compressed.json perf/baselines/bdn/
git add perf/baselines/bdn/
git commit -m "perf: refresh BDN baseline after .NET 10 upgrade"
```

Every baseline-refresh commit must explain *why*. A future engineer staring at "the gate failed because p99 went from 320 ms to 350 ms" needs to be able to find the commit that legitimised today's number.

## Tolerance design — ratio AND delta

The naive gate is "fail if mean increased > 5%". On its own it triggers constantly when a benchmark is intrinsically noisy or when CI runners drift. The robust gate is **both**:

```
regression =
    (candidate.Mean / baseline.Mean) > 1.05   AND
    (candidate.Mean - baseline.Mean) > 3 * baseline.StdDev
```

The ratio clause says "at least 5% slower". The StdDev clause says "and that's beyond the noise observed when the baseline was captured." Either alone is wrong:

- Ratio alone fires on benchmarks where the mean is microseconds and StdDev is half the mean (lots of micro-benchmarks live there).
- StdDev alone misses small-but-real regressions on very stable benchmarks.

For allocations, ratio is enough — `Allocated` is deterministic. A regression from 0 B to 24 B is **infinite** ratio but should clearly fail; gate it as "any non-zero increase" rather than ratio.

For macro/k6 percentiles, gate on **absolute SLO**, not ratio:

```
p99 < 400ms   AND   error_rate < 0.001
```

Ratio-vs-baseline at the macro level is too noisy to be the primary gate. SLO is what the user feels — gate there. Use ratio-vs-baseline as a *trend* signal in regression tracking (see [`performance-regression-tracking.md`](./performance-regression-tracking.md)) rather than a hard build break.

## CI snippet — BDN gate on every PR

```yaml
# .github/workflows/perf-gate.yml
name: perf-gate
on: pull_request

jobs:
  bdn:
    runs-on: [self-hosted, perf-pinned]   # critical: not ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Run BDN
        run: |
          dotnet run -c Release --project bench -- \
            --filter '*' --exporters json \
            --artifacts ./bdn-candidate

      - name: Diff vs baseline
        run: |
          dotnet tool restore
          dotnet results-comparer \
            --base perf/baselines/bdn \
            --diff bdn-candidate/results \
            --threshold 5%   \
            --noise 3        \
            --out-md regression-report.md
        # results-comparer is the BDN companion (Microsoft.Crank.Models / dotnet/performance);
        # any diffing tool that supports ratio + stddev gating works.

      - name: Comment on PR
        if: always()
        uses: thollander/actions-comment-pull-request@v2
        with:
          filePath: regression-report.md
```

The gate is the `results-comparer` exit code. Fail-closed: missing baseline file aborts the build and asks the developer to capture one rather than silently passing.

## What to gate where

| Stage | Gate | Why |
|---|---|---|
| PR | BDN micro on changed projects only | Cheap, < 5 min, blocks obvious regressions |
| PR (label-driven) | Full BDN suite | When the diff touches hot paths; opt-in via `perf` label |
| Nightly on `main` | Full BDN + smoke load test | Catches slow drift and merge-skew regressions |
| Pre-release | Full load test (10-30 min sustain) at SLO | Hard gate before tagging release |
| Release | Soak test (4 h) | Catches leaks; not a build gate but a deploy gate |

Don't run k6 / macro tests on every PR. The runner cost is real and the noise floor is too high to inform a per-commit gate.

## Soft vs. hard gates

A hard gate fails the build. A soft gate posts a warning on the PR but lets it merge. The right mix:

- **Hard**: SLO violations on the macro side, allocation regressions on micro (deterministic), explicit-budget benchmarks (e.g. "this hot loop is allocation-free"). These are categorical: fail fast, force a conversation.
- **Soft**: timing regressions in the 5-10% range on PRs not labelled `perf`. Comment, but allow merge. This avoids weaponising the perf gate against unrelated PRs that nudge a barely-loaded benchmark.
- **Hard, escalating**: timing regression > 25% always blocks; > 10% blocks unless overridden by a maintainer label.

Tunable thresholds live next to the baseline (`perf/gate-config.yml`), reviewed like any other policy file.

## Re-baselining workflow

A legitimate baseline refresh:

1. The candidate change is **explicitly improving or accepting** a perf shift.
2. The PR description says so, with the new numbers and the reason.
3. A dedicated commit *only* updates the baseline JSON (`git mv`-style — not mixed with code changes), so a future blame on "why is the baseline this number?" lands on a clear commit.
4. The baseline-refresh commit is reviewed by someone other than the author.

A baseline that drifts silently because everyone keeps clicking "override" is worse than no baseline.

## Senior-level gotchas

- **GitHub-hosted runners are the wrong place to gate on micro perf.** Their CPU is shared, frequency varies wildly, and consecutive identical jobs can swing 30%+. Gate micro perf on a self-hosted, pinned, frequency-locked runner. If you can't have one, soft-gate only.
- **"Everything regressed by 5%" almost always means the runner changed.** A new kernel, a new CPU model, a new hyper-threading setting — investigate the *runner* before investigating the code.
- **`Mean` is the wrong metric to gate on for skewed distributions.** Use `Median` for benchmarks that occasionally have a slow iteration (e.g. anything that may trigger a Gen2 GC). BDN reports both — gate on Median for noisy benchmarks and Mean for clean ones.
- **Ratio AND StdDev — both clauses, no shortcut.** A regression that exceeds 3× StdDev but is < 5% is statistically real but operationally negligible. A 5% regression within 1× StdDev is noise. Require both.
- **Allocation regressions are categorical, not ratio.** Going from 0 B to 8 B per op is a 100% real regression even though the ratio is infinite. Gate allocations as a strict-non-increase budget on hot paths.
- **Percentile gating beats average gating for SLO endpoints.** A change that shifts 1% of requests from 50 ms to 5 s leaves the mean nearly unchanged but breaks p99. Always gate p95 + p99, never mean alone, on user-facing endpoints.
- **Don't gate on absolute numbers across runners.** "p99 < 400 ms" means different things on a 16-core perf runner and an 8-core CI runner. Either pin the runner or gate on ratio-vs-baseline-captured-on-this-runner.
- **`[IterationSetup]` accuracy trade-off bites here.** A benchmark that uses `[IterationSetup]` has lower precision; calibrate its tolerance wider and accept a noisier gate, or refactor to `[GlobalSetup]` if state can be shared.
- **Beware of `OutlierMode.RemoveAll` becoming a coverup.** BDN drops outliers by default. If a *real* regression manifests as more outliers (e.g. a new GC pause), removing them hides the regression. Inspect raw measurements when "Mean barely moved but the service feels slower."
- **The Mann-Whitney U / signed-rank test beats t-test on benchmark data.** Latency distributions are not normal. If you have access to per-iteration data (`-m` flag in BDN), prefer non-parametric tests for the gate decision; if you don't, the ratio + StdDev heuristic above is the practical compromise.
- **Snapshots get stale in dependency-heavy services.** A baseline captured against an old service-bus client may be invalid after a client upgrade. When you bump a major dependency, refresh the baseline in the same PR or accept a noise spike that you'll have to chase later.
- **Don't gate the baseline-refresh PR against itself.** The CI rule must read the *previous* baseline and skip the gate when only baseline files changed (or compare the new baseline against the previous one as a sanity check). Otherwise refreshing always passes by definition.
- **Test the gate by intentionally regressing.** A perf gate that has never failed is a perf gate you don't trust. Once a quarter, push a known-bad PR and verify the gate catches it; this also revalidates the runner is still pinned.
- **The release-gate output goes to humans.** A gate failure that says "ratio 1.07 exceeds 1.05" is useless on its own. The CI comment must show: which benchmark, baseline numbers, candidate numbers, the runner host, the linked baseline-refresh commit, and a link to instructions for either fixing or legitimately refreshing.
