# Insights — curated monitoring experiences

> **Exam mapping:** AZ-204 · AZ-305
> **One-liner:** "Insights" are **pre-built Monitor packs** that combine collection, curated workbooks, and alerts for a given service type — turn them on instead of building from scratch.
> **Related:** [Data sources](01-data-sources-and-collection.md) · [Workbooks](08-visualization-workbooks-dashboards.md)

## What "Insights" means

Before DCRs, Monitor shipped "solutions" via a Management Pack model. Those are being retired. Modern **Insights** are portal experiences backed by:

- Azure Monitor Agent + DCRs
- Curated Log Analytics tables
- Built-in Workbooks
- Optional Managed Prometheus + Managed Grafana integration

## The main Insights to know

| Insight | Target | Key tables / features |
|---------|--------|----------------------|
| **Container Insights** | AKS, Arc-enabled Kubernetes, AKS Edge | `ContainerLogV2`, `KubePodInventory`, `Perf`, `InsightsMetrics`. Now integrates with **Managed Prometheus + Managed Grafana** for metrics. Deployed via `omsagent`/`ama-logs` DaemonSet or Azure Monitor pipeline. |
| **VM Insights** | VM, VMSS, Arc-enabled servers | Perf counters (CPU/Mem/Disk/Net), **Map feature** (process + connection topology) via Dependency Agent. Needs AMA + DCR; Map is optional. |
| **Network Insights** | VNet, NSG, ExpressRoute, Load Balancer, Firewall, Application Gateway | Topology maps, NSG flow log analytics, connection monitor. |
| **SQL Insights** *(preview → replaced by "Database Watcher")* | Azure SQL DB, MI, SQL on VM | WMI/DMV perf metrics via monitoring VM + AMA. Check for **Database Watcher for Azure SQL** (newer). |
| **Storage Insights** | Storage accounts | Capacity, transactions, latency, availability across storage services. |
| **Key Vault Insights** | Key Vaults | Request volume, failures, latency, identity. |
| **Cosmos DB Insights** | Cosmos DB accounts | RU/s usage, throttled requests, storage, per-partition hot keys. |
| **App Service Insights** | App Service plans | Active instances, HTTP queue, CPU, memory, restarts. |
| **Cache for Redis Insights** | Redis | Ops/sec, cache hits, memory, latency. |
| **Azure OpenAI / AI Foundry monitoring** | AOAI resources | Tokens in/out per deployment, throttling, content-safety filter hits, latency. Configure via diagnostic settings. |

## Container Insights — exam-weighted

- Onboard from AKS blade → enables AMA + preset DCR.
- **Cost** heavily depends on **container stdout/stderr verbosity** — tune via DCR transforms (drop noisy namespaces) or **ContainerLogV2** schema with table plans.
- **Managed Prometheus** for scraping cluster metrics, **Managed Grafana** for dashboards; both now included in AKS monitoring blueprint.
- Live Data view for near-real-time pod logs.

## VM Insights — Map feature

- Needs **Dependency Agent** in addition to AMA.
- Builds `VMConnection` / `VMProcess` tables → topology queries.
- Helpful for application dependency discovery pre-migration.

## Migrating off legacy solutions

Anything relying on MMA-based "Management Solutions" (legacy Security & Audit, legacy Update Management tied to MMA, legacy `ContainerLog` table) needs migration:

- `ContainerLog` → `ContainerLogV2`.
- MMA-based VM Insights → AMA-based VM Insights.
- Classic App Insights → Workspace-based + OTel.
- Update Management (legacy) → **Azure Update Manager**.

## Exam traps

- Insights are **free to enable**; you pay for the **data they ingest**.
- Container Insights' cost is dominated by container log volume — **filter at the DCR**, not downstream.
- **`ContainerLog` vs `ContainerLogV2`**: new deployments default to V2 (better schema, enables Basic plan).
- Map feature in VM Insights requires the **Dependency Agent**, which is separate from AMA.
- SQL Insights preview content may appear on AZ-305 mocks — but the current recommended path is **Database Watcher for Azure SQL**.

## Sources

- [Azure Monitor Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/insights/insights-overview)
- [Container Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview)
- [VM Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-overview)
- [Network Insights](https://learn.microsoft.com/en-us/azure/network-watcher/network-insights-overview)
- [Database Watcher for Azure SQL](https://learn.microsoft.com/en-us/azure/azure-sql/database-watcher-overview)
- [Azure OpenAI monitoring](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/monitor-openai)
