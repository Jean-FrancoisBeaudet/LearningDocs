# Alerts & Action Groups

> **Exam mapping:** AZ-204 · AZ-305 · AI-300
> **One-liner:** An alert is a **rule** that evaluates a **signal** against **criteria** and invokes one or more **action groups**, optionally modified by **alert processing rules**.
> **Related:** [Metrics](02-metrics.md) · [Logs](03-logs-and-log-analytics.md) · [Autoscale](07-autoscale.md)

## Alert anatomy

```
┌───────────┐    ┌────────────┐    ┌──────────────┐    ┌──────────────┐
│  Signal   │ ─► │ Alert rule │ ─► │ Alert fires  │ ─► │ Action group │
│ metric/log│    │ criteria + │    │ (with state) │    │ (channels)   │
│ activity  │    │ severity   │    │              │    │              │
└───────────┘    └────────────┘    └─────┬────────┘    └──────────────┘
                                         ▼
                              Alert processing rule
                         (suppress / route / add AG)
```

## The 7 alert types

| Type | Source | Cost | Latency | Example |
|------|--------|------|---------|---------|
| **Metric alert** | Metrics store | Per rule + per time-series | < 1 min | CPU > 80 % 5 min |
| **Log (scheduled) alert** | Log Analytics / App Insights | Per query execution | 1–5 min | `AppExceptions | count > 10` |
| **Activity Log alert** | Subscription activity log | Free | Near real-time | RBAC role assigned at subscription |
| **Service Health alert** | Azure Status / Advisories | Free | Platform-driven | Region X outage |
| **Resource Health alert** | Resource-level health | Free | Platform-driven | VM unhealthy |
| **Smart Detection** (App Insights) | ML over telemetry | Free | Minutes | Failure-rate anomaly |
| **Prometheus alert** | Azure Monitor Workspace (Prometheus) | Per rule | Seconds–minutes | PromQL rule group, e.g. `rate(http_errors[5m])` |

## Metric alerts — power features

- **Static threshold**: classic `>`/`<` comparison.
- **Dynamic thresholds**: ML-derived upper/lower bands, sensitivity High/Med/Low. Good for seasonal workloads.
- **Multi-resource**: one rule over many VMs / storage accounts / Kubernetes namespaces.
- **Multiple conditions**: combine several signals in one rule (all must fire).
- **Target dimensions**: fire a separate alert per dimension value (e.g. per instance).

## Log alerts — things to know

- Runs on a **recurrence** (1 min → 1 day). Faster recurrence = more cost.
- Evaluate **Number of results** or **Metric measurement** (with grouping).
- **Stateful** log alerts can auto-resolve.
- You can't alert on **Basic** or **Auxiliary** plan tables directly — promote via search job if needed.

```kusto
// Example log-alert query
AppExceptions
| where TimeGenerated > ago(5m)
| summarize Count = count() by bin(TimeGenerated, 1m), Cloud_RoleName
```

## Severities

`Sev 0` Critical → `Sev 4` Verbose. Sev is **metadata** — it doesn't gate routing by itself, but action groups and processing rules commonly branch on it.

## Action groups

An Action Group is a **reusable bundle of channels**.

| Channel | Notes |
|---------|-------|
| Email / SMS / Push (Azure mobile app) / Voice | Per-user. SMS and voice have rate limits (≈ 1 msg/5 min). |
| Webhook / Secure Webhook | Secure uses Entra ID auth. |
| Azure Function | Arbitrary logic. |
| Logic App | Low-code integration (Teams, ServiceNow, Jira). |
| Automation Runbook | Remediation. |
| ITSM connector | ServiceNow / Cherwell / Provance / SCSM. |
| Event Hub | Stream alerts to SIEM. |

Limits (exam-worthy):

- **Unlimited action groups per subscription.**
- Within one AG: up to **1 000 email addresses**, **10 SMS**, **10 voice**, **10 webhooks** per type.
- Short suppression windows prevent storms (Alert Processing Rules handle this more cleanly).

## Alert processing rules

Layer **on top of alerts** to:

- **Suppress** during maintenance windows (one-shot or recurring).
- **Apply an action group** to many rules at once (e.g. send all `Sev 0` to PagerDuty).
- **Filter** by tags, resource type, severity.

Prefer processing rules over putting suppression logic in each alert rule — cleaner governance, fewer edits at scale.

## Authoring alerts — at-scale

- **Azure Policy** with `DeployIfNotExists` for diagnostic settings + alert rules.
- **Bicep / ARM**: `Microsoft.Insights/metricAlerts`, `scheduledQueryRules`, `actionGroups`, `activityLogAlerts`, `processingRules`.
- **Terraform** provider has full parity.

```bicep
resource cpu 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'vm-cpu-high'
  location: 'global'
  properties: {
    severity: 2
    enabled: true
    scopes: [ vmssId ]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [ {
        name: 'cpu'
        metricName: 'Percentage CPU'
        operator: 'GreaterThan'
        threshold: 80
        timeAggregation: 'Average'
      } ]
    }
    actions: [ { actionGroupId: ag.id } ]
  }
}
```

## Exam traps

- **Activity Log ≠ Metric ≠ Log alert.** Know which signal is where.
- **Dynamic thresholds** only apply to **metric** alerts, not log alerts.
- **Smart Detection** is App-Insights-specific; don't confuse with dynamic thresholds.
- Alerts on Basic/Auxiliary plan tables: **not supported** directly.
- Action groups are **regional** objects but can target resources in any region (the alert rule's location pins the rule).

## Sources

- [Alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Types of alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types)
- [Action groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [Dynamic thresholds](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-dynamic-thresholds)
- [Alert processing rules](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-action-rules)
- [Prometheus alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/prometheus-alerts)
- [Service limits](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits)
