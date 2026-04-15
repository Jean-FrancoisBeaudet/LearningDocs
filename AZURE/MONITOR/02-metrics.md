# Metrics

> **Exam mapping:** AZ-204 · AZ-305 · AI-300
> **One-liner:** A lightweight, high-frequency time-series store for numeric data — use it for dashboards, autoscale, and sub-minute alerting; don't use it for anything you need to query past **93 days**.
> **Related:** [Overview](00-overview.md) · [Logs](03-logs-and-log-analytics.md) · [Alerts](06-alerts-and-action-groups.md) · [Autoscale](07-autoscale.md)

## What the metrics store is

| Property | Value |
|----------|-------|
| Engine | Purpose-built time-series DB (separate from Log Analytics) |
| Granularity | 1-minute default; some resources emit at 15s |
| Retention | **93 days** (fixed for platform metrics) |
| Query interface | Metrics Explorer, REST API, SDK, Managed Grafana, Workbooks |
| Alert latency | Typically **< 1 minute** end-to-end |

Design rule: if you need the data beyond 93 days → route to a workspace via Diagnostic Setting → **log-based metric alert** or archive.

## Metric types

| Type | Source | Billing |
|------|--------|---------|
| **Platform metrics** | Emitted by every Azure resource (CPU, Requests, Dependencies, RU/s). | Free |
| **Guest OS metrics** | Windows perf counters / Linux metrics via AMA + DCR. | Billed as custom |
| **Custom metrics** | App Insights `TrackMetric`, REST API, OTel exporter. | **Billed per time-series per month** |
| **Prometheus metrics** | Scraped via **Azure Monitor Managed Service for Prometheus** (AKS + Arc). | Billed per sample |

## Multi-dimensional metrics

Metrics are `(timestamp, value, {dimensions})`. Dimensions let you slice without pre-aggregating:

```
RequestCount { app=api, region=eastus, statusCode=500 }  → 42
```

Splitting + filtering by dimension is the superpower of Metrics Explorer. Custom metric dimensions have limits (default **10 dims**, **100k time-series per metric**).

## Azure Monitor Managed Service for Prometheus

Fully managed Prometheus-compatible TSDB. Two big exam points:

- Scrapes AKS, AKS Edge, Arc-enabled Kubernetes, Arc servers.
- Visualized in **Azure Managed Grafana** (first-class integration), alerted via **Prometheus rule groups** (PromQL).
- Built on **Azure Monitor Workspace** (distinct from Log Analytics workspace — don't confuse them).

```
┌─────────────┐   scrape    ┌──────────────────────────┐   PromQL    ┌────────────────────┐
│ AKS cluster │ ──────────► │ Azure Monitor Workspace  │ ──────────► │ Managed Grafana    │
└─────────────┘             └──────────────────────────┘             └────────────────────┘
                                          │
                                          ▼
                                Prometheus alert rule group → Action Group
```

## Metrics Explorer patterns

- **Split by dimension**: pick `Azure OpenAI / Prompt Tokens`, split by `ModelDeploymentName`.
- **Apply multiple metrics on one chart** (up to 10) with independent aggregations.
- **Save to Dashboard / Workbook**, pin as tile, or export to Grafana.

## When to pick metric vs log for alerting

| Need | Pick |
|------|------|
| Fast (< 1 min), simple threshold on a numeric counter | **Metric alert** |
| Correlate across tables, regex parsing, joins | **Log alert** |
| Burst/Spike detection with ML baselines | **Metric alert — dynamic thresholds** |
| Aggregate over > 93 days | **Log alert** (on workspace data) |

## Exam traps

- Platform metrics retention is **93 days**, not configurable. Archive via Diagnostic Setting.
- **Azure Monitor workspace ≠ Log Analytics workspace.** The former stores Prometheus metrics; the latter stores logs.
- Custom metric cardinality explosions are the #1 cost/quota issue — watch dimension count.
- Log-based metric alerts exist but run on the Logs engine (slower, billed per query). Prefer a native metric alert when possible.

## Sources

- [Metrics overview](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-platform-metrics)
- [Metrics Explorer](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/analyze-metrics)
- [Custom metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-custom-overview)
- [Managed Service for Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview)
- [Azure Monitor workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/azure-monitor-workspace-overview)
