# Application Insights

> **Exam mapping:** AZ-204 *(primary topic — instrumenting apps)* · AZ-305 *(observability architecture)* · AI-102 / AI-103 *(LLM telemetry)* · AI-300
> **One-liner:** The **APM** feature of Azure Monitor — collects requests, dependencies, exceptions, traces, metrics, browser telemetry, and correlates them end-to-end in a single workspace-based resource.
> **Related:** [Logs](03-logs-and-log-analytics.md) · [KQL](04-kql-essentials.md) · [Alerts](06-alerts-and-action-groups.md)

## Status of the product (2026)

| Thing | State |
|-------|-------|
| **Classic (non-workspace) Application Insights** | **Retired Feb 29, 2024.** All resources are now workspace-based. |
| **Workspace-based App Insights** | Current default. Data lives in Log Analytics tables (`AppRequests`, `AppDependencies`, `AppExceptions`, `AppTraces`, `AppPageViews`, `AppPerformanceCounters`, `AppMetrics`, `AppBrowserTimings`, `AppAvailabilityResults`, `AppSystemEvents`). |
| **OpenTelemetry Distro** (Azure Monitor OpenTelemetry) | **GA** — preferred path for new .NET/Java/Python/Node apps. |
| **Classic SDK 2.x (.NET)** | Deprecated; **retires March 31, 2027**. Migrate to OTel Distro or SDK 3.x. |
| **URL Ping availability test** | **Retires September 30, 2026.** Use **Standard test** (or TrackAvailability / custom TrackAvailabilityTest via Functions). |

## Instrumentation options

| Method | When | Notes |
|--------|------|-------|
| **Auto-instrumentation (codeless)** | App Service, Functions, AKS sidecar, VMs | Toggle in portal or app setting `APPLICATIONINSIGHTS_CONNECTION_STRING` + enable; no code change. |
| **OpenTelemetry Distro** | New app, multi-cloud friendliness | Use `Azure.Monitor.OpenTelemetry.AspNetCore` (C#), `azure-monitor-opentelemetry` (Python), etc. |
| **Classic SDK 2.x / 3.x** | Existing apps | Still works; migrate to OTel. |
| **Client-side JS SDK** | Browser telemetry (page views, AJAX, exceptions) | `@microsoft/applicationinsights-web`. |

**Connection string** (not the legacy Instrumentation Key alone) is the current way to point telemetry at a resource.

```csharp
// Program.cs — ASP.NET Core OTel Distro
builder.Services.AddOpenTelemetry().UseAzureMonitor(o =>
{
    o.ConnectionString = builder.Configuration["APPINSIGHTS_CONN"];
});
```

## Data model (exam-critical)

| Telemetry | Log Analytics table | When emitted |
|-----------|--------------------|--------------|
| Request | `AppRequests` | Incoming HTTP / RPC to your app |
| Dependency | `AppDependencies` | Outgoing calls (SQL, HTTP, Cosmos, Service Bus) |
| Exception | `AppExceptions` | `ILogger` LogError with exception, unhandled, manual `TrackException` |
| Trace | `AppTraces` | Structured log messages |
| Custom event | `AppEvents` | `TrackEvent("CheckoutStarted", …)` |
| Custom metric | `AppMetrics` | `TrackMetric` / OTel counter |
| Availability | `AppAvailabilityResults` | Availability tests (Standard / Custom TrackAvailability) |
| Page view | `AppPageViews` | Browser SDK |

All are correlated via **operation_Id** (end-to-end trace) and **operation_ParentId** (span).

## Live Metrics

Real-time (sub-second) pulse of your app. Not stored; streamed from running instances for troubleshooting. Works independently of sampling.

## Sampling — keep cost in check

| Mode | Where it runs | Behavior |
|------|---------------|----------|
| **Adaptive sampling** | SDK (local) | Default for ASP.NET / ASP.NET Core SDK. Target items/sec; adjusts rate automatically. Preserves correlation. |
| **Fixed-rate sampling** | SDK or OTel | You pick a percentage. Deterministic. |
| **Ingestion sampling** | Server-side at ingest | Last resort — applied after data leaves the app. No cost saving on egress. |
| **Daily cap** | Workspace / AI resource | Hard ceiling; drops data for the rest of the day. |

Rule: push sampling as close to the source as possible — adaptive first, ingestion last.

## Availability tests

| Test type | Status | Scenario |
|-----------|--------|----------|
| **URL Ping test** | **Retires 2026-09-30** | Single URL, 5-region ping. Migrate. |
| **Standard test** | Current default | Like URL ping + SSL validation, TLS config, HTTP headers, custom status code. |
| **Custom TrackAvailability** | Any scenario | Azure Function runs logic (multi-step, auth) and reports result. |

## Useful features often missed

- **Application Map** — auto-generated topology of components and dependencies, with failure/perf hotspots.
- **Smart Detection** — ML-based anomaly alerts (failure-rate spikes, response-time degradation).
- **Profiler / Snapshot Debugger** — CPU profiles and exception snapshots from production (Windows Apps).
- **Usage analytics** — Funnels, Cohorts, Retention, Impact, User Flows.
- **Releases annotations** — mark deployments on charts via REST API / GitHub Actions task.

## Cost control checklist

1. Adaptive sampling ON.
2. Daily cap set (portal → Usage and estimated costs).
3. Disable dependency tracking for ultra-chatty callers (Redis heartbeats).
4. Disable `AppPerformanceCounters` if not needed.
5. Use Basic plan for high-volume verbose tables (custom) when alerts not needed.

## Exam traps

- **Classic App Insights is gone** — any answer creating one is wrong.
- **Connection String**, not just Instrumentation Key.
- OTel Distro is the **recommended** instrumentation path now.
- URL Ping test → use **Standard test** for new work.
- `AppRequests` etc. are **the new table names**; old names (`requests`, `exceptions`) still appear in Application Insights Analytics UI via a shim.

## Sources

- [Application Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Workspace-based resources](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource)
- [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)
- [Classic resource retirement](https://learn.microsoft.com/en-us/azure/azure-monitor/app/classic-api)
- [Availability tests](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability)
- [Sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling)
- [Application Insights FAQ](https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-faq)
