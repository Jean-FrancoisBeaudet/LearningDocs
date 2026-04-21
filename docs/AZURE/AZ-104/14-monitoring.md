# Monitor Resources in Azure

> **Exam mapping:** AZ-104 -- "Monitor and maintain Azure resources" (10--15 %)
> **One-liner:** Use Azure Monitor to collect metrics and logs from every resource, query them with KQL, visualise trends in Metrics Explorer and Insights, fire alerts through action groups, and diagnose network issues with Network Watcher -- the exam tests you on all six sub-skills below.
> **Related:** [15-backup-and-recovery](15-backup-and-recovery.md) | [AZURE/MONITOR/ series](../MONITOR/00-overview.md) (covers every topic here in much greater depth)

---

## 1 Interpret Metrics in Azure Monitor

[Azure Monitor metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/data-platform-metrics) are **numeric time-series data** collected at regular intervals. They are lightweight, near-real-time, and stored in a dedicated time-series database optimised for fast alerting and charting.

### 1.1 Platform vs Custom Metrics

| Aspect | Platform metrics | Custom metrics |
|--------|-----------------|----------------|
| **Source** | Emitted automatically by Azure resource providers | Sent via Application Insights SDK, Azure Monitor Agent (AMA) custom metrics, or REST API |
| **Namespace** | `microsoft.<provider>/<resourceType>` (e.g. `microsoft.compute/virtualmachines`) | Custom namespace you define |
| **Cost** | Free (included in the resource) | [Ingestion + query charges](https://learn.microsoft.com/en-us/azure/azure-monitor/cost-usage#azure-monitor-metrics) apply |
| **Retention** | **93 days** | **93 days** |

> **Key fact:** Both platform and custom metrics are retained for **93 days**. After that they are dropped. To keep metrics longer, create a [diagnostic setting](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings) that routes them to a **Log Analytics workspace** (up to 2 years interactive / 12 years archive) or to a **Storage Account** for indefinite archival.

### 1.2 Metrics Explorer

[Metrics Explorer](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/analyze-metrics) is the portal blade for interactive charting.

| Feature | Detail |
|---------|--------|
| **Chart types** | Line, area, bar, scatter |
| **Aggregation** | Avg, Sum, Min, Max, Count |
| **Time range** | Last 30 min to last 30 days per chart; pan across the full 93-day retention window |
| **Time granularity** | Auto (1 min, 5 min, 1 h, etc.) or manual |
| **Splitting** | Break a metric into per-dimension lines (e.g. split CPU % by VM instance) |
| **Filtering** | Include/exclude specific dimension values |
| **Multi-resource** | Select multiple resources of the **same type and region** on one chart |
| **Pin to dashboard** | Pin any chart to an Azure Dashboard or Workbook |

```text
Portal path: Monitor > Metrics  (or Resource blade > Monitoring > Metrics)
```

### 1.3 Diagnostic Settings for Metrics

To route platform metrics to Log Analytics (for KQL queries and long-term retention):

```bash
az monitor diagnostic-settings create \
  --name "send-metrics-to-la" \
  --resource "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm}" \
  --workspace "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}" \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

> **DCR alternative (preview):** [Data Collection Rules for metrics export](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/metrics-export-create) offer lower latency (~3 min vs 6--10 min for diagnostic settings), dimension support, and per-metric filtering. If the exam references DCR-based metric export, remember this is the newer path.

---

## 2 Configure Log Settings in Azure Monitor

### 2.1 Log Analytics Workspace

A [Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview) is the central store for log data in Azure Monitor. Think of it as a KQL-queryable database with tables for every data type.

| Setting | Exam-relevant detail |
|---------|---------------------|
| **Pricing tier** | Pay-as-you-go (per-GB ingestion) or commitment tiers (100--5 000 GB/day) |
| **Interactive retention** | Default 30 days, configurable up to **730 days (2 years)** |
| **Archive tier** | Up to **12 years** total; archived data is cheaper but needs a `search` job or `restore` to query |
| **Access control** | Workspace-context (RBAC on the workspace) or resource-context (RBAC on the source resource) |

### 2.2 Diagnostic Settings

[Diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings) tell Azure **where to send** the telemetry that a resource produces. Each resource can have up to **five** diagnostic settings.

**What they route:**

| Data category | Examples |
|--------------|----------|
| **Platform metrics** | CPU %, Transactions, DTU % |
| **Resource logs** | App-specific logs (e.g. Key Vault audit events, SQL query stats, Storage read/write/delete) |
| **Activity log** | Control-plane operations (via a subscription-level diagnostic setting) |

**Destinations:**

| Destination | Use case |
|-------------|----------|
| **Log Analytics workspace** | KQL queries, alerts, Workbooks, long-term retention |
| **Storage Account** | Compliance archival, third-party SIEM ingestion |
| **Event Hub** | Stream to external systems (Splunk, Datadog, etc.) |
| **Partner solution** | Integrated partner monitoring (e.g. Elastic, Datadog native integration) |

### 2.3 Data Collection Rules (DCR) and Azure Monitor Agent

The [Azure Monitor Agent (AMA)](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-overview) is the single, unified agent for guest-level telemetry. It replaces the legacy Log Analytics agent (MMA/OMS), Diagnostics extension (WAD/LAD), and Telegraf agent.

**Data Collection Rules** govern what AMA collects:

```
DCR defines:
  ├── Data sources  (Windows Events, Syslog, Performance Counters, custom text logs)
  ├── Transformations  (KQL transform at ingestion -- filter, enrich, project)
  └── Destinations  (Log Analytics workspace, Azure Monitor Metrics, Event Hub)
```

| DCR concept | Detail |
|-------------|--------|
| **Association** | A DCR is linked to one or more VMs (or Arc machines) via a **DCR association** |
| **Multiple DCRs** | A single VM can have several DCR associations |
| **Built-in DCRs** | VM Insights auto-creates a DCR when you enable it |

### 2.4 Resource Logs vs Activity Log

| | Resource logs | Activity log |
|---|---|---|
| **Scope** | Per-resource (data-plane + resource-specific events) | Per-subscription (control-plane operations) |
| **Enabled by** | Diagnostic setting on the resource | Always emitted; routed via subscription diagnostic setting |
| **Retention (native)** | Depends on destination | **90 days** in the portal (free); route to Log Analytics for longer |
| **Examples** | Key Vault access events, SQL query stats, Blob read counts | VM created, RBAC role assigned, policy evaluated |
| **Categories** | Vary by resource type (audit, request, allLogs, etc.) | Administrative, Security, Service Health, Alert, Recommendation, Policy, Autoscale, Resource Health |

---

## 3 Query and Analyze Logs in Azure Monitor

All log data is queried via [Kusto Query Language (KQL)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/get-started-queries) in the **Log Analytics** blade.

### 3.1 KQL Essentials

```kusto
// -- Filtering --
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue == "MICROSOFT.COMPUTE/VIRTUALMACHINES/WRITE"

// -- Projection --
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| project TimeGenerated, Computer, CounterValue

// -- Aggregation --
Heartbeat
| summarize count() by Computer
| order by count_ desc

// -- Time binning & charting --
Perf
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart

// -- Extending with calculated columns --
AzureActivity
| extend DurationMin = DurationMs / 60000.0
| where DurationMin > 5

// -- String search --
Syslog
| where SyslogMessage contains "error"
| project TimeGenerated, Computer, Facility, SyslogMessage
```

**Must-know operators for the exam:**

| Operator / Function | Purpose |
|---------------------|---------|
| `where` | Filter rows |
| `project` | Select / rename columns |
| `summarize` | Aggregate (count, avg, sum, min, max, percentile, dcount) |
| `extend` | Add calculated column |
| `order by` / `sort by` | Sort results |
| `render` | Visualise (timechart, barchart, piechart, table) |
| `ago()` | Relative time (e.g. `ago(1h)`, `ago(7d)`) |
| `bin()` | Bucket timestamps for grouping (e.g. `bin(TimeGenerated, 1h)`) |
| `join` | Combine rows from two tables |
| `union` | Stack rows from multiple tables |
| `contains` / `has` | String matching (`has` is token-based and faster) |
| `count` | Shortcut for `summarize count()` |

### 3.2 Common Log Analytics Tables

| Table | Contains |
|-------|----------|
| `AzureActivity` | Subscription-level control-plane events (create, delete, role assign) |
| `AzureDiagnostics` | Resource logs for services using Azure Diagnostics mode (Key Vault, SQL, etc.) |
| `Heartbeat` | Agent heartbeat -- confirms the VM agent is alive and reporting |
| `Perf` | Performance counters from AMA (CPU, memory, disk, network) |
| `Syslog` | Linux syslog messages |
| `Event` | Windows Event Log entries |
| `AppRequests` | Application Insights incoming HTTP requests |
| `AppExceptions` | Application Insights exceptions |
| `AppTraces` | Application Insights trace/log messages |
| `InsightsMetrics` | Metrics collected by VM Insights and Container Insights |

### 3.3 Practical Features

- **Saved queries:** save and share KQL queries within a workspace; available from the query explorer sidebar.
- **Export:** results can be exported to CSV, Power BI (`M` query), or pinned to a Workbook / Dashboard.
- **Query packs:** group related queries for team sharing across workspaces.
- **Search jobs:** run long-running queries against archived data (returns results to a new table).

---

## 4 Alert Rules, Action Groups, and Alert Processing Rules

### 4.1 Alert Rule Types

| Type | Signal source | Evaluation | Key detail |
|------|--------------|------------|------------|
| **[Metric alert](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-create-metric-alert-rule)** | Metrics store | Every 1--5 min (configurable) | Stateful (auto-resolves when condition clears); supports dynamic thresholds (ML-based) |
| **[Log search alert](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-create-log-alert-rule)** | Log Analytics query (KQL) | Every 5--15 min (configurable) | Stateful or stateless; runs a KQL query and checks if result count / metric value exceeds threshold |
| **Activity log alert** | Activity log events | Event-driven (no polling) | Fires on specific control-plane operations or Service Health events; **no configurable severity** (always Verbose) |
| **Smart detection** | Application Insights | Continuous ML | Auto-detects anomalies in failure rates, response times, dependency calls |

> **Exam trap:** Metric alerts evaluate more frequently (as low as every 1 minute) than log search alerts (minimum 1 minute, commonly 5--15 minutes). Know which to use for near-real-time scenarios.

### 4.2 Alert Severity Levels

| Sev | Label | Guidance |
|-----|-------|----------|
| **0** | Critical | Immediate action required |
| **1** | Error | Needs prompt attention |
| **2** | Warning | Potential issue |
| **3** | Informational | FYI |
| **4** | Verbose | Diagnostic / debug |

> Activity log alerts always fire at **Sev 4 (Verbose)** and you cannot change the severity.

### 4.3 Action Groups

An [action group](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups) defines **who to notify** and **what to trigger** when an alert fires.

| Action type | Notes |
|-------------|-------|
| **Email / SMS / Push / Voice** | Notification; subject to rate limits (max 100 emails/hr, 1 SMS/5 min) |
| **Azure Function** | Run serverless code |
| **Logic App** | Orchestrate a workflow |
| **Webhook** | Call an external HTTP endpoint (secure webhook uses AAD auth) |
| **ITSM connector** | Create incident in ServiceNow, etc. |
| **Automation Runbook** | Run an Azure Automation PowerShell/Python runbook |
| **Event Hub** | Stream alert payload to Event Hub |

- You can attach **up to 5 action groups** per alert rule.
- One action group can be shared across many alert rules.

### 4.4 Alert Processing Rules

[Alert processing rules](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-processing-rules) modify **fired alerts in flight** -- they do NOT create new alerts.

| Capability | Example |
|------------|---------|
| **Add action groups** | "For all Sev 0 alerts on subscription X, also page the on-call team" |
| **Suppress notifications** | "During the maintenance window Saturday 02:00--06:00, suppress all alert notifications for resource group rg-prod" |
| **Filter by scope** | Subscription, resource group, resource type, or specific resources |
| **Schedule** | One-time or recurring windows (daily, weekly) |

> **Key distinction:** An **alert rule** defines *when* to fire an alert. An **alert processing rule** modifies *what happens* after the alert fires (add/suppress action groups). They are configured separately.

---

## 5 Azure Monitor Insights

[Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-overview) are curated monitoring experiences built on top of metrics and logs for specific resource types.

### 5.1 VM Insights

| Feature | What it shows |
|---------|--------------|
| **Performance** | Pre-built charts for CPU, memory, disk IOPS, network bytes, logical disk capacity across all monitored VMs |
| **Map** | Auto-discovered dependency map of processes, connections, and ports (requires **Dependency agent**) |
| **Health** | Guest OS health criteria (heartbeat, disk space, memory) with rollup health state |

**Enabling VM Insights:**

1. Install **Azure Monitor Agent (AMA)** on the VM (auto-installed when you enable VM Insights from the portal).
2. Optionally install the **Dependency agent** for the Map feature (processes and connections).
3. A **Data Collection Rule** is automatically created, targeting the workspace you choose.

```bash
# Enable VM Insights via CLI
az monitor vm-insights enable \
  --resource-group rg-prod \
  --vm-name vm-web-01 \
  --workspace "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}"
```

### 5.2 Storage Insights

[Storage Insights](https://learn.microsoft.com/en-us/azure/storage/common/storage-insights-overview) provides a unified view of all Storage Accounts:

- **Capacity:** blob, file, table, queue capacity per account, growth trends.
- **Transactions:** success / failure counts, latency (E2E and server), availability.
- **Performance:** latency percentiles, throttling events.

No agent needed -- built entirely on platform metrics.

### 5.3 Network Insights

[Network Insights](https://learn.microsoft.com/en-us/azure/network-watcher/network-insights-overview) aggregates topology and health for networking resources:

- **Topology view:** visual map of VNets, subnets, NICs, NSGs, load balancers, gateways.
- **Connectivity:** surface Connection Monitor results and Network Watcher diagnostics.
- **Traffic:** NSG flow-log analytics and traffic-analytics dashboards.

### 5.4 Container Insights (Brief)

Monitors AKS clusters and Arc-enabled Kubernetes: node/pod/container performance, live logs, Prometheus metrics scraping. Uses AMA with a Container Insights DCR.

---

## 6 Network Watcher and Connection Monitor

### 6.1 Azure Network Watcher

[Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview) is a **regional** diagnostic and monitoring service for Azure networking.

| Fact | Detail |
|------|--------|
| **Scope** | One Network Watcher instance per region per subscription |
| **Auto-enabled** | Automatically created in a hidden `NetworkWatcherRG` resource group when you deploy the first networking resource in a region |
| **Must be enabled** | If it was disabled or deleted, you must re-enable it per region to use any tools |

### 6.2 Network Watcher Tools

| Tool | What it does | Exam scenario |
|------|-------------|---------------|
| **[IP flow verify](https://learn.microsoft.com/en-us/azure/network-watcher/ip-flow-verify-overview)** | Tests if a packet is allowed/denied to/from a VM (checks NSG rules) | "VM can't reach port 443 -- which NSG rule blocks it?" |
| **[Next hop](https://learn.microsoft.com/en-us/azure/network-watcher/next-hop-overview)** | Shows the next hop for a packet from a VM (UDR, VNet gateway, Internet, etc.) | "Traffic is not going through my NVA -- check routing" |
| **[Packet capture](https://learn.microsoft.com/en-us/azure/network-watcher/packet-capture-overview)** | Captures packets on a VM NIC; stores to storage account or local file | "I need a pcap for traffic analysis" |
| **[Connection troubleshoot](https://learn.microsoft.com/en-us/azure/network-watcher/connection-troubleshoot-overview)** | One-time connectivity check from VM to destination (TCP/ICMP); shows latency and hops | "Can this VM reach the SQL server on port 1433?" |
| **[NSG flow logs](https://learn.microsoft.com/en-us/azure/network-watcher/nsg-flow-logs-overview)** | Logs all flows through an NSG (src/dst IP, port, protocol, allow/deny); stored in Storage Account | "Audit all traffic allowed/denied by the NSG" |
| **[Traffic analytics](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics)** | Processes NSG flow logs and visualises traffic patterns, top talkers, threats | "Show me traffic patterns and anomalies" |
| **[VPN troubleshoot](https://learn.microsoft.com/en-us/azure/network-watcher/vpn-troubleshoot)** | Diagnoses VPN gateway and connection health | "S2S VPN tunnel is down -- why?" |
| **[NSG diagnostics](https://learn.microsoft.com/en-us/azure/network-watcher/nsg-diagnostics-overview)** | Evaluates effective NSG rules for a NIC, showing allow/deny for a given flow | Similar to IP flow verify but shows all matching rules |

### 6.3 Connection Monitor

[Connection Monitor](https://learn.microsoft.com/en-us/azure/network-watcher/connection-monitor-overview) provides **continuous, scheduled connectivity monitoring** (unlike Connection Troubleshoot, which is one-time).

**Architecture:**

```
Test Group
  ├── Sources:       Azure VMs, VMSS, Arc machines (need Network Watcher extension)
  ├── Destinations:   Azure VMs, FQDNs, URLs, IP addresses (no extension needed)
  └── Test configs:   Protocol (TCP/HTTP/ICMP), port, thresholds (latency, % failed)
```

| Concept | Detail |
|---------|--------|
| **Test group** | Pairs a set of sources with destinations and test configurations |
| **Checks** | HTTP (status code + latency), TCP (connect success + latency), ICMP (loss + latency) |
| **Frequency** | Configurable (default every 30 s for TCP/ICMP, every 60 s for HTTP) |
| **Agent** | Network Watcher extension (VM extension) on source machines; AMA can also be used for connectivity logs |
| **Metrics** | `ChecksFailedPercent` and `RoundTripTimeMs` emitted to Azure Monitor Metrics -- can be alerted on |

### 6.4 CLI Quick Reference

```bash
# Show Network Watcher status
az network watcher list --output table

# Enable Network Watcher in a region
az network watcher configure \
  --resource-group NetworkWatcherRG \
  --locations eastus \
  --enabled true

# IP flow verify
az network watcher test-ip-flow \
  --vm vm-web-01 \
  --resource-group rg-prod \
  --direction Inbound \
  --protocol TCP \
  --local 10.0.1.4:80 \
  --remote 10.0.2.4:12345

# Next hop
az network watcher show-next-hop \
  --vm vm-web-01 \
  --resource-group rg-prod \
  --source-ip 10.0.1.4 \
  --dest-ip 10.0.2.4

# Connection troubleshoot (one-time)
az network watcher test-connectivity \
  --source-resource vm-web-01 \
  --dest-address 10.0.3.5 \
  --dest-port 1433 \
  --resource-group rg-prod
```

---

## Exam Traps

| Trap | What to remember |
|------|-----------------|
| **Metrics retention** | Platform and custom metrics are kept for **93 days**. For longer retention, route to Log Analytics via diagnostic settings or DCR. |
| **Activity log retention** | Free portal view retains activity log events for **90 days**. Route to Log Analytics workspace for longer. |
| **Action group vs alert processing rule** | Action group = *who/what to notify*. Alert processing rule = *modify fired alerts in flight* (add/suppress action groups, filter by scope/time). They are separate resources. |
| **Metric alert vs log search alert frequency** | Metric alerts can evaluate as frequently as every **1 minute**. Log search alerts have a minimum period of **1 minute** but are commonly set to 5--15 min. For near-real-time, prefer metric alerts. |
| **Activity log alert severity** | Always **Sev 4 (Verbose)**; you cannot set a custom severity on activity log alerts. |
| **Network Watcher per region** | Network Watcher is auto-enabled per region, but if it was deleted/disabled you must **re-enable it in each region** before its tools work. |
| **Connection Troubleshoot vs Connection Monitor** | Connection Troubleshoot = **one-time** diagnostic. Connection Monitor = **continuous** scheduled monitoring with test groups. |
| **Dependency agent** | Required for the **Map** feature of VM Insights (process-level dependencies). AMA alone gives you performance data only. |
| **Diagnostic settings limit** | Up to **5 diagnostic settings** per resource. Each can target a different destination. |
| **DCR vs diagnostic settings** | DCRs are the newer path (lower latency, filtering, transforms). Diagnostic settings are legacy but still fully supported. Do not enable both for the same data to avoid duplication. |

---

## Sources

- [Azure Monitor metrics overview](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/data-platform-metrics)
- [Analyze metrics with Metrics Explorer](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/analyze-metrics)
- [Diagnostic settings in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings)
- [Data Collection Rules overview](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-overview)
- [Get started with log queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/get-started-queries)
- [Azure Monitor alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Types of Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types)
- [Action groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [Alert processing rules](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-processing-rules)
- [VM Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-overview)
- [Enable VM Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-enable)
- [Network Watcher overview](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview)
- [Connection Monitor overview](https://learn.microsoft.com/en-us/azure/network-watcher/connection-monitor-overview)
- [Network Insights](https://learn.microsoft.com/en-us/azure/network-watcher/network-insights-overview)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 training path: Monitor and back up Azure resources](https://learn.microsoft.com/en-us/training/paths/az-104-monitor-backup-resources/)
