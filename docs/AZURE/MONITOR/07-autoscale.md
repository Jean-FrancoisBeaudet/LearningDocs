# Autoscale

> **Exam mapping:** AZ-204 · AZ-305
> **One-liner:** Autoscale is an Azure Monitor feature that **adds or removes instances** of supported compute based on metric rules or a schedule — it is *not* vertical scaling and it is *not* the same as KEDA.
> **Related:** [Metrics](02-metrics.md) · [Alerts](06-alerts-and-action-groups.md)

## What supports Azure Monitor Autoscale

| Service | Scope |
|---------|-------|
| **Virtual Machine Scale Sets** (Uniform & Flexible) | Instance count |
| **Azure App Service** (Standard+/Premium) | Worker count per plan |
| **Azure Cloud Services** (legacy) | Instance count |
| **API Management** | Unit count (Premium/Isolated) |
| **Azure Data Explorer** clusters | Instances |
| **Integration Services Environment** | Units (retiring — Standard Logic Apps instead) |

**Not covered by Monitor Autoscale** (use their own engines):

- **Azure Functions Consumption / Premium** — internal scaler.
- **Azure Container Apps** / **AKS** — **KEDA** / HPA / Cluster Autoscaler.
- **Cosmos DB autoscale** — RU/s autoscale, independent feature.

## Rule anatomy

A **profile** contains one or more rules + scale limits. A rule = condition + action.

```
Profile: "business-hours"
├── Min 2 / Default 3 / Max 20 instances
├── Rule (scale-out): Avg CPU > 75% over 5 min  →  +2 instances, cooldown 5 min
└── Rule (scale-in) : Avg CPU < 25% over 10 min →  -1 instance , cooldown 5 min
```

You can have multiple profiles:

- **Default (metric-based)** — always evaluated.
- **Recurring** — e.g. business-day 08:00–18:00.
- **Fixed-date** — e.g. Black Friday weekend.

Priority: fixed-date → recurring → default.

## Avoid flapping (exam favorite)

Always set the **scale-in threshold strictly lower** than the scale-out threshold, with enough gap that one action doesn't immediately trigger the reverse:

```
Scale-out when CPU > 75 %
Scale-in  when CPU < 25 %   ← NOT 70 % — flapping guaranteed
```

Also tune **cooldowns** (min 1 min, default 5) to let the new instance(s) actually absorb load before the next evaluation.

## Supported signals

- Any **resource metric** (host CPU, memory, HTTP queue length).
- **Custom metric** (App Insights) — popular pattern: scale on queue length in Service Bus or Cosmos RU consumption.
- **Schedule** only (no metric).

A common AZ-204 scenario: *"Scale App Service on Service Bus queue depth"* → use a custom metric scale rule referencing the Service Bus queue's `ActiveMessages` metric.

## Predictive autoscale (VMSS)

For VMSS, **predictive autoscale** learns CPU patterns via ML and provisions ahead of predicted load. Opt-in; great for daily cyclical workloads.

## Bicep example (App Service plan)

```bicep
resource scale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: 'plan-scale'
  location: resourceGroup().location
  properties: {
    enabled: true
    targetResourceUri: plan.id
    profiles: [ {
      name: 'default'
      capacity: { minimum: '2', maximum: '10', default: '2' }
      rules: [
        {
          metricTrigger: {
            metricName: 'CpuPercentage'
            metricResourceUri: plan.id
            operator: 'GreaterThan'
            threshold: 75
            timeAggregation: 'Average'
            timeGrain: 'PT1M'
            timeWindow: 'PT5M'
            statistic: 'Average'
          }
          scaleAction: { direction: 'Increase', type: 'ChangeCount', value: '1', cooldown: 'PT5M' }
        }
        {
          metricTrigger: {
            metricName: 'CpuPercentage'
            metricResourceUri: plan.id
            operator: 'LessThan'
            threshold: 25
            timeAggregation: 'Average'
            timeGrain: 'PT1M'
            timeWindow: 'PT10M'
            statistic: 'Average'
          }
          scaleAction: { direction: 'Decrease', type: 'ChangeCount', value: '1', cooldown: 'PT5M' }
        }
      ]
    } ]
  }
}
```

## Observability of autoscale itself

- Every autoscale action writes to the **Activity Log** → create alert on failed scale actions.
- Enable **diagnostic logs** (`AutoscaleEvaluations`, `AutoscaleScaleActions`) to a workspace for KQL analysis.
- Hook action groups for scale success/failure notifications.

## Exam traps

- **Functions Consumption / Container Apps / AKS do NOT use Monitor Autoscale** — don't pick it as the answer.
- Scale-in threshold must be **strictly and generously** lower than scale-out.
- Minimum evaluation granularity is **1 minute**.
- A profile with *only a schedule* is valid (time-based scale).
- Predictive autoscale is **VMSS-only**.

## Sources

- [Autoscale overview](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview)
- [Best practices](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-best-practices)
- [Predictive autoscale (VMSS)](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-predictive)
- [Scaling App Service plans](https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up)
