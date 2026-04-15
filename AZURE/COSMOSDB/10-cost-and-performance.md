# Cost & Performance Tuning

> **Exam mapping:** AZ-204 *(SDK tuning, retry policy)* · AZ-305 *(cost optimization pillar)*
> **One-liner:** Two levers dominate Cosmos DB cost: **RU/s provisioned** and **indexed write amplification**. Two levers dominate latency: **connection mode** and **partition-aligned queries**.
> **Related:** [Throughput](03-throughput-and-ru.md) · [Indexing](06-indexing-and-query.md) · [Partitioning](02-partitioning.md)

## Cost tuning checklist

### 1. Right-size throughput
- Start **serverless** for dev + low-traffic prod (dev/test, admin APIs, rarely-hit features).
- Flip to **provisioned autoscale** when sustained > 200 RU/s or when serverless caps loom.
- Flip to **manual provisioned** when utilization > 66 % of autoscale max all day (1.5× autoscale premium no longer pays for itself).

### 2. Shrink write RU via indexing policy
- Baseline write ≈ 5 RU/KB — but doubles with a dozen indexed properties on large docs.
- **Exclude** cold properties (large strings, embedded blobs, vectors already vector-indexed).
- Use composite indexes instead of trying to range-index everything.

### 3. Use TTL, don't accumulate
Per-container or per-document `ttl` (seconds):

```jsonc
{ "id": "…", "pk": "…", "ttl": 604800 }   // delete after 7 days
```

TTL deletes are **free** RU-wise (background process). Use for logs, LLM-cache entries, session state.

### 4. Point reads over queries
`ReadItemAsync(id, pk)` = **1 RU**, bypasses the query engine. If you know both values, *never* use a SELECT.

### 5. Bulk-load with `AllowBulkExecution`
For migrations, nightly recomputes:

```csharp
new CosmosClientOptions { AllowBulkExecution = true };
await Task.WhenAll(items.Select(i =>
    container.CreateItemAsync(i, new PartitionKey(i.Pk))));
```

The SDK groups by partition range and nearly saturates provisioned RU/s without over-retrying 429s.

### 6. Reserved Capacity
1- or 3-year commitments on provisioned RU/s → up to **65 % off**. Stack it on stable baseline throughput; use autoscale for the variable top.

### 7. Use the integrated cache (dedicated gateway)
Read-heavy patterns (catalogs, RAG embedding lookups) can serve 60–90 % of reads from cache at a fraction of RU — effectively a cached read tier.

## Latency tuning

### Connection mode

```csharp
new CosmosClientOptions { ConnectionMode = ConnectionMode.Direct }   // TCP, default
```

**Direct/TCP** beats Gateway/HTTPS by ~40 %. If you're in serverless Functions or restricted egress where TCP 10250+ is blocked, fall back to Gateway.

### Preferred regions
Always set `ApplicationPreferredRegions` — without it, SDK defaults to account default region and you pay cross-region latency even when local is available.

### Singleton client
`CosmosClient` opens long-lived connections per backend replica. Instantiating per request = cold connections = slow tails.

```csharp
services.AddSingleton(_ => new CosmosClient(endpoint, credential, options));
```

### Retry policy
Defaults: 9 attempts, 30 s max wait. Raise for bulk workloads, lower for hot user-facing paths where you'd rather fail fast.

### Container warm-up
`await client.CreateAndInitializeAsync(endpoint, credential, containers)` pre-opens connections to all declared containers, eliminating cold-start latency on the first request.

## Monitoring stack

### Metrics (free)
- **Normalized RU Consumption** — % of provisioned RU/s per partition. The single most useful chart.
- **Total Request Units** split by operation type — find query vs write cost.
- **Server-side latency** vs **client-side latency** — gap = network / SDK issue.
- **Throttled Requests (429)** per partition.

### Diagnostic settings → Log Analytics
Emit categories: `DataPlaneRequests`, `QueryRuntimeStatistics`, `PartitionKeyStatistics`, `ControlPlaneRequests`.

```kql
AzureDiagnostics
| where Category == "DataPlaneRequests"
| where TimeGenerated > ago(1h)
| summarize p95 = percentile(duration_s, 95),
            total_ru = sum(requestCharge_s)
          by operationName_s, bin(TimeGenerated, 5m)
```

```kql
// Top RU-spending queries
AzureDiagnostics
| where Category == "QueryRuntimeStatistics"
| summarize total_ru = sum(toreal(requestCharge_s)), count() by querytext_s
| top 20 by total_ru desc
```

### Application Insights
SDK exports client-side latency, retry counts. Correlate via `operation_Id` with server-side Log Analytics entries.

### Cost Management
Cosmos DB appears as "Azure Cosmos DB" with tags per account. Apply **resource tags** (app, env, team) and slice the Cost Management view by tag.

## Common perf anti-patterns

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| 429s with RU utilization < 50 % | Hot partition | [Partitioning note](02-partitioning.md) |
| Writes 10× expected RU | Every property indexed, large doc | Exclude cold paths |
| Slow tail latencies | Gateway mode, short-lived client | Direct mode + singleton |
| Queries charge 100+ RU | Cross-partition + no composite index | Add partition key to `WHERE`, add composite index |
| Serverless throttling | > 5 000 RU/s burst | Migrate to provisioned autoscale |
| Global reads slow | No `ApplicationPreferredRegions` | Set it; close client on replica promotion |

## Exam traps

- TTL deletion is **not** billed in RU/s consumption.
- Point reads use **1 RU** regardless of doc size up to 1 KB (then scale linearly).
- Reserved Capacity is **provisioned only** — serverless doesn't qualify.
- Integrated cache = dedicated gateway; NOT the same as Azure Cache for Redis.
- Normalized RU consumption > 80 % on a single partition = hot partition, not low total throughput.

## Sources

- [Performance tips (SDK v3)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/performance-tips-dotnet-sdk-v3)
- [Troubleshoot performance](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/troubleshoot-query-performance)
- [Monitor Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/monitor)
- [Integrated cache](https://learn.microsoft.com/en-us/azure/cosmos-db/integrated-cache)
- [Reserved capacity](https://learn.microsoft.com/en-us/azure/cosmos-db/reserved-capacity)
- [TTL](https://learn.microsoft.com/en-us/azure/cosmos-db/time-to-live)
