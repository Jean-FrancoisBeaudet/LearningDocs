# Azure Monitor — Overview

> **Exam mapping:** AZ-204 *(Instrument solutions to support monitoring and logging)* · AZ-305 *(Design monitoring solutions)* · AI-102 / AI-103 *(Monitor AI solutions)* · AI-300 *(Observability for GenAIOps)*
> **One-liner:** The single Azure-native platform that collects, correlates, analyzes, visualizes, and acts on **metrics, logs, traces, and changes** from every Azure, hybrid, and on-prem resource.
> **Related:** [Data sources & collection](01-data-sources-and-collection.md) · [Metrics](02-metrics.md) · [Logs & Log Analytics](03-logs-and-log-analytics.md) · [Application Insights](05-application-insights.md) · [Alerts](06-alerts-and-action-groups.md)

## What Azure Monitor is

Azure Monitor is an **umbrella service**, not a single product. It unifies what used to be several tools (OMS, Log Analytics, Application Insights, Azure Diagnostics) into one platform with two primary data stores and several consumers on top.

```
                 ┌────────────────────── Sources ──────────────────────┐
 Azure resources │ Platform metrics · Activity log · Resource logs     │
 OS / apps       │ Guest OS metrics/logs (AMA) · App Insights / OTel  │
 Hybrid / Arc    │ Custom logs · Syslog · Windows Events · Prometheus  │
                 └────────────────────────┬────────────────────────────┘
                                          │
                               ┌──────────┴──────────┐
                               ▼                     ▼
                       ┌───────────────┐     ┌──────────────────┐
                       │ Metrics store │     │  Logs store      │
                       │ (time-series) │     │ (Log Analytics)  │
                       └──────┬────────┘     └──────┬───────────┘
                              │                     │
       ┌──────────────────────┴──────┬──────────────┴──────────────────┐
       ▼                             ▼                                 ▼
   Visualize              Alert & Respond                      Analyze
   (Dashboards,           (Alerts, Action Groups,              (KQL, Workbooks,
    Workbooks,             Autoscale, Logic Apps,               Insights,
    Managed Grafana)       Functions, Runbooks)                 App Insights UI)
```

## The 4 pillars

| Pillar | What happens | Surfaces |
|--------|--------------|----------|
| **Collect** | Diagnostic Settings, Data Collection Rules (DCR), AMA, SDK/OpenTelemetry | `01` |
| **Analyze** | KQL over Log Analytics, Metrics Explorer, Insights (Container/VM/Network), App Insights | `03`, `04`, `05`, `09` |
| **Respond** | Alerts + Action Groups, Autoscale, Alert Processing Rules | `06`, `07` |
| **Visualize** | Azure Dashboards, Workbooks, Managed Grafana, Power BI | `08` |

## Two data stores — know the difference

| | **Metrics** | **Logs** |
|---|---|---|
| Schema | Numeric time-series + dimensions | Typed table rows (Kusto schema) |
| Query | Metrics Explorer (simple aggregations) | **KQL** (rich joins, regex, ML) |
| Latency | Near real-time (< 1 min) | 1–5 min typical |
| Retention | **93 days** (platform metrics) | Up to **12 years** (archive) |
| Cost model | Free for platform; custom metrics billed per time-series | Per-GB ingestion + retention |
| Use | Dashboards, autoscale, fast alerts | Forensics, correlation, compliance |

See [`02-metrics.md`](02-metrics.md) and [`03-logs-and-log-analytics.md`](03-logs-and-log-analytics.md).

## Distributed Tracing — the "third pillar"

The industry-standard **three pillars of observability** are **Metrics, Logs, and Distributed Tracing**. Azure Monitor covers all three — tracing just doesn't have its own data store; it lives inside the **Logs store** as correlated rows in Application Insights tables.

| Pillar | Azure Monitor home | Key tables / tools |
|--------|--------------------|--------------------|
| **Metrics** | Metrics store (time-series) | Metrics Explorer, platform & custom metrics |
| **Logs** | Logs store (Log Analytics) | KQL, `AppTraces`, `AzureDiagnostics`, `Syslog`, etc. |
| **Distributed Tracing** | Logs store (via Application Insights) | `AppRequests` + `AppDependencies` correlated by `operation_Id` / `operation_ParentId` |

### How it works

Every telemetry item carries **`operation_Id`** (= W3C `trace-id`) and **`operation_ParentId`** (= `parent-span-id`). This links requests, dependency calls, exceptions, and logs into a single end-to-end transaction tree across services.

```
Browser → Frontend API → Order Service → Cosmos DB
(PageView)  (Request)    (Dep→Request)   (Dependency)
   └──────────── all share the same operation_Id ──────────────┘
```

| OpenTelemetry concept | App Insights table | Notes |
|-----------------------|--------------------|-------|
| Trace | Correlated set via `operation_Id` | End-to-end transaction |
| Span (server) | `AppRequests` | Incoming call |
| Span (client) | `AppDependencies` | Outgoing call (HTTP, SQL, Service Bus, etc.) |
| SpanEvent | `AppExceptions` / `AppTraces` | Exception or log attached to a span |

### Portal surfaces for tracing

- **Application Map** — auto-generated topology with failure/latency hotspots.
- **End-to-end transaction search** — trace waterfall view showing the full span tree.
- **Failures / Performance blades** — drill from aggregates into individual traces.

### Cross-service / cross-workspace tracing

If services emit telemetry to **different** Log Analytics workspaces, trace stitching requires **cross-workspace queries**: `workspace('other').AppRequests | where operation_Id == "..."`. For full-fidelity tracing, a **single workspace** is preferred (AZ-305 design point).

> **Why "2 stores" not "3 pillars"?** Azure frames its architecture around storage engines. Traces decompose into table rows in the Logs store — they're a query-time reconstruction, not a separate backend. The distinction matters for cost modeling (traces are billed as log ingestion) and for exam answers.

## Related (but separate) products

| Service | Scope | Relationship |
|---------|-------|--------------|
| **Microsoft Sentinel** | SIEM/SOAR on top of Log Analytics | Uses the same workspace; adds threat detection, hunting, incident mgmt. |
| **Microsoft Defender for Cloud** | CSPM + CWP | Writes security signals to a workspace; complements Monitor. |
| **Azure Advisor / Service Health** | Best-practice + platform incident feeds | Surface inside Monitor, not part of it. |

Exam line: **Monitor = observability. Sentinel = security analytics. Defender = posture + workload protection.** Never mix them in an answer.

## Exam mapping cheatsheet

| Exam | What they ask |
|------|---------------|
| **AZ-204** | Instrument App Insights (SDK/OTel), custom metrics/events, Live Metrics, KQL basics, alert rule vs action group, autoscale rules for App Service/VMSS. |
| **AZ-305** | Workspace topology (centralized vs federated), retention/archive design, alert routing, DR for monitoring, cost control, regulatory retention. |
| **AI-102 / AI-103** | Diagnostic settings for Azure OpenAI / AI Search / Document Intelligence; capturing prompts/completions; content safety telemetry. |
| **AI-300** | End-to-end GenAIOps: evaluation metrics to Monitor, dashboarding quality/safety KPIs, alerting on drift. |

## Exam traps

- Azure Monitor **is** Application Insights + Log Analytics + Metrics — they are not separate products.
- Platform metrics retention is always **93 days** and cannot be extended; archive via Diagnostic Setting → workspace.
- "Azure AD" is now **Microsoft Entra ID** in all current docs and exam questions.

## Sources

- [Azure Monitor overview (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Data platform](https://learn.microsoft.com/en-us/azure/azure-monitor/data-platform)
- [Service limits](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits)
- [Monitor vs Sentinel vs Defender](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
