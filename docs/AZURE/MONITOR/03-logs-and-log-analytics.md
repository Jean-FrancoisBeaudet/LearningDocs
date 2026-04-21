# Logs & Log Analytics

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 / AI-103
> **One-liner:** A schema-on-write columnar store (Kusto engine) that keeps every signal queryable via **KQL** for up to **12 years**, with per-table cost plans that let you tune price vs query power.
> **Related:** [Data sources](01-data-sources-and-collection.md) · [KQL](04-kql-essentials.md) · [App Insights](05-application-insights.md) · [Cost](10-cost-and-optimization.md)

## The workspace

A **Log Analytics workspace** is the unit of:

- Ingestion endpoint
- Billing
- RBAC and data access
- Geographic residency (region is fixed at create time)

Workspaces hold tables (e.g. `Heartbeat`, `AppRequests`, `AzureDiagnostics`, custom `MyApp_CL`). The same workspace is used by **Microsoft Sentinel** and **Defender for Cloud** — check the pricing implications before combining.

## Workspace design patterns

| Pattern | When to use | Watch out for |
|---------|------------|---------------|
| **Single central workspace** | Small org, single sovereign region, uniform access. | Cross-subscription RBAC, region lock-in. |
| **Per-environment** (dev/test/prod) | Clean cost & retention separation. | Cross-env queries need cross-workspace KQL. |
| **Per-business-unit / per-region** (federated) | Data residency, chargeback, large orgs. | Correlating incidents across workspaces. |
| **Sentinel-dedicated + Monitor separate** | Defenders want full logs; ops team on a cheaper plan. | Duplicate ingestion cost unless you route carefully. |

Exam line: *"Minimize complexity while meeting residency/RBAC"* → **default to a single workspace**; split only for concrete compliance or scale reasons.

## Table plans (2025+ model)

Each table has a **plan** that controls query power and price.

| Plan | Query | Ingest cost | Retention | Use case |
|------|-------|-------------|-----------|----------|
| **Analytics** | Full KQL, alerts, dashboards | Standard | **Up to 2 yrs interactive**, then **up to 12 yrs archive** | Hot operational data, security signals |
| **Basic** | Single-table, last **30 days** only, no alerts on it | ~1/5 of Analytics | 30-day interactive + archive | High-volume verbose logs (NGINX access, CDN) |
| **Auxiliary** *(custom tables only, via API)* | Slow single-table search | Very low | Long retention (up to 12 yrs) | Audit / compliance data rarely queried |

Plan rules to memorize:

- **Analytics** supports everything.
- **Basic** and **Auxiliary** do **not** support scheduled alerts, summary rules, or multi-table joins in normal queries — use **search jobs** or **restore** to promote data temporarily.
- **Auxiliary** is create-time only (custom tables), **cannot** be switched.
- All DCR-based custom tables can use Basic; many built-in Azure tables can too.

## Retention and archive

```
Interactive retention (queryable by KQL)       Archive (search job / restore required)
├── 4 to 730 days (Analytics) ─────────────────┼── up to 12 years ──────────────┤
├── 30 days fixed (Basic)                       │
```

- Default **30 days interactive** is included in ingestion price.
- **Archived data** is cheap to store but requires a **search job** (asynchronous, scans to a new table) or a **restore** (makes data queryable for N days) to access.
- Set retention **per table** — not only workspace default.

## Ingestion paths

1. **Diagnostic Settings** (Azure resources)
2. **Azure Monitor Agent + DCR** (VMs, Arc servers, Syslog, custom logs)
3. **Logs Ingestion API** (custom apps, third-party shippers) — replaces the retired HTTP Data Collector API
4. **Azure Functions / Logic Apps** writing via the ingestion API
5. **Solutions/Insights** (Container Insights, Sentinel connectors)

## Dedicated Clusters & commitment tiers

- **Commitment tiers**: 100, 200, 300, 400, 500 GB/day → up to 5 TB/day for ~15–30% discount. Billed whether you hit the tier or not.
- **Dedicated Cluster** (≥ 500 GB/day): Customer-Managed Keys (CMK), Lockbox, cross-workspace query without cross-resource penalties, double encryption. Multiple workspaces can be linked to one cluster.

## Security & access

- Scope data access with **Resource-Context RBAC** (user sees only logs for resources they have read on) or **Workspace-Context RBAC** (classic).
- **Table-level RBAC** to hide sensitive tables (e.g. `SigninLogs`).
- Private ingestion via **Azure Monitor Private Link Scope (AMPLS)** + Private Endpoint.

## Exam traps

- **`PerGB2018`** is the modern pricing tier name; legacy names still appear in old practice exams.
- A table's **plan** and the workspace's **retention default** are independent settings.
- Sentinel attaches to a workspace — enabling Sentinel changes the cost model (adds Sentinel analytics price on top of ingestion).
- "Archive" data is **not** directly queryable — you must **search job** or **restore** first.
- Workspace **region** is fixed at creation.

## Sources

- [Log Analytics workspace overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview)
- [Table plans (Analytics / Basic / Auxiliary)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-table-plans)
- [Manage retention](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure)
- [Search jobs](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/search-jobs)
- [Dedicated clusters](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-dedicated-clusters)
- [Cost calculations](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs)
