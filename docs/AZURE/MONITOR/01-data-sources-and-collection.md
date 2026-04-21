# Data Sources & Collection

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 / AI-103
> **One-liner:** Every Azure resource emits data in one of four lanes — platform metrics, activity log, resource logs, guest OS / apps — and each lane has a distinct **collection mechanism** you must know.
> **Related:** [Overview](00-overview.md) · [Metrics](02-metrics.md) · [Logs](03-logs-and-log-analytics.md) · [App Insights](05-application-insights.md)

## The four data lanes

| Lane | What it is | Collection | Destination |
|------|-----------|------------|-------------|
| **Platform metrics** | Numeric counters Azure emits automatically (CPU %, requests/sec, RU/s) | None — always on | Metrics store |
| **Activity Log** | Subscription-level **control-plane** events (resource created, RBAC change, policy assignment) | None — always on | Activity log store + optional Diagnostic Setting to workspace/Event Hub/Storage |
| **Resource (platform) logs** | Data-plane logs from the service (SQL audit, Key Vault access, App Gateway requests, Azure OpenAI prompt logs) | **Diagnostic Settings** | Log Analytics / Event Hub / Storage / partner |
| **Guest OS + application** | Windows perf counters, Syslog, custom app logs, OTel traces | **Azure Monitor Agent + DCR**, App Insights SDK / OpenTelemetry | Log Analytics / Metrics |

## Diagnostic Settings (control-plane + data-plane → destination)

A Diagnostic Setting routes one resource's logs/metrics to up to **5 destinations in parallel**:

1. Log Analytics workspace
2. Azure Storage account (long-term archive)
3. Event Hub (stream to SIEM / third-party)
4. Partner (Datadog, etc. via Marketplace)
5. (As of 2025) send to another DCR endpoint

Rules of thumb:

- Up to **5 diagnostic settings per resource**.
- Activity Log needs its own (subscription-level) diagnostic setting to be stored beyond 90 days.
- Use Azure Policy **`DeployIfNotExists`** to enforce diagnostic settings at scale.

```bicep
resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'to-law'
  scope: keyVault
  properties: {
    workspaceId: law.id
    logs: [ { categoryGroup: 'allLogs', enabled: true } ]
    metrics: [ { category: 'AllMetrics', enabled: true } ]
  }
}
```

## Data Collection Rules (DCR) — the new control plane

DCRs are the **replacement for legacy agent config**. A DCR declares: *what* to collect, *how* to transform it (KQL-lite), and *where* to send it.

```
Source ─► DCR (input streams + transform + destinations) ─► Log Analytics / custom table / metrics
                                     ▲
                                     │  linked via DCR Association (DCRA)
                           Azure Monitor Agent on VM / Arc
```

Key objects:

| Object | Purpose |
|--------|---------|
| **DCR** | Rule definition (JSON/ARM/Bicep). Reusable across many hosts. |
| **DCRA** (association) | Binds a DCR to a resource (VM, VMSS, Arc-enabled server). |
| **DCE** (endpoint) | Public/Private ingestion endpoint for custom logs / Private Link scenarios. |
| **Transformations** | `source | where Severity == "Error" | project TimeGenerated, Message` applied at ingest — drop noise before billing. |

## Azure Monitor Agent (AMA) — the only agent

**AMA has replaced all legacy agents.** Migrate before deadlines or ingestion stops.

| Legacy agent | Status |
|--------------|--------|
| **MMA / Log Analytics agent** (Windows/Linux) | **Retired Aug 31, 2024.** Cloud ingestion can be shut down by Microsoft **after March 2, 2026**. |
| **OMS agent** | Retired (same as MMA). |
| **Diagnostics Extension (WAD/LAD)** | Being phased out; migrate to AMA. |
| **Dependency agent** (VM Insights Map) | Still used *alongside* AMA for Map feature. |

AMA advantages: single agent for Monitor/Sentinel/Defender, DCR-driven config, Managed-Identity auth, filtering/transforms at source, Arc support.

```bash
# Install AMA via extension on a VM
az vm extension set \
  --resource-group rg1 --vm-name vm1 \
  --name AzureMonitorWindowsAgent --publisher Microsoft.Azure.Monitor
```

## Application telemetry (separate lane, separate product)

App traces/metrics/logs go through **Application Insights** — see [`05-application-insights.md`](05-application-insights.md). Use the **OpenTelemetry Distro** (now GA path) rather than the classic SDK for new work.

## Custom metrics and custom logs

- **Custom metrics**: send via App Insights SDK, Azure Monitor REST API, or AMA DCR custom-metric stream. Billed per time-series.
- **Custom logs (tables)**: DCR + Logs Ingestion API into a *custom table* (`MyApp_CL`). Replaces the retired HTTP Data Collector API.

## Exam traps

- Activity Log → **subscription-level** diagnostic setting, not per-resource.
- A Diagnostic Setting is **per-resource**; at-scale deployment = Azure Policy `DeployIfNotExists`.
- You **cannot** edit built-in platform metric categories — they're emitted by the resource provider.
- Choosing between Storage / Event Hub / Workspace: **Storage = cheap archive**, **Event Hub = stream out**, **Workspace = query with KQL + alerts**.
- MMA is gone. If an exam answer says "install the Log Analytics agent", it's wrong in 2026 content.

## Sources

- [Diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings)
- [Activity log](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log)
- [Azure Monitor Agent overview](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/agents-overview)
- [Data Collection Rules](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview)
- [AMA migration (MMA retirement)](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-migration)
- [Logs Ingestion API](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)
