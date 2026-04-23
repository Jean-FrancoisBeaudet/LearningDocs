# Azure CLI — Monitor & Diagnostics

> **`az monitor` — metrics, Log Analytics KQL, diagnostic settings, activity log, alerts, action groups, autoscale, Application Insights.**
> **Related:** [AZURE/MONITOR](../MONITOR/) · [AZ-104 / 14 Monitoring](../AZ-104/14-monitoring.md)
> **Sources:** [`az monitor`](https://learn.microsoft.com/en-us/cli/azure/monitor) · [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)

---

## Table of contents

1. [Log Analytics workspaces](#log-analytics-workspaces)
2. [Running KQL from the CLI](#running-kql-from-the-cli)
3. [Diagnostic settings](#diagnostic-settings)
4. [Metrics](#metrics)
5. [Activity log](#activity-log)
6. [Action groups](#action-groups)
7. [Metric alerts](#metric-alerts)
8. [Log (scheduled-query) alerts](#log-scheduled-query-alerts)
9. [Autoscale](#autoscale)
10. [Application Insights](#application-insights)

---

## Log Analytics workspaces

```bash
az monitor log-analytics workspace create \
  -g rg-ops -n laws-prod \
  --location eastus \
  --sku PerGB2018 \
  --retention-time 90

az monitor log-analytics workspace list -o table
az monitor log-analytics workspace show -g rg-ops -n laws-prod

# Get shared keys (primary/secondary) — legacy agent wiring; prefer DCR-based agents
az monitor log-analytics workspace get-shared-keys -g rg-ops -n laws-prod

# Customer-managed key
az monitor log-analytics workspace update \
  -g rg-ops -n laws-prod \
  --key-vault-key-id https://kv-ops.vault.azure.net/keys/laws-cmk/v1

# Delete (soft-delete 14 days then purge)
az monitor log-analytics workspace delete -g rg-ops -n laws-prod --yes --force
```

### Solutions (legacy) and tables

```bash
# Tables (commitment tier, retention)
az monitor log-analytics workspace table list -g rg-ops --workspace-name laws-prod -o table
az monitor log-analytics workspace table update \
  -g rg-ops --workspace-name laws-prod -n AzureActivity \
  --retention-time 365 --total-retention-time 730
```

- Table-level retention overrides workspace default; long retention ≥ `90` days + archive past that.
- Interactive queries only work within the interactive retention window.

## Running KQL from the CLI

```bash
# One-shot query
az monitor log-analytics query \
  -w $(az monitor log-analytics workspace show -g rg-ops -n laws-prod --query customerId -o tsv) \
  --analytics-query "AzureActivity | take 10 | project TimeGenerated, Caller, OperationNameValue" \
  -o table

# Time-bounded (server-side)
az monitor log-analytics query \
  -w <workspaceCustomerId> \
  --analytics-query "AppRequests | where Success == false | summarize count() by OperationName" \
  --timespan 'PT1H' \
  -o json
```

- `-w` takes the **workspace customer ID (GUID)**, not the resource ID — capture via `.customerId`.
- `--timespan` accepts ISO 8601 durations (`PT1H`, `P1D`) or `start/end` boundaries.
- Result shape: `{tables:[{name, columns:[...], rows:[...]}]}` — for scripts pipe to `jq .tables[0].rows`.

### Query across workspaces

```kql
union workspace("laws-prod").AppRequests, workspace("laws-qa").AppRequests
| where TimeGenerated > ago(1h)
```

Cross-workspace / cross-resource requires `Log Analytics Reader` on every target.

## Diagnostic settings

Route resource logs + metrics to Log Analytics, Event Hub, or Storage.

```bash
# Inspect supported categories
az monitor diagnostic-settings categories list --resource <resourceId>

# Create
az monitor diagnostic-settings create \
  --name diag-to-laws \
  --resource <resourceId> \
  --workspace $(az monitor log-analytics workspace show -g rg-ops -n laws-prod --query id -o tsv) \
  --logs    '[{"category":"AuditEvent","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Multiple destinations in one setting
az monitor diagnostic-settings create \
  --name diag-dual \
  --resource <resourceId> \
  --workspace <lawsId> \
  --event-hub-name aad-logs --event-hub-rule <authRuleId> \
  --storage-account <storageAcctId> \
  --logs '[{"categoryGroup":"audit","enabled":true}]'

# List / update / delete
az monitor diagnostic-settings list --resource <resourceId> -o table
az monitor diagnostic-settings delete --name diag-to-laws --resource <resourceId>
```

- `categoryGroup` shortcut: `allLogs` or `audit` — easier than enumerating all categories.
- [Diagnostic settings policies](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings-policy) (`DeployIfNotExists`) are the way to scale this to hundreds of resources.

## Metrics

```bash
# List metric definitions for a resource
az monitor metrics list-definitions --resource <resourceId> -o table

# Query a time series
az monitor metrics list \
  --resource <storageAccountId> \
  --metric "Transactions,Ingress,Egress" \
  --interval PT1M --aggregation Total Average \
  --start-time 2026-04-21T00:00:00Z \
  --end-time   2026-04-21T01:00:00Z \
  --filter "ApiName eq 'GetBlob' and GeoType eq 'Primary'"

# Just one stat, scalar
az monitor metrics list \
  --resource <vmId> --metric "Percentage CPU" --aggregation Average --interval PT5M \
  --query "value[0].timeseries[0].data[-1].average"
```

- `--aggregation` accepts `Average`, `Minimum`, `Maximum`, `Total`, `Count`.
- `--interval` ISO 8601: `PT1M`, `PT5M`, `PT1H`, `P1D`. Granularity must match the metric's supported list.
- Unsupported combinations (e.g. 1-minute granularity on a metric aggregated at 5-min) return empty series — check `list-definitions`.

## Activity log

```bash
# By RG, last 24 h
az monitor activity-log list \
  --resource-group rg-dev \
  --start-time $(date -u -d '24 hours ago' +'%Y-%m-%dT%H:%M:%SZ') \
  --query "[].{time:eventTimestamp, caller:caller, op:operationName.localizedValue, status:status.localizedValue}" \
  -o table

# By resource
az monitor activity-log list --resource-id <resourceId>

# By correlation ID (all events for a single deployment)
az monitor activity-log list --correlation-id <guid>

# By caller
az monitor activity-log list --caller alice@contoso.com
```

- Activity log retains **90 days** in-portal; route to LAW via diagnostic setting for longer.
- Same data is queryable in KQL via the `AzureActivity` table once diagnosticSettings is wired.

## Action groups

```bash
az monitor action-group create \
  -g rg-ops -n ag-oncall \
  --short-name oncall \
  --email-receiver name=team email=oncall@contoso.com useCommonAlertSchema=true \
  --sms-receiver name=alice country-code=1 phone=5551234567 \
  --webhook-receiver name=pd service-uri=https://events.pagerduty.com/... useCommonAlertSchema=true \
  --azure-app-push-receiver name=team email=team@contoso.com

az monitor action-group list -g rg-ops -o table
az monitor action-group show -g rg-ops -n ag-oncall

# Add or remove receivers without recreating
az monitor action-group update -g rg-ops -n ag-oncall \
  --add-action email backup name=backup email=oncall-backup@contoso.com

# Delete
az monitor action-group delete -g rg-ops -n ag-oncall
```

- `useCommonAlertSchema=true` is a must-have — the consistent payload shape makes downstream routing (Logic Apps, PagerDuty, ServiceNow) drastically simpler.

## Metric alerts

```bash
# CPU > 80% for 5 min on a VM
az monitor metrics alert create \
  -g rg-ops -n alert-vm-cpu \
  --scopes $(az vm show -g rg-dev -n vm-web --query id -o tsv) \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m --evaluation-frequency 1m \
  --severity 2 --description "VM CPU saturation" \
  --action ag-oncall

# Multi-resource (all VMs in an RG)
az monitor metrics alert create \
  -g rg-ops -n alert-vm-cpu-all \
  --scopes $(az group show -g rg-dev --query id -o tsv) \
  --target-resource-type "Microsoft.Compute/virtualMachines" \
  --target-resource-region eastus \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m --evaluation-frequency 1m \
  --action ag-oncall

# List / update
az monitor metrics alert list -g rg-ops -o table
az monitor metrics alert update -g rg-ops -n alert-vm-cpu --severity 3
az monitor metrics alert delete -g rg-ops -n alert-vm-cpu
```

- Severities: 0 (Critical) – 4 (Verbose).
- `--window-size` / `--evaluation-frequency` ISO 8601 — shorter windows raise eval cost (each condition = one evaluation × N resources).

## Log (scheduled-query) alerts

```bash
# KQL-based alert
az monitor scheduled-query create \
  -g rg-ops -n alert-5xx-spike \
  --scopes $(az monitor log-analytics workspace show -g rg-ops -n laws-prod --query id -o tsv) \
  --condition "count 'AppRequests | where ResultCode startswith \"5\" | summarize count() by OperationName' > 20" \
  --window-size 10m --evaluation-frequency 5m \
  --severity 1 --description "HTTP 5xx spike" \
  --action-group $(az monitor action-group show -g rg-ops -n ag-oncall --query id -o tsv)

# List
az monitor scheduled-query list -g rg-ops -o table
```

- Scheduled-query alerts can target a LAW, AppInsights resource, or (cross-resource) a subscription.
- Time grain of the query **inside** KQL should align with `--window-size` to avoid under-/over-counting.

## Autoscale

```bash
# Attach to a VMSS
az monitor autoscale create \
  -g rg-dev \
  --resource $(az vmss show -g rg-dev -n vmss-web --query id -o tsv) \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-vmss-web \
  --min-count 2 --max-count 10 --count 3

# Scale-out rule
az monitor autoscale rule create \
  -g rg-dev --autoscale-name autoscale-vmss-web \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Scale-in rule
az monitor autoscale rule create \
  -g rg-dev --autoscale-name autoscale-vmss-web \
  --condition "Percentage CPU < 30 avg 15m" \
  --scale in 1

# Time-based profile (scheduled scale)
az monitor autoscale profile create \
  -g rg-dev --autoscale-name autoscale-vmss-web -n business-hours \
  --min-count 4 --max-count 20 --count 6 \
  --start 2026-04-21T06:00:00 --end 2026-04-21T22:00:00 \
  --timezone "Eastern Standard Time" --recurrence week mon tue wed thu fri
```

- `--scale out/in <N>` sets change-count style scaling. `--scale to <M>` sets target instance count.
- Default cooldown: 5 minutes; adjust with `--cooldown 5`.

## Application Insights

```bash
# Workspace-based (recommended — stores data in a LAW)
az monitor app-insights component create \
  --app appi-web -g rg-ops --location eastus --kind web \
  --workspace $(az monitor log-analytics workspace show -g rg-ops -n laws-prod --query id -o tsv)

# Get connection string
az monitor app-insights component show -g rg-ops --app appi-web --query connectionString -o tsv

# Query (KQL)
az monitor app-insights query \
  --app appi-web -g rg-ops \
  --analytics-query "requests | summarize count() by bin(timestamp, 1h), resultCode | order by timestamp desc" \
  --offset 1h

# Event/Funnels/Metrics
az monitor app-insights metrics show \
  --app appi-web -g rg-ops \
  --metric "requests/failed" --interval PT5M

# Purge (GDPR requests)
az monitor app-insights component purge \
  --app appi-web -g rg-ops \
  --table traces --filters '[{"column":"user_Id","operator":"==","value":"alice-user-id"}]'

# Smart detection rules
az monitor app-insights component smart-detection list --app appi-web -g rg-ops -o table
```

- Workspace-based AppInsights is **required** for new resources — classic (standalone) is retired.
- For `.NET`, `Java`, `Node`, Python autoinstrumentation, the AppInsights SDK reads the connection string from env var `APPLICATIONINSIGHTS_CONNECTION_STRING`.
- Some commands (`app-insights`) require the extension: `az extension add --name application-insights`.

## Gotchas

- **Workspace `customerId` vs resource `id`** trip up users — `az monitor log-analytics query` needs the customer ID (GUID), `--workspace` on `diagnostic-settings` needs the full resource ID.
- **Metric alerts are evaluated per resource** in a multi-resource alert — billing scales with resource count.
- **Diagnostic settings are per-resource** — there is no "apply to RG." Scale via Azure Policy `DeployIfNotExists`.
- **Log alerts on rapidly evolving tables** (e.g., `AzureDiagnostics` before normalized schemas) can silently break when schema changes. Test after platform updates.
- **KQL from CLI escapes double quotes unpleasantly** — put the query in a file and use `--analytics-query @query.kql`.
- **Activity log retention is 90 days in-portal**. Longer retention requires diagnostic setting → LAW or Storage.
- **Autoscale rule conditions are ANDed** inside a profile; multiple rules create an OR across conditions — design carefully for flapping.
- **Application Insights sampling** (adaptive, SDK-level) silently drops data — a query showing "0 requests" may mean zero traffic or aggressive sampling. Check `AppSystemEvents` or raise sampling thresholds.

## Sources

- [`az monitor`](https://learn.microsoft.com/en-us/cli/azure/monitor) · [`az monitor log-analytics`](https://learn.microsoft.com/en-us/cli/azure/monitor/log-analytics) · [`az monitor diagnostic-settings`](https://learn.microsoft.com/en-us/cli/azure/monitor/diagnostic-settings) · [`az monitor metrics`](https://learn.microsoft.com/en-us/cli/azure/monitor/metrics) · [`az monitor alert`](https://learn.microsoft.com/en-us/cli/azure/monitor/alert) · [`az monitor autoscale`](https://learn.microsoft.com/en-us/cli/azure/monitor/autoscale) · [`az monitor app-insights`](https://learn.microsoft.com/en-us/cli/azure/monitor/app-insights)
- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [KQL language reference](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)
- [Common alert schema](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema)
- [Workspace-based Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource)
