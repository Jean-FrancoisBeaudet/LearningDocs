# Throughput & Request Units (RU)

> **Exam mapping:** AZ-204 *(throttling, retries, capacity mode)* · AZ-305 *(cost/scale design)*
> **One-liner:** Everything in Cosmos DB — reads, writes, queries, indexing — is billed in **Request Units per second (RU/s)**. Picking the right capacity mode is the single biggest cost lever.
> **Related:** [Partitioning](02-partitioning.md) · [Indexing & query](06-indexing-and-query.md) · [Cost & performance](10-cost-and-performance.md)

## What an RU is

| Operation | Typical cost |
|-----------|--------------|
| Point read (1 KB doc by id + pk) | **1 RU** |
| Write (1 KB doc, default indexing) | ~**5 RU** |
| Query returning 1 doc via indexed filter | 2–3 RU |
| Query scanning 10 k docs unindexed | hundreds–thousands RU |
| Stored-proc transaction | sum of inner ops + overhead |

`x-ms-request-charge` header is on every response; the SDK exposes `RequestCharge`. **Log it** for every hot query while tuning.

## Capacity modes

| Mode | Bill | Floor | Ceiling | Best fit |
|------|------|-------|---------|----------|
| **Provisioned — Manual** | $ per RU/s per hour (flat) | 400 RU/s (container) · 100 RU/s (DB-shared) | unlimited | Predictable, steady traffic |
| **Provisioned — Autoscale** | $ per **max** RU/s per hour × **1.5** | 10 % of max | unlimited | Variable but sustained traffic |
| **Serverless** | $ per RU **consumed** | 0 | **5 000 RU/s + 1 TB per container** | Dev/test, low or spiky traffic |

### Provisioned: manual vs autoscale

- Manual: cheapest if utilization > ~66 % of max.
- Autoscale: scales between **10 % → max** RU/s per second; billed hourly at the highest RU/s hit that hour × 1.5× the manual rate.
- **Autoscale entry point**: 1 000 RU/s (scales 100 → 1 000).
- **Break-even rule:** if you'd sustain > 66 % of the max all day, manual wins; otherwise autoscale usually wins.

Manual vs autoscale math example — max = 10 000 RU/s:

| Usage profile | Manual cost | Autoscale cost |
|---------------|-------------|----------------|
| Flat 10 000 RU/s | 10 000 × 1× | 10 000 × 1.5× (worse) |
| 3 h @ 10 000, 21 h @ 1 000 | 10 000 × 1× (paying peak) | ≈ avg RU/s × 1.5× (better) |

### Serverless caveats

- Per-container ceiling: **1 TB**, **5 000 RU/s**. Exceed it → throttled, not auto-upgraded.
- No dedicated gateway; some features unavailable (e.g. older multi-region writes). Check the compat matrix before committing.
- **Serverless ↔ Provisioned** in-place migration is GA (both directions).

## Throughput scopes

- **Database-shared** — up to 25 containers share the RU pool. Cheap, but noisy-neighbors. Auto-split between containers by activity, not explicit.
- **Dedicated container** — isolation, predictable perf, required for very large containers.

## Worked RU budget (OLTP API)

Assume 1 KB docs, default indexing:
- 100 writes/s × 5 RU = 500 RU/s
- 400 point-reads/s × 1 RU = 400 RU/s
- 10 queries/s × 10 RU = 100 RU/s
- Overhead / retries buffer (25 %): 250 RU/s
- **Total: ~1 250 RU/s → provision 1 500 manual or 2 000 autoscale max.**

## Throttling (HTTP 429)

Cosmos returns `429 Too Many Requests` with header `x-ms-retry-after-ms` when the partition exceeds its RU budget for the second.

SDK v3 handles this via `CosmosClientOptions.MaxRetryAttemptsOnRateLimitedRequests` (default 9) and `MaxRetryWaitTimeOnRateLimitedRequests` (default 30 s). Tune for write bursts:

```csharp
var client = new CosmosClient(endpoint, credential, new CosmosClientOptions
{
    MaxRetryAttemptsOnRateLimitedRequests = 20,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(60),
    ConnectionMode = ConnectionMode.Direct,
    AllowBulkExecution = true
});
```

For batch loads, prefer **`AllowBulkExecution = true`** — the SDK coalesces items per partition range and nearly saturates provisioned RU/s without over-retrying.

## Bicep — provisioned autoscale container

```bicep
resource container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2024-05-15' = {
  name: '${account.name}/shop/orders'
  properties: {
    resource: {
      id: 'orders'
      partitionKey: { paths: [ '/customerId' ], kind: 'Hash' }
      indexingPolicy: { indexingMode: 'consistent', automatic: true }
    }
    options: {
      autoscaleSettings: { maxThroughput: 4000 }   // scales 400 → 4000
    }
  }
}
```

## Exam traps

- Autoscale **always** scales to 10 % of max, not below.
- Serverless caps are **per container**, not per account.
- RU cost is **per partition**, so 10 k RU/s max ÷ N physical partitions = per-partition ceiling (hot-partition math).
- `429` is normal; SDK retries are automatic; don't add your own Polly retry unless you also disable the SDK's.

## Sources

- [Request Units](https://learn.microsoft.com/en-us/azure/cosmos-db/request-units)
- [Provisioned vs serverless](https://learn.microsoft.com/en-us/azure/cosmos-db/throughput-serverless)
- [Autoscale throughput](https://learn.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autoscale)
- [How to choose manual vs autoscale](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-choose-offer)
- [Serverless → Provisioned migration GA blog](https://devblogs.microsoft.com/cosmosdb/generally-available-seamless-migration-from-serverless-to-provisioned-throughput-in-azure-cosmos-db/)
