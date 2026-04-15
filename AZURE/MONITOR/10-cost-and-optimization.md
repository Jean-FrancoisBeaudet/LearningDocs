# Cost & Optimization

> **Exam mapping:** AZ-305 *(cost optimization pillar)* · AZ-204 *(App Insights sampling)*
> **One-liner:** Monitor's bill is dominated by **Log Analytics ingestion** and **App Insights telemetry volume** — everything else is rounding error. Cut volume before you cut retention.
> **Related:** [Logs](03-logs-and-log-analytics.md) · [App Insights](05-application-insights.md) · [Data sources](01-data-sources-and-collection.md)

## What you actually pay for

| Line item | Typical share | Knob |
|-----------|---------------|------|
| Log Analytics ingestion | **Dominant** | Table plan, DCR filters/transforms, sampling, daily cap |
| Log Analytics retention beyond 30 days | Meaningful | Per-table retention, archive |
| App Insights telemetry | Dominant in dev-heavy orgs | Adaptive sampling, dependency tracking toggles |
| Custom metrics | Can surprise you | Dimension cardinality |
| Alerts | Minor | Metric alerts free-tier, log alerts per query cost |
| Managed Prometheus | Scales with samples | Scrape config |
| Managed Grafana | Fixed SKU | SKU choice |
| Metrics store (platform) | **Free** | n/a |
| Activity log | **Free** for 90 days | n/a |

## Pricing model for Logs

1. **Pay-as-you-go** per-GB ingested (differs per plan):
   - **Analytics** — full price.
   - **Basic** — ~20 % of Analytics.
   - **Auxiliary** — lowest ingestion cost, minimal query power.
2. **Commitment tiers** (workspace-scoped): 100 GB/day → 5 TB/day; 15–30 % discount; **billed whether you use it or not**.
3. **Dedicated Cluster** (≥ 500 GB/day across linked workspaces): biggest discount, CMK, double encryption.
4. **Retention**: interactive (days 1–30 free, 31–730 billed/GB/month); **archive** (cheap, requires restore/search-job to query).
5. **Basic** tables: per-GB **search queries** (you pay per query volume scanned).

## The 7-step cost playbook

1. **Pick the right plan per table.** Chatty verbose tables → **Basic**; compliance-only → **Auxiliary**; everything with alerts/joins → **Analytics**.
2. **Filter at the source.** DCR `transformKql` drops unwanted rows *before* billing kicks in.
3. **Project columns out** in DCR transforms — huge `Properties` blobs are expensive.
4. **Adaptive sampling** in App Insights (default ON in SDK; re-confirm).
5. **Daily cap** per workspace and per App Insights — a safety net, not a strategy.
6. **Reduce retention where audits allow.** Default 30 days; lengthen only where required.
7. **Archive + search job** for compliance data you almost never query.

## DCR transformation example

```kusto
source
| where Severity in ("Error","Critical","Warning")
| where Namespace !startswith "kube-system"
| project TimeGenerated, Computer, Severity, Message, Namespace
```

Everything you drop here is **never billed**.

## App Insights specific levers

- **Adaptive sampling** — target items/sec, preserves correlation.
- **Disable auto-collected modules** you don't need (perf counters, SQL dependencies).
- **TelemetryProcessor / Processor** to drop noisy dependencies (health pings).
- **Live Metrics** doesn't count against ingestion — use instead of logging for hot debug.
- **Daily cap** in AI resource settings.

## Monitoring the bill itself

- Portal → *Log Analytics workspace → Usage and estimated costs* → projection and breakdown by table.
- `Usage` table in KQL:

```kusto
Usage
| where TimeGenerated > ago(30d)
| summarize GB = sum(Quantity) / 1024 by DataType, bin(TimeGenerated, 1d)
| render timechart
```

- Pair with **Cost Management** budgets + action group for spend alerts.

## Exam traps

- **Commitment tier is billed even if you under-consume.** Only choose it with confident baseline.
- Switching a built-in table to Basic is allowed for a subset of tables; **Auxiliary requires a new custom table**, not existing.
- **Sentinel** adds a per-GB Sentinel price on top of ingestion; don't ignore in cost designs.
- Retention reduction is **destructive** — data older than the new retention is dropped after grace period.
- Metrics (platform) are **always free**. Custom metrics are not.

## Sources

- [Monitor pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/)
- [Log Analytics cost calculations](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs)
- [Commitment tiers](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs#commitment-tiers)
- [Optimize Monitor costs](https://learn.microsoft.com/en-us/azure/azure-monitor/best-practices-cost)
- [Daily cap](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/daily-cap)
- [App Insights sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling)
