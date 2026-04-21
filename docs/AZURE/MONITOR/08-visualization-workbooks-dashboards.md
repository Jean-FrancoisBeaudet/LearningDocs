# Visualization — Workbooks, Dashboards, Grafana

> **Exam mapping:** AZ-204 · AZ-305
> **One-liner:** Four ways to put Monitor data in front of humans — pick based on **audience**, **interactivity**, and **multi-source** needs.
> **Related:** [Metrics](02-metrics.md) · [Logs](03-logs-and-log-analytics.md) · [KQL](04-kql-essentials.md)

## Decision table

| You want… | Use |
|-----------|-----|
| Quick, personal tile layout from portal pieces | **Azure Dashboard** |
| Parameterized, interactive, multi-source reports with narrative | **Workbook** |
| Unified view spanning Azure, AWS, on-prem, Prometheus, Elastic | **Azure Managed Grafana** |
| Self-serve BI, drillable reports for business users | **Power BI** (with Log Analytics connector) |
| Live, sub-second in-app pulse during debugging | **Live Metrics** (App Insights) |

## Azure Dashboards

- Portal-native tiles (charts, markdown, metrics, KQL tiles).
- Shared via ARM (resource type `Microsoft.Portal/dashboards`).
- **Limited interactivity** — no parameters, no drill-down beyond what a tile offers.
- Best for *fixed* team views pinned next to resource blades.

## Workbooks

The flagship Monitor authoring surface. Think *notebook-meets-BI*.

Capabilities:

- **Parameters** (time range, subscription, resource pickers, dropdowns fed by KQL).
- **Multi-source**: KQL against workspaces / App Insights, ARM queries, metrics, Azure Resource Graph, Log Analytics in other workspaces.
- **Visualizations**: grids, charts, maps, honeycomb, tiles, tree grids, JSON.
- **Conditional formatting** (traffic-light grids).
- **Step-by-step narrative** + text/markdown blocks.
- **Templates**: dozens of built-ins per service (VM Insights, AKS, Storage).
- **Publish as ARM/Bicep** (resource type `Microsoft.Insights/workbooks`) — treat as code.

When to prefer Workbook over Dashboard: whenever the user will **interact** (filter, drill, toggle) or the data crosses resources.

## Azure Managed Grafana

Fully managed Grafana OSS with Entra ID auth and first-class Azure data sources:

- Azure Monitor Metrics, Azure Monitor Logs, **Managed Prometheus**, Azure Data Explorer, Resource Graph.
- Also native Grafana plugins for Prometheus, Loki, PostgreSQL, Elastic, etc.
- SKUs: **Essential** (smaller, free tier) vs **Standard** (SLA 99.9, zone-redundant, Private Link).

Use Managed Grafana when:

- You already run Grafana and want to lift-and-shift.
- You need unified dashboards across clouds / OSS stack.
- You want PromQL for AKS + KQL for Azure side-by-side.

Alert note: **author alerts in Azure Monitor, not Grafana** — Grafana-side alerts on Azure sources are not officially supported by Microsoft.

## Power BI

- Log Analytics → Power BI connector (M query over KQL).
- Use for scheduled executive reports, drill-through BI, joins with non-telemetry datasets (finance, CRM).
- Not for real-time or alerting.

## Authoring as code

```bicep
resource wb 'Microsoft.Insights/workbooks@2022-04-01' = {
  name: guid('my-workbook')
  location: resourceGroup().location
  kind: 'shared'
  properties: {
    displayName: 'Platform Health'
    serializedData: loadTextContent('workbook.json')
    category: 'workbook'
    sourceId: logAnalyticsWorkspaceId
  }
}
```

Store the JSON in Git next to code; deploy via Bicep pipelines alongside app infra.

## Exam traps

- **Workbooks vs Dashboards**: dashboards are static tiles; workbooks are parameterized. When the exam says "interactive parameters" → workbook.
- **Managed Grafana ≠ Azure Dashboard** — Grafana is a separate service SKU with its own RBAC and URL.
- Grafana **alerts on Azure data sources** → not officially supported; use Monitor alerts.
- Live Metrics **streams** — it's not a dashboard replacement and has no storage.

## Sources

- [Workbooks overview](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview)
- [Azure Dashboards](https://learn.microsoft.com/en-us/azure/azure-portal/azure-portal-dashboards)
- [Azure Managed Grafana](https://learn.microsoft.com/en-us/azure/managed-grafana/overview)
- [Power BI connector for Log Analytics](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-powerbi)
- [Deploy workbooks with ARM/Bicep](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-automate)
