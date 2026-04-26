# Performance regression tracking

_Targets .NET 10 / C# 14, BenchmarkDotNet 0.14+, crank 0.2.x. See also: [BenchmarkDotNet](./benchmarkdotnet.md), [Load testing](./load-testing.md), [Performance baselines and release gates](./performance-baselines-and-release-gates.md), [dotnet-counters](./dotnet-counters.md), [dotnet-trace](./dotnet-trace.md), [memory leaks in .NET](./memory-leaks-in-dotnet.md), [GC pressure](./gc-pressure.md)._

A regression isn't "today's number is bigger than yesterday's." It's a **trend that lifts a metric out of its historical noise band** and is attributable to a specific commit (or window of commits). Without tracking, the gates from [`performance-baselines-and-release-gates.md`](./performance-baselines-and-release-gates.md) only catch step changes — anything that drifts in over a month slips through, because each PR alone is below the gate threshold.

This file is about the **system around the gate**: how to capture, store, analyse, and alert on perf data over time so that drift is visible long before any single PR exceeds tolerance.

## The pipeline

```
+--------+     +--------+     +----------+     +--------+
| Capture| --> | Store  | --> | Analyse  | --> | Alert  |
+--------+     +--------+     +----------+     +--------+
   BDN/k6        Time DB      ChangePoint /     Slack /
   JSON,         or git       signed-rank /     PR check
   crank         branch       3σ window         dashboard
```

Each stage has well-known choices. The trap is over-engineering — a `git`-tracked JSON folder and a Grafana dashboard reading from it covers 80% of teams. A full timeseries DB only pays off when you have hundreds of benchmarks and dozens of runners.

## Capture

| Source | Format | Cadence |
|---|---|---|
| BDN | `BenchmarkDotNet.Artifacts/results/*-report-full-compressed.json` | Every PR + every `main` merge |
| k6 | `--out json=results.json` (or `--out experimental-prometheus-rw`) | Nightly + on `release/*` |
| NBomber | `WithReportFormats(ReportFormat.Csv, ReportFormat.Md)` + custom JSON sink | Nightly |
| Crank | `crank.exe --output results.json` | Nightly (ASP.NET Core macro) |
| Production | OTel `Meter` histograms scraped to Prometheus | Continuous |

`crank` is the tool the .NET team uses internally to drive ASP.NET Core benchmark scenarios end-to-end (TechEmpower-style). It owns process lifecycle, agent coordination, and metric scraping. Worth knowing about even if you don't adopt it — its scenario format is a reasonable model for "macro benchmarks living in source control."

Tag every captured result with: commit SHA, branch, runner host, .NET SDK version, base image digest, capture timestamp. Without these, a regression alert that fires three weeks later is unactionable.

## Store

Two viable patterns. Pick one and stick with it.

### Pattern A — Git-tracked JSON (smallest team, lowest ops)

```
/perf-data
  /bdn
    abc1234.json    # one file per main commit, named by SHA
    def5678.json
  /k6
    nightly-2026-04-25.json
    nightly-2026-04-24.json
```

A side branch (`perf-data` or `perf/results`) accumulates these files. Grafana scrapes a JSON-over-HTTP endpoint that reads from `git`. Storage is cheap, history is intrinsic, and reverting a bad data write is a `git revert`. Caps out around 10 000 files before `git` operations get slow.

### Pattern B — Timeseries database

```
benchmark{name="HashBench.Sha256",size="4096",runtime="net90"} 12340  # mean ns/op
allocated{name="HashBench.Sha256",size="4096",runtime="net90"} 32     # bytes
```

Push results to Prometheus (via remote-write), InfluxDB, or TimescaleDB. Grafana reads natively. Better at scale, worse at provenance — losing a database doesn't undo a code commit.

Either way, **do not** keep results only in CI artifacts. GitHub Actions retention is 90 days by default; trends are 12-month conversations.

## Analyse — three useful methods

### 3σ on a rolling window

For each (benchmark, metric), compute the rolling mean and StdDev over the last 14 days of `main` commits. Flag a commit whose value falls more than **3σ** outside that window. Robust to single noisy runs, sensitive to genuine step changes.

```python
# Pseudocode for the alerting cron.
window = results.tail(14_days)
mean, sd = window.mean(), window.std()
latest = results.iloc[-1]
if abs(latest.value - mean) > 3 * sd:
    alert(f"{latest.benchmark}: {latest.value} vs {mean}±{sd}")
```

3σ on a 14-day window is the right default for daily-cadence captures. For per-PR data, the noise floor is too high — go to 4σ or use ChangePoint detection instead.

### ChangePoint detection

A statistical test (PELT, Bayesian online ChangePoint, or simply `scipy.stats.ks_2samp` on two windows) that asks "is there a step change in this series at point X?" Better than 3σ when the distribution shifts but doesn't make any single point an outlier — the classic "everything is 8% slower starting tuesday" pattern.

Microsoft's `dotnet/performance` repo runs ChangePoint detection over BDN history and files GitHub issues automatically when a step change is detected. The same idea works at any scale; `BenchmarkDotNet.Tool` and the `ResultsComparer` companion ship the building blocks.

### Mann-Whitney U / Wilcoxon signed-rank

When you have *raw per-iteration measurements* (BDN `--full-record` or per-request k6 output), don't compare means. Compare distributions with a non-parametric test. Latency data is rarely normal — t-test gives wrong p-values, Mann-Whitney U doesn't.

This is the test that powers BDN's `--statistical-test` flag (e.g. `--statisticalTest=10ms` for "are these distributions different by at least 10 ms?").

## Alert

Three audiences, three channels:

| Audience | Channel | Cadence |
|---|---|---|
| The author of the regressing commit | GitHub PR check / commit comment | Per-commit |
| The team owning the affected component | Slack channel | When 3σ alert fires |
| The platform / perf-engineering group | Dashboard with all benchmarks | Continuous |

Don't alert humans on every per-PR result — that's the gate's job. The tracking system alerts on **trends** that the gate isn't catching. False-positive alerts kill the system: people learn to ignore them, and three months later the real regression is also ignored.

## Synthetic vs. production drift

| Synthetic regression tracking | Production telemetry drift |
|---|---|
| BDN, k6, crank | OTel histograms, App Insights, Prometheus |
| You control the workload | Workload changes by user behaviour |
| Stable, reproducible | Noisy, but real |
| Catches code changes | Catches code + traffic shape + downstream changes |
| Owned by perf engineering | Owned by SRE / oncall |

Both matter. A synthetic graph that's flat but a production graph that climbs means traffic shape changed (more big payloads, longer tail of slow downstream calls, scale-out reducing CPU per pod) — code is fine, capacity isn't. The reverse — synthetic regresses, prod is flat — usually means the regression is in a hot path that isn't actually hit at the prod traffic mix; deal with it but don't panic.

Keep the dashboards separate. Mixing them tempts everyone to "explain away" synthetic regressions with prod data ("it's fine in real traffic") and the slow rot resumes.

## Attribution — bisecting a regression window

If the alert fires across a window of N commits, `git bisect run` finds the offender automatically:

```bash
#!/usr/bin/env bash
# bisect-perf.sh — exit 0 if benchmark within tolerance, 1 otherwise.
set -euo pipefail

dotnet run -c Release --project bench -- \
  --filter '*HashBench.Sha256*' \
  --exporters json --artifacts ./bisect-out \
  --warmupCount 3 --iterationCount 5  # smaller for speed; precision OK during bisect

mean=$(jq '.Benchmarks[0].Statistics.Mean' bisect-out/results/*.json)
threshold=15000  # ns; a known-good upper bound for this benchmark

awk -v m="$mean" -v t="$threshold" 'BEGIN { exit (m > t) ? 1 : 0 }'
```

```bash
git bisect start HEAD known-good-sha
git bisect run ./bisect-perf.sh
```

`git bisect run` exits at the first bad commit. Caveats: noisy benchmarks confuse bisect (a single run might fail on a "good" commit by chance), so script the inner check to retry 3× and take the median, or use a ChangePoint window directly on the captured data and skip bisect entirely.

## Production-side: OTel histograms feeding Grafana

The custom counters from [`dotnet-counters.md`](./dotnet-counters.md) are also OpenTelemetry instruments — register them once, expose them via the OTel Prometheus exporter, and Grafana sees them directly:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(b => b
        .AddMeter("Acme.Orders")            // your custom meter
        .AddRuntimeInstrumentation()        // gen0/1/2, alloc-rate, threadpool
        .AddAspNetCoreInstrumentation()     // request duration histogram
        .AddHttpClientInstrumentation()     // outbound httpclient
        .AddPrometheusExporter());
```

Grafana panel queries to keep:

```promql
# p99 request latency by route, 5-minute window.
histogram_quantile(0.99, sum by (le, http_route) (
  rate(http_server_request_duration_seconds_bucket[5m])
))

# alloc-rate (bytes/sec) trend, daily average.
avg_over_time(dotnet_gc_allocations_size_bytes_total[1d])

# gen-2 collection rate.
rate(dotnet_gc_collections_total{generation="2"}[5m])
```

Wire alerts on these the same way as synthetic — 3σ over a rolling baseline, not absolute thresholds. Absolute thresholds are SLOs, which the [`load-testing.md`](./load-testing.md) gate already covers.

## Senior-level gotchas

- **Latency distributions are not normal.** Mean ± StdDev is a rough proxy; for trustworthy regression tests use Mann-Whitney U or signed-rank on raw iteration data. BDN's `--statistical-test` is exactly this.
- **3σ on a 14-day window beats per-PR alarms.** Single-PR alerts fight the gate; the tracking system's value is in trends that the gate misses. Calibrate sensitivity for "drift over weeks", not "this PR is bad."
- **GC-induced bimodality wrecks averages.** A benchmark that occasionally trips a Gen2 GC has a 99% fast / 1% slow distribution. The mean tells you nothing; track Median + p99 separately and alert on each. See [`gc-pressure.md`](./gc-pressure.md).
- **"Everything regressed at 02:00 UTC" usually means the runner rotated.** Cloud CI runners get re-provisioned; a new VM means a new CPU and a 5-15% baseline shift. Tag every result with the runner host and filter trends by it.
- **Micro-benchmarks are not a substitute for SLO tracking.** A green BDN dashboard with red production p99 means your synthetic surface doesn't match the real hot path. Sample real traffic into a synthetic scenario periodically (replay) — k6 supports CSV-driven user data.
- **Alert fatigue kills tracking systems.** 30 alerts per day get muted, then real ones get missed. Each new benchmark should justify being on the alert path; the rest go on the dashboard only. A regression-tracking system with 5 always-actionable alerts beats one with 50 noisy ones.
- **Archive baselines forever.** A regression report that says "p99 was 280 ms last June" is invaluable a year later when capacity planning. Don't roll your storage at 90 days.
- **Don't confuse a metric change with a benchmark change.** If a benchmark's StdDev suddenly doubled but Mean didn't move, *something changed* — runner contention, a new background process, a kernel update. Investigate StdDev shifts the same way you investigate Mean shifts.
- **Prod-side regression often shows in `requests-queue-duration` first.** Code that doesn't allocate more or compute more but holds connections longer manifests upstream as queue growth before it shows as latency. Track [`System.Net.Http`](./connection-pooling.md) `requests-queue-duration` as a perf metric, not just a health metric.
- **A regression in tail latency is invisible at 1-second resolution.** OTel histograms with 1-minute scrape intervals smooth a 200 ms GC pause into nothing. Use `summary` or short-interval histograms (1-5 s scrape) for the percentile metrics that matter.
- **`git bisect run` with a noisy benchmark gives wrong commits.** Bake retry+median logic into the bisect script, set `git bisect skip` for ambiguous results, and validate the answer by running the suspected commit and its parent ten times each.
- **A regression that "isn't really a regression" still costs.** If a PR adds 5% latency for a feature the team agrees is worth it, document the trade-off in the baseline-refresh commit. Future reads of the trend dashboard otherwise look like ongoing rot.
- **The team most likely to break perf is the team that owns it.** A perf engineering team that owns the tracking system but not the code under test ends up shouting into the void. Wire alerts to the *code owners* (CODEOWNERS-driven Slack routing), not to a central perf channel.
- **Track the harness as carefully as the SUT.** A k6 minor upgrade or a BDN configuration change can shift every number simultaneously. Pin the harness version in `global.json` / Docker tag, and treat upgrading it as a re-baseline event.
- **Consider the cost of *not* alerting.** A leak that compounds 0.5% per day is a 17% regression after a quarter and 80% after a year. The tracking system's worst failure mode isn't false positives — it's silent rot, where every individual PR was below threshold and you only notice when oncall pages.
