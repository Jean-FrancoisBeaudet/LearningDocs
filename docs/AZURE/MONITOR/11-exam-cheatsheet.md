# Exam Cheatsheet

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 / AI-103 · AI-300
> **One-liner:** The tiny distinctions Azure exams love to test about Monitor.
> **Related:** all other files in this folder.

## Service boundaries — don't mix them up

| Product | Role |
|---------|------|
| **Azure Monitor** | Observability (metrics/logs/traces/changes + alerts + autoscale). |
| **Microsoft Sentinel** | SIEM/SOAR on top of a Log Analytics workspace. |
| **Microsoft Defender for Cloud** | CSPM + workload protection; writes signals to a workspace. |
| **Azure Advisor / Service Health / Resource Health** | Surface inside Monitor; source of Activity-Log-style alerts. |

## Top 30 exam traps

### Data platform

1. **Azure Monitor workspace ≠ Log Analytics workspace.** Former = Prometheus metrics; latter = logs.
2. Platform metrics retention = **93 days, fixed**.
3. Activity Log retention is **90 days** in the native store; longer requires Diagnostic Setting to workspace/Storage.
4. **Diagnostic Settings** = per resource, up to 5 per resource. Activity Log diagnostic setting = subscription scope.
5. MMA/OMS agents are **retired**; ingestion shuts down after **March 2, 2026**. Use **Azure Monitor Agent (AMA) + DCR**.
6. HTTP Data Collector API retired — use **Logs Ingestion API** via DCR/DCE.

### Logs & KQL

7. **Table plans**: Analytics (default), Basic (cheap, 30-day query, single-table), Auxiliary (cheapest, custom table only, API-only).
8. Archive (up to 12 yrs) needs **search job** or **restore** to query.
9. Alerts **cannot target** Basic/Auxiliary tables directly.
10. `has` (indexed, whole-term) >> `contains` (slow substring).
11. `==` is case-sensitive; use `=~` for case-insensitive.
12. Workspace region is fixed at creation.

### Application Insights

13. Classic App Insights is **retired** — use **workspace-based**.
14. **Connection String**, not Instrumentation Key alone.
15. **OpenTelemetry Distro** is the recommended instrumentation path.
16. **URL Ping test retires Sept 30, 2026** → **Standard test** or **TrackAvailability** custom.
17. **Adaptive sampling** preserves end-to-end correlation; ingestion sampling does not save egress.

### Metrics & alerts

18. **Metric alert** < 1 min latency; **log alert** 1–5 min and costs per query.
19. **Dynamic thresholds** are metric-alert only.
20. **Smart Detection** is App-Insights-only.
21. **Activity Log alerts** are free; use for Service Health / policy / RBAC.
22. Action group **unlimited per subscription**; per-AG channel limits (e.g. 1 000 emails).
23. **Alert processing rules** for maintenance suppression and at-scale action group assignment.
24. **Prometheus alerts** require Azure Monitor workspace + rule group (PromQL).

### Autoscale

25. Functions Consumption, Container Apps, AKS — **NOT** Monitor Autoscale.
26. Scale-in threshold must be strictly and comfortably below scale-out (avoid flapping).
27. **Predictive autoscale** = VMSS only.

### Visualization

28. **Workbooks** = parameterized interactive reports; **Dashboards** = static tiles.
29. **Azure Managed Grafana** is best for multi-cloud / OSS unification; author alerts in Monitor, not Grafana.

### Cost

30. Biggest cost lever is **DCR transformation filtering** — drop rows before ingest; **retention reduction is destructive**.

## Quick picker tables

### "Which alert type?"

| Signal | Pick |
|--------|------|
| Resource metric, fast, < 1 min | **Metric alert** |
| KQL query across tables | **Log alert** |
| Subscription control-plane event | **Activity Log alert** |
| Azure outage notification | **Service Health alert** |
| Prometheus scrape on AKS | **Prometheus alert** |
| App Insights anomaly (ML) | **Smart Detection** |

### "Where do I put this data?"

| Data | Destination |
|------|-------------|
| Numeric, near-real-time, autoscale/alerting | Metrics store |
| Forensics, joins, long retention | Log Analytics (Analytics) |
| Verbose access logs you rarely query | Log Analytics (Basic) |
| Compliance-only, cheapest | Log Analytics (Auxiliary) + Archive |
| Stream to third-party SIEM | Diagnostic Setting → Event Hub |
| Cold archive, audit | Diagnostic Setting → Storage |

### "Which agent?"

| Need | Pick |
|------|------|
| New deployment | **Azure Monitor Agent (AMA)** |
| VM Insights Map feature | AMA + **Dependency Agent** |
| Prometheus scrape on AKS | Managed Prometheus via AKS monitoring addon |
| Legacy (MMA/OMS) | **Retired** — migrate before March 2, 2026 |

## One-line summaries

- **Metrics are fast, cheap, short-lived. Logs are rich, expensive, long-lived.**
- **Diagnostic Settings put Azure data into Monitor. DCRs put everything else in.**
- **App Insights = APM feature of Monitor, not a separate product.**
- **Workbooks = code-as-dashboards. Ship them in Bicep.**
- **Autoscale = horizontal, and only on specific services.**
- **Cost = volume. Trim with DCR transforms and table plans.**

## Sources

- [AZ-204 skills outline](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)
- [AZ-305 skills outline](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305)
- [AI-102 skills outline](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-102)
- [Azure Monitor best practices](https://learn.microsoft.com/en-us/azure/azure-monitor/best-practices)
- [Service limits](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits)
