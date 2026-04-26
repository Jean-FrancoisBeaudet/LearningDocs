# Load testing

_Targets .NET 10 / C# 14, k6 0.50+, NBomber 5.x. See also: [BenchmarkDotNet](./benchmarkdotnet.md), [Performance baselines and release gates](./performance-baselines-and-release-gates.md), [Performance regression tracking](./performance-regression-tracking.md), [dotnet-counters](./dotnet-counters.md), [backpressure handling](./backpressure-handling.md), [connection pooling](./connection-pooling.md), [memory leaks in .NET](./memory-leaks-in-dotnet.md), [async/await performance](./async-await-performance.md), [service mesh latency](./service-mesh-latency.md)._

Load testing exercises the **whole system** under representative traffic. Where [BenchmarkDotNet](./benchmarkdotnet.md) tells you whether method A is faster than method B in isolation, a load test tells you whether the deployed service holds its p99 SLO at 5 000 RPS with realistic data, real network, and real downstream services. The two are complementary: a BDN win that doesn't move the load-test number is usually a micro-optimisation that hits cache or I/O before it ever helps.

It is **not** a unit test, **not** a profiler, and **not** a substitute for production telemetry. Load tests are synthetic — the traffic shape, payload mix, and downstream behaviour are approximations. Their value is comparative (today vs. yesterday, with-fix vs. without) far more than absolute.

## What load testing answers

| Question | The right test shape |
|---|---|
| Does my service still meet its SLO under expected peak traffic? | **Load** — sustained at peak RPS for 15-60 min |
| What's the breaking point — where does latency cliff or error rate climb? | **Stress** — ramp until something gives |
| Does anything leak / drift / fragment over hours? | **Soak** — steady moderate load for 4-24 h |
| Can the autoscaler keep up with a traffic spike? | **Spike** — instant 10× ramp, then back down |
| Does the deploy pipeline itself work end-to-end? | **Smoke** — 1-5 RPS for 60 s; no SLO assertion |

Pick the shape *before* writing the script. Reusing one shape (e.g. always running stress when you need soak) gives misleading data — a stress test that finishes in 5 min won't catch a 6-hour heap-fragmentation leak.

## Closed-model vs. open-model load

This single distinction explains more bad load-test data than any other:

- **Closed model** (default in JMeter, Gatling, NBomber's `KeepConstant`): N virtual users (VUs) loop sending requests; each VU's next request waits for its previous response. **Slow service → less load** — the test self-throttles, and reported p99 looks artificially clean. This is the **coordinated-omission** problem: the latency the script *would have* observed is silently dropped.
- **Open model** (k6 `arrivalRate`, Vegeta, wrk2): the load generator commits to firing N requests per second regardless of how the server responds. Slow responses queue up, p99 reflects the actual user-visible latency, and pile-up failures appear in the metrics.

```js
// k6: open-model — fires 1000 RPS no matter how slow the target is.
export const options = {
  scenarios: {
    constant_rps: {
      executor: 'constant-arrival-rate',
      rate: 1000,            // requests per timeUnit
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 200,  // initial pool
      maxVUs: 2000,          // grow if responses queue
    },
  },
};
```

```js
// k6: closed-model — 200 VUs, each waits for its previous response.
// Use this only when you literally model 200 concurrent users in a session.
export const options = {
  scenarios: {
    fixed_users: {
      executor: 'constant-vus',
      vus: 200,
      duration: '5m',
    },
  },
};
```

User-facing HTTP traffic is almost always best modeled as **open**. Background-job pools and connection-bound work (a worker that consumes a queue with a fixed concurrency cap) are **closed**.

## The three tools to know

| Tool | Strengths | Weaknesses |
|---|---|---|
| **k6** (Grafana) | First-class open model, JS scripts, native gRPC/WebSocket, HDR-style percentiles, Grafana Cloud integration | JS not C#; Go runtime; large script extension story |
| **NBomber** (.NET-native) | C# scenarios, IDE debugger, xUnit/CLI runners, plug-and-play with EF / gRPC / Kafka clients you already use | Closed model is its default — open mode (`Inject`) needs care |
| **Azure Load Testing** | Managed runners (no self-hosted infra), JMeter or k6 scripts, integrated with Azure Monitor / App Insights | Per-test pricing; fewer regional runners than self-hosted; opaque agent CPU |
| Legacy: JMeter, wrk, Bombardier, Gatling, Vegeta | Mature; some niche use (Vegeta = great open-model HTTP for Go fans, wrk2 = open-model upgrade of wrk) | Ecosystem fragmentation; .NET teams gain little over k6/NBomber today |

For a .NET team standardising on one tool: **k6 for HTTP/gRPC**, **NBomber when the scenario needs in-process .NET helpers** (e.g. signing requests with the same DI-registered handler the prod app uses).

## k6 example — gated against SLOs

```js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    main: {
      executor: 'ramping-arrival-rate',
      startRate: 100,
      timeUnit: '1s',
      preAllocatedVUs: 200,
      maxVUs: 4000,
      stages: [
        { duration: '2m', target: 1000 },  // ramp up
        { duration: '10m', target: 1000 }, // sustain
        { duration: '1m', target: 0 },     // ramp down
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.001'],                  // < 0.1% errors
    http_req_duration: ['p(95)<150', 'p(99)<400'],    // p95 < 150ms, p99 < 400ms
    'http_req_duration{name:checkout}': ['p(99)<800'],// per-tag SLO
  },
  discardResponseBodies: true, // saves a lot of allocations on the runner
};

export default function () {
  const res = http.post('https://api.example.com/orders',
    JSON.stringify({ sku: 'abc-123', qty: 1 }),
    { headers: { 'Content-Type': 'application/json' }, tags: { name: 'checkout' } });
  check(res, { 'status 201': (r) => r.status === 201 });
}
```

`thresholds` are the gate. If any one fails, k6 exits non-zero — wire that into CI and a failing build means the change missed the SLO. Cross-reference [`performance-baselines-and-release-gates.md`](./performance-baselines-and-release-gates.md) for the full release-gate flow.

## NBomber example

```csharp
using NBomber.CSharp;
using NBomber.Http.CSharp;

var http = HttpClientFactory.Create();

var scenario = Scenario.Create("checkout", async ctx =>
{
    var req = Http.CreateRequest("POST", "https://api.example.com/orders")
        .WithHeader("Content-Type", "application/json")
        .WithBody(new StringContent("""{"sku":"abc-123","qty":1}"""));

    var res = await Http.Send(http, req);
    return res.IsError ? Response.Fail() : Response.Ok();
})
.WithLoadSimulations(
    // Open-model injection: 1000 RPS for 10 minutes.
    Simulation.Inject(rate: 1000, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(10)));

NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFolder("./load-results")
    .WithReportFormats(ReportFormat.Md, ReportFormat.Csv, ReportFormat.Html)
    .Run();
```

`Simulation.Inject` is the open-model entry. `KeepConstant` is closed-model (N concurrent users) — pick deliberately. NBomber returns a non-zero exit code when scenario step errors or threshold violations occur, so CI integration is the same shape as k6.

## What to measure during the run

A load test that only records client-side latency hides half the story. Always pair the run with **server-side counters** captured at the same time.

| Side | Signal | Tool |
|---|---|---|
| Client | RPS, error rate, p50/p95/p99 latency, request duration histogram | k6 / NBomber report |
| Server | CPU %, working set, alloc-rate, gen-2 GCs, time-in-gc | [`dotnet-counters`](./dotnet-counters.md) |
| Server | Threadpool queue length, lock contentions/sec, exceptions/sec | [`dotnet-counters`](./dotnet-counters.md) |
| Server | `current-requests`, `requests-queue-duration` (HttpClient pool), DB pool waits | [`dotnet-counters`](./dotnet-counters.md), `Microsoft.AspNetCore.Hosting`, `System.Net.Http` |
| Infra | Network in/out, disk wait, container CPU throttling, downstream RTT | Prometheus / Azure Monitor |

The classic patterns:

- **Client p99 climbs while server CPU is < 50%** → blocked on I/O or downstream; check `requests-queue-duration` and downstream traces.
- **Threadpool queue length sustained > 0** → sync-over-async or blocking I/O; see [thread contention](./thread-contention.md).
- **Alloc-rate climbing without throughput gain** → GC pressure ([`gc-pressure.md`](./gc-pressure.md)) — the test is creating garbage faster than work.
- **Soak test working-set growing linearly over 4 h** → leak; capture a [`dotnet-dump`](./dotnet-dump.md) and diff against an early-soak snapshot.

## Test-environment realism

A load test against an empty database with one warm pod tells you nothing useful. Realistic enough means:

- **Data volumes** — same row counts, same index distribution as prod. A query that returns 10 rows in test and 10 M in prod has a different plan.
- **Network distance** — runner in the same region as the SUT (system under test) for steady runs; intentionally cross-region for a "what happens at 80 ms RTT" check. Don't mix.
- **Warmed JIT and tiered compilation** — discard the first 60-90 s of every run. Tier-1 promotion happens during warmup; the cold p99 is irrelevant to the steady-state SLO.
- **Connection pool warmth** — k6's `discardResponseBodies` and pre-allocated VUs help, but the *server's* `HttpClient` and DB pool also need warming. A separate 30-second warmup stage solves both.
- **Downstream realism** — a real DB or a faithful stub. A no-op mock makes the SUT look 100× faster than it ever will be.
- **Single shared runner host** — one k6 instance saturates network long before it saturates CPU on a modern box; if you're chasing > 50 000 RPS, the runner is often the bottleneck. Run distributed (k6 Cloud / Kubernetes operator) or accept the cap.

## CI integration

```yaml
# .github/workflows/load-test.yml — runs on nightly + manual dispatch only.
name: load-test
on:
  schedule: [{ cron: '0 3 * * *' }]
  workflow_dispatch:

jobs:
  k6:
    runs-on: [self-hosted, perf-pinned]   # pinned runner with stable CPU governance
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
      - name: Deploy SUT to perf env
        run: ./scripts/deploy-perf.sh
      - name: Warmup
        run: k6 run -e SCENARIO=warmup load/script.js --duration=60s --vus=10
      - name: Load test (gates on thresholds)
        run: k6 run --out json=results.json load/script.js
      - uses: actions/upload-artifact@v4
        with: { name: load-results, path: results.json }
```

A failing threshold in `script.js` exits non-zero and fails the workflow. For trend storage and alerts on slow drift (rather than hard SLO violation), feed `results.json` into the pipeline described in [`performance-regression-tracking.md`](./performance-regression-tracking.md).

## Senior-level gotchas

- **Closed-model load tests undersell p99.** If the SUT slows down, a closed-model test sends fewer requests; the dropped requests would have observed the worst latency. Open-model (`arrivalRate` / `Inject`) is the default unless you're literally modelling a fixed concurrent-user count.
- **The first 60-90 seconds are JIT and pool warmup.** Discard them. Reporting "average latency" over a run that includes cold start is a per-deploy slander.
- **DNS caching skews retry behaviour.** k6's resolver caches by default (good for steady state, bad for testing failover); the SUT's `HttpClient` cache may differ. Pin both or document the divergence.
- **Keep-alive masks connection-establishment cost.** If prod runs behind a proxy that forces short-lived connections, your test with persistent keep-alive is 2-3× rosier than reality. Mirror the prod proxy or set `--http-no-conn-reuse` (k6) for a worst-case run. See [connection pooling](./connection-pooling.md).
- **A single k6 / NBomber runner caps near network saturation, not CPU.** ~2 Gbps of HTTP traffic with payloads in the few-KB range is roughly 50-80k RPS per host. Above that, distribute the runners — and accept the runner-coordination overhead.
- **Soak tests catch what load tests can't.** A 30-minute load run is fine for SLO gating but useless for memory leaks, LOH fragmentation, timer-accumulation bugs, and connection-pool drift. Run a 4-24 h soak weekly; diff working-set, gen-2 size, and `active-timer-count`. See [memory leaks in .NET](./memory-leaks-in-dotnet.md).
- **`http_req_duration` in k6 is end-to-end including DNS + TLS + send + wait + receive.** If you want server processing time only, look at `http_req_waiting`. The two diverge when TLS handshakes dominate (frequent new connections).
- **Retries inflate RPS.** If a script silently retries on 5xx, your "achieved 1000 RPS" might be 600 unique + 400 retries. Set `redirects: 0` and explicit failure handling, and verify against server-side request count.
- **Average latency is meaningless under heavy tails.** A bimodal distribution (90% fast, 10% timing out at 30 s) shows a "good" mean and a catastrophic p99. Always gate on p95 + p99, never on mean.
- **1-second metric resolution hides bursts.** If the server GC pauses for 800 ms, a 1-second average drowns it. For burst diagnosis, drop k6's `summaryTrendStats` to higher-resolution percentiles and capture [`dotnet-trace`](./dotnet-trace.md) at the same time.
- **The load test is a system; treat it like one.** The test, the runner, and the SUT all evolve. Pin the script + runner image + k6 version together. A k6 minor upgrade has measurably changed scheduler behaviour before; an unrelated runner reboot can change kernel TCP defaults. Bake the harness into the baseline you compare against, not just the SUT version.
- **Don't load-test in prod by accident.** A misconfigured base URL is the most common load-test outage. Make scenarios fail closed: assert hostname starts with `perf.` or `staging.` before the first request.
- **Shared-tenant cloud DBs sandbag the result.** Azure SQL standard tiers, RDS general-purpose, Cosmos DB free-tier — all have noisy-neighbour behaviour that masks code wins. Run perf tests against an isolated SKU or accept ±20% noise in DB-bound scenarios.
- **CI auto-cancellation will kill long soaks.** GitHub Actions cancels jobs after 6 h on hosted runners. Self-hosted runners default to 24 h; if you exceed that, run as a long-lived workload in Kubernetes and ship results back via a webhook.
