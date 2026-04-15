# KQL Essentials

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 / AI-103 (reading prompts/completions telemetry)
> **One-liner:** Kusto Query Language is a **pipe-forward, read-only** language for logs — master 8 operators and you can answer 90% of operational questions.
> **Related:** [Logs](03-logs-and-log-analytics.md) · [App Insights](05-application-insights.md) · [Alerts](06-alerts-and-action-groups.md)

## Mental model

```
TableName
| operator1
| operator2
| ...
```

Each `|` pipes the previous result set to the next operator. No mutation, no joins unless asked. Shape of the result is always a tabular dataset.

## The core 8 operators

| Operator | Purpose | Example |
|----------|---------|---------|
| `where` | Filter rows | `| where TimeGenerated > ago(1h)` |
| `project` / `project-away` | Select/drop columns | `| project TimeGenerated, Message` |
| `extend` | Add computed columns | `| extend DurationMs = toreal(Duration)/1e6` |
| `summarize` | Aggregate | `| summarize count() by bin(TimeGenerated, 5m), ResultType` |
| `top` / `take` | Top-N / sample | `| top 10 by Duration desc` |
| `sort` (a.k.a. `order by`) | Sort | `| sort by TimeGenerated desc` |
| `join` | Combine tables | `| join kind=inner (Heartbeat) on Computer` |
| `render` | Visualize | `| render timechart` |

## Time is first-class

```kusto
// Last 24h, 5-min buckets, failures only
AppRequests
| where TimeGenerated > ago(24h)
| where Success == false
| summarize Failures = count() by bin(TimeGenerated, 5m), Name
| render timechart
```

Use `ago(...)`, `now()`, `startofday()`, `datetime_diff()`, and **always** bucket with `bin(TimeGenerated, 1m/5m/1h)` for charts.

## High-value recipes

### Top errors by dependency

```kusto
AppDependencies
| where TimeGenerated > ago(1h) and Success == false
| summarize Count = count() by Target, ResultCode
| top 10 by Count desc
```

### P95 latency per operation

```kusto
AppRequests
| where TimeGenerated > ago(6h)
| summarize P50 = percentile(DurationMs,50),
            P95 = percentile(DurationMs,95),
            P99 = percentile(DurationMs,99) by OperationName
| sort by P95 desc
```

### Join requests to exceptions

```kusto
AppRequests
| where Success == false
| join kind=leftouter (
    AppExceptions | project OperationId, ExceptionType=Type, OuterMessage
  ) on OperationId
```

### Parse unstructured text

```kusto
Syslog
| parse SyslogMessage with "user=" User " action=" Action *
```

### Anomaly detection (built-in ML)

```kusto
Perf
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render anomalychart
```

## Cross-workspace & cross-resource queries

```kusto
union
  AppRequests,
  workspace("other-ws").AppRequests,
  app("other-ai-resource").requests
| where TimeGenerated > ago(1h)
```

- `workspace("name")` — query another Log Analytics workspace (needs read).
- `app("resource")` — query another App Insights resource.
- Dedicated clusters remove the cross-workspace performance penalty.

## Functions, sets, and packs

- **Saved queries**: personal or shared per workspace.
- **Functions** (stored KQL with parameters): appear as callable names — treat them as views or stored procedures.
- **Query packs**: portable bundles of functions/queries that can be deployed via ARM/Bicep across workspaces.

## `let`, variables, and dynamic JSON

```kusto
let threshold = 500;
let lookback = 1h;
AppRequests
| where TimeGenerated > ago(lookback) and DurationMs > threshold
| extend Props = todynamic(CustomDimensions)
| extend Model = tostring(Props["gen_ai.model"])
| summarize count() by Model
```

## Performance tips

1. **Filter by time first** (`TimeGenerated > ago(...)`) — the engine partitions on time.
2. Filter on indexed columns (`_ResourceId`, `Type`, `Category`) before joining.
3. Prefer `summarize ... by bin()` over `sort | take` for charts.
4. Use `project` early to drop wide columns like `Properties` you don't need.
5. Avoid `contains` on very large datasets — prefer `has` (word-level, indexed) or `startswith`.

## Exam traps

- `==` is case-sensitive; for text use `=~` (case-insensitive equality) or `has_cs`/`has`.
- `has` ≠ `contains`. `has` matches whole terms and uses the index (fast); `contains` is substring (slow).
- `render timechart` requires the **first column to be a datetime** after summarize.
- Cross-workspace queries need `Log Analytics Reader` on the target.

## Sources

- [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Log queries in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-query-overview)
- [Cross-resource queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cross-workspace-query)
- [Query packs](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/query-packs)
