# dotnet-counters

_Targets .NET 10 / C# 14, `dotnet-counters` 9.x. See also: [EventPipe](./eventpipe.md), [dotnet-trace](./dotnet-trace.md), [dotnet-dump](./dotnet-dump.md), [PerfView](./profiling-perfview.md), [GC pressure](./gc-pressure.md), [GC generations](./gc-generations-gen0-gen1-gen2.md), [thread contention](./thread-contention.md), [I/O saturation](./io-saturation.md), [BenchmarkDotNet](./benchmarkdotnet.md)._

`dotnet-counters` is the **lowest-friction observability tool** in the .NET diagnostics stack: a CLI that attaches to a running process over [EventPipe](./eventpipe.md) and prints live counter values — CPU, working set, GC heap size, allocations per second, lock contention, ASP.NET request rate. No admin, no instrumentation, no restart. It's the first tool to reach for when "the service is slow" and the second after `dotnet-trace ps` for any production triage.

It is **not** a metrics backend. There's no aggregation across instances, no retention, no alerting. For dashboards and historical data, scrape OpenTelemetry / Prometheus from your app. `dotnet-counters` is for the **next 60 seconds on this one process**.

## Where it fits

| Strength | Weakness |
|---|---|
| Zero setup; attaches to any .NET 5+ process | Per-process; no aggregation across replicas |
| No admin, no restart, no recompile | Polling-based — typical 1 s resolution, not microbenchmarks |
| Sees framework + ASP.NET + custom `Meter` counters | Not a substitute for a metrics pipeline |
| `collect` mode writes CSV/JSON for ad-hoc dashboards | Console UI is small — only fits ~30 counters at once |
| Works in containers (over the diagnostic socket) | Needs `TMPDIR` / pipe access — distroless images often need adjustments |

Reach for `dotnet-counters` when:

1. A service is misbehaving and you need a rough picture *now* — heap size, GC rate, exception rate, threadpool queue length.
2. You want to correlate a counter spike with a [trace capture](./dotnet-trace.md): "what did GC look like during the spike?".
3. You're iterating on a custom `Meter` instrument and want immediate feedback without standing up Prometheus.
4. You want to confirm a fix in a short loop (deploy, monitor, observe) without dashboards.

## Install and modes

```bash
dotnet tool install -g dotnet-counters

# 1. List PIDs.
dotnet-counters ps

# 2. Live monitor — stays attached until Ctrl-C, repaints in place.
dotnet-counters monitor -p 12345

# 3. Collect to file (CSV by default; JSON via --format json).
dotnet-counters collect -p 12345 \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting \
  --refresh-interval 1 \
  --duration 00:05:00 \
  --output ./counters.csv
```

`monitor` is interactive and ephemeral. `collect` writes a structured time series for offline analysis (Excel, pandas, jupyter). `ps` lists every .NET process the current user can attach to via diagnostic IPC.

## Built-in counters (`System.Runtime`)

The runtime ships ~20 counters under `System.Runtime`. Memorise the high-value ones:

| Counter | What it tells you |
|---|---|
| `cpu-usage` | Process CPU % |
| `working-set` | RSS in MB — physical memory the process holds |
| `gc-heap-size` | Total managed heap (Gen0+Gen1+Gen2+LOH+POH) MB |
| `gen-0-gc-count` / `gen-1-gc-count` / `gen-2-gc-count` | Collections per minute (a *rate*, not cumulative) |
| `gen-0-size` / `gen-1-size` / `gen-2-size` | Bytes per generation at last collection |
| `loh-size` | LOH bytes |
| `poh-size` | Pinned Object Heap bytes (.NET 5+) |
| `gc-fragmentation` | % of heap that's unused gaps |
| `time-in-gc` | % of last interval spent in GC |
| `alloc-rate` | Allocation rate, bytes/sec — best leading indicator of GC pressure |
| `threadpool-thread-count` | Worker threads currently in the pool |
| `threadpool-queue-length` | Pending work items — > 0 sustained means starvation |
| `threadpool-completed-items-count` | Items completed per minute (rate) |
| `monitor-lock-contention-count` | `Monitor.Enter` contentions per minute |
| `exception-count` | Managed exceptions thrown per minute (rate) |
| `active-timer-count` | Live `System.Threading.Timer` instances |
| `assembly-count` | Currently-loaded assemblies |
| `il-bytes-jitted` / `methods-jitted-count` | Cumulative — flat after warmup |

**Diagnostic shortcuts:**

- `alloc-rate` consistently above 1 GB/sec on a non-pipeline workload → look at [allocations and boxing](./allocations-and-boxing.md).
- `gen-2-gc-count` > 0 at a non-trivial rate → promotion pressure; see [GC generations](./gc-generations-gen0-gen1-gen2.md).
- `threadpool-queue-length` > 0 sustained → blocking sync work, see [thread contention](./thread-contention.md) and [async/await performance](./async-await-performance.md).
- `exception-count` > a few/sec sustained → exception-driven control flow somewhere; capture stacks with `dotnet-trace --providers Microsoft-Windows-DotNETRuntime:0x8000:5`.
- `time-in-gc` > 5% sustained → GC pressure problem ([gc-pressure.md](./gc-pressure.md)).

## ASP.NET / framework counters

```bash
dotnet-counters monitor -p 12345 \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting,Microsoft.AspNetCore.Server.Kestrel,System.Net.Http
```

| Provider | Notable counters |
|---|---|
| `Microsoft.AspNetCore.Hosting` | `requests-per-second`, `total-requests`, `current-requests`, `failed-requests` |
| `Microsoft.AspNetCore.Server.Kestrel` | `connections-per-second`, `current-connections`, `current-tls-handshakes`, `total-tls-handshakes` |
| `System.Net.Http` | `requests-started`, `requests-failed`, `current-requests`, `connections-current`, `requests-queue-duration` |
| `System.Net.NameResolution` | DNS resolutions and durations |
| `System.Net.Security` | TLS handshakes |
| `System.Net.Sockets` | bytes received/sent |
| `Microsoft.EntityFrameworkCore` | active DbContexts, queries, savechanges (when EF Core is used) |

`current-requests` rising while `requests-per-second` is flat is the classic backpressure signature — see [backpressure handling](./backpressure-handling.md). `requests-queue-duration` on `System.Net.Http` measures time spent waiting for a connection from the pool — see [connection pooling](./connection-pooling.md).

## Custom counters: `Meter` (preferred ≥ .NET 8)

```csharp
using System.Diagnostics.Metrics;

public sealed class OrdersMetrics : IDisposable
{
    private readonly Meter _meter = new("Acme.Orders", "1.0.0");
    private readonly Counter<long> _orders;
    private readonly Histogram<double> _priceMs;
    private readonly UpDownCounter<long> _inFlight;

    public OrdersMetrics()
    {
        _orders   = _meter.CreateCounter<long>("orders.placed", unit: "{order}");
        _priceMs  = _meter.CreateHistogram<double>("orders.price.duration", unit: "ms");
        _inFlight = _meter.CreateUpDownCounter<long>("orders.in_flight", unit: "{order}");
    }

    public void OrderPlaced() => _orders.Add(1);
    public void PriceLatency(double ms) => _priceMs.Record(ms);
    public void InFlight(long delta) => _inFlight.Add(delta);

    public void Dispose() => _meter.Dispose();
}
```

Watch them by meter name:

```bash
dotnet-counters monitor -p 12345 --counters Acme.Orders
```

`Meter` instruments are the **same** ones OpenTelemetry exports — register once, read everywhere. Use `Counter<T>` for monotonic, `UpDownCounter<T>` for gauges that move both ways, `Histogram<T>` for distributions (latencies, sizes), and `ObservableGauge<T>` for callback-polled values.

The legacy `EventCounter` / `IncrementingEventCounter` / `PollingCounter` API still works (it's what `System.Runtime` itself uses) but `Meter` is the path forward — same wire format from `dotnet-counters`' perspective, far better OTel/Prometheus story.

## Counter-list discovery

```bash
# All known counters for the process — catalogues both built-in and custom.
dotnet-counters list -p 12345

# All providers known statically (no PID needed).
dotnet-counters list
```

`list -p` is the only way to discover the names of `Meter` instruments after the fact. If your custom counter doesn't appear, the `Meter` name is wrong (case-sensitive) or the meter wasn't created yet (lazy-init pattern).

## Containers and Kubernetes

The diagnostic socket is in the target's filesystem, so the consumer must be there too.

```bash
# Pattern 1: kubectl exec, dotnet-counters baked into the image.
kubectl exec -it orders-7df-abc12 -- dotnet-counters monitor -p 1

# Pattern 2: shared emptyDir at /tmp; sidecar with dotnet-counters mounts the same volume.
# In manifest:
#   volumes: { name: tmp, emptyDir: {} }
#   containers:
#     - name: app
#       volumeMounts: [{ name: tmp, mountPath: /tmp }]
#     - name: diag
#       image: mcr.microsoft.com/dotnet/monitor
#       volumeMounts: [{ name: tmp, mountPath: /tmp }]

# Pattern 3: DOTNET_DiagnosticPorts pointing at a sidecar listener — the runtime dials out.
# env: { name: DOTNET_DiagnosticPorts, value: /diag/diag.sock }
```

For a fleet (50+ pods), wrap with **`dotnet-monitor`** (the supported sidecar product, not to be confused with `dotnet-counters monitor`). It exposes counters and trace triggers over HTTP/gRPC and integrates with Prometheus / OTel.

## Production capture: a short script

```bash
#!/usr/bin/env bash
# Watch the GC + threadpool + ASP.NET basics for 5 minutes; emit CSV.
set -euo pipefail
PID=${1:?usage: $0 <pid>}
OUT=./counters-$(date +%Y%m%d-%H%M%S).csv

dotnet-counters collect -p "$PID" \
  --counters \
    System.Runtime[cpu-usage,working-set,gc-heap-size,alloc-rate,time-in-gc,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count,threadpool-queue-length,monitor-lock-contention-count,exception-count],\
    Microsoft.AspNetCore.Hosting[requests-per-second,current-requests,failed-requests],\
    System.Net.Http[requests-queue-duration,connections-current] \
  --refresh-interval 1 \
  --duration 00:05:00 \
  --format csv \
  --output "$OUT"

echo "wrote $OUT"
```

Bracket syntax `Provider[counter1,counter2]` filters to specific counter names — keeps the CSV narrow and the screen readable in `monitor` mode.

## Senior-level gotchas

- **Counters are sampled, not event-driven.** Default `--refresh-interval` is 1 s. A latency burst that lasts 200 ms might not move any counter visibly. For sub-second resolution, drop the interval — but at < 0.5 s the polling overhead becomes measurable. For real diagnosis below 1 s, use [`dotnet-trace`](./dotnet-trace.md).
- **`alloc-rate` and `gen-N-gc-count` are *rates per minute*, not cumulative.** Reading the column as "total allocations" leads to wrong conclusions. The CSV `collect` output makes this clearer than the live monitor display.
- **Custom `Meter` instruments must be created and `.Add`ed to before they appear.** A `Counter<long>` exists from the moment `CreateCounter` is called, but `dotnet-counters` may not list it until at least one observation has flowed. Trigger the code path once before assuming the meter is missing.
- **Histogram counters render as percentiles in `dotnet-counters monitor`** but only `p50/p95/p99` by default. The full distribution requires an OTel exporter. For ad-hoc histogram viewing, prefer `dotnet-trace` with the `Microsoft-Diagnostics-DiagnosticSource` provider plus PerfView.
- **`dotnet-counters ps` lists nothing → check `TMPDIR`.** Distroless / read-only-root images often relocate `TMPDIR`. Both the runtime and the tool must agree on the path. Set `TMPDIR=/tmp` explicitly in the image if portability matters.
- **A frozen process won't surface counters.** If the IPC thread is blocked (deadlock, GC pause longer than the IPC keepalive), `dotnet-counters` will hang or time out. Symptom of "tool says 'no process found' but `ps` shows the PID" — the process is alive but unresponsive on the diagnostic port. Time for [`dotnet-dump`](./dotnet-dump.md).
- **The `monitor` UI rounds and truncates aggressively.** A counter showing `0 B` for `alloc-rate` might be rounding 250 KB/s. Use `collect` and inspect the CSV when the magnitude matters.
- **Counter names are case-sensitive and non-overloadable.** Two `Meter`s creating an instrument named `"requests"` clash silently — only the first wins per process. Namespace your meters and instruments (`Acme.Orders`, `orders.placed`).
- **`EventCounter` (legacy) is polled by the runtime; `Meter` `Counter<T>` is push.** A custom `EventCounter` with an expensive `getValue` callback runs that callback *for every active session*. Multiple `dotnet-counters` instances multiply the cost. Migrate to `Meter` for new code.
- **`time-in-gc` is per-interval, not cumulative.** A spike to 80% on one sample can be a single Gen2 pause and not a chronic problem. Sustained > 5% across multiple samples is the actionable threshold.
- **Counters never include kernel-side data.** CPU usage is computed from the process's user/kernel time; disk wait shows up as flat CPU but can't be distinguished here. For "is this CPU or I/O wait?" you need ETW (`xperf -on PROC_THREAD+CSWITCH`) — see [`etw-event-tracing-for-windows.md`](./etw-event-tracing-for-windows.md).
- **`--counters` is comma-separated by *provider*, but bracket syntax filters by counter name.** A common mistake is `--counters cpu-usage,gc-heap-size` (won't work — those aren't provider names). The correct form is `--counters System.Runtime[cpu-usage,gc-heap-size]`.
- **Don't run `dotnet-counters` against your own benchmarking process.** `[MemoryDiagnoser]` already collects equivalent data and counter polling perturbs the timings. See [`benchmarkdotnet.md`](./benchmarkdotnet.md).
- **`dotnet-monitor` (the sidecar product) is what you want for production at scale**, not a hand-rolled `kubectl exec dotnet-counters` loop. It supports rule-based capture (collect a trace when CPU > X for Y seconds), authentication, and Prometheus scrape endpoints.
