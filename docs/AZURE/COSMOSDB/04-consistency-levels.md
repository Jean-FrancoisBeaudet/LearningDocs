# Consistency Levels

> **Exam mapping:** AZ-204 · AZ-305 *(heavily tested — know the 5 and their trade-offs by heart)*
> **One-liner:** Cosmos DB offers **five** consistency levels on a spectrum; you trade availability, latency, and RU cost for stricter ordering/freshness guarantees.
> **Related:** [Global distribution & HA](05-global-distribution-and-ha.md) · [Throughput](03-throughput-and-ru.md)

## The spectrum

From strongest to weakest:

| Level | What you see | Read RU cost | Multi-region read SLA | Typical use |
|-------|--------------|-------------:|-----------------------|-------------|
| **Strong** | Linearizable. Every read returns the latest committed write. | 2× | 99.99% | Financial ledger, inventory with strict invariants |
| **Bounded Staleness** | Reads lag writes by at most **K versions** *or* **T time** (configurable). Ordered. | 2× | 99.999% | Status dashboards tolerating N-second lag |
| **Session** *(default)* | Read-your-writes, monotonic reads, within one session (token). Across sessions = eventual. | 1× | 99.999% | Most user-facing apps |
| **Consistent Prefix** | No gaps in update order; may be stale. | 1× | 99.999% | Social timelines |
| **Eventual** | No ordering, no freshness guarantee. | 1× | 99.999% | Counters, telemetry where order is irrelevant |

Set the **default** at the account level; **clients can request weaker** on a per-request basis, never stronger than the account default.

## When each one actually matters

### Strong
- Hard invariant needs (bank account never negative, unique reservation).
- **Requires**: single-region write or "Strong-capable" regions (you pick a write region set). Cannot span all regions globally.
- Write latency hit: every write waits for majority commit across regions → visible > 10 ms multi-region.

### Bounded Staleness
- You're OK with "≤ 5 s OR ≤ 100 updates stale". Preserves global order.
- Best choice when Strong is too slow but you need ordering.
- Config: `maxStalenessPrefix` (versions), `maxIntervalInSeconds`.

### Session *(default — and almost always the right answer)*
- Client carries a **session token** (`x-ms-session-token`). Within that token scope you get read-your-writes and monotonic reads.
- Crossing sessions (e.g. browser → background worker reading the same doc) → treat as eventual unless you forward the session token.
- Best read SLA + lowest RU cost.

### Consistent Prefix
- Rare in practice. Think "followers see tweets in the same order" without caring about freshness.

### Eventual
- Highest availability, lowest RU. Use for write-heavy telemetry, counters, analytics ingestion where the client re-reads later or doesn't read at all.

## Multi-region writes + consistency

- **Single-region write** + any level: straightforward.
- **Multi-region write** + Strong: **unsupported** — topology rejects it.
- **Multi-region write** + Bounded Staleness: supported with caveats; conflicts resolved by the configured policy (LWW on `_ts` by default, or custom stored proc).
- **Multi-region write** + Session/Prefix/Eventual: default, fast.

See [`05-global-distribution-and-ha.md`](05-global-distribution-and-ha.md) for conflict-resolution details.

## Setting consistency in code

Account-level default is set in the portal / Bicep. Per-request override in SDK v3:

```csharp
var options = new ItemRequestOptions
{
    ConsistencyLevel = ConsistencyLevel.Eventual   // weaker than account default → OK
};
var resp = await container.ReadItemAsync<Order>(id, new PartitionKey(customerId), options);
```

Requesting **stronger** than the account default throws.

## Session-token propagation pattern

Distributed system with a web tier + worker tier, both reading Cosmos:

```csharp
// Web tier after write:
string token = writeResp.Headers.Session;
bus.Publish(new OrderPlaced { OrderId = id, CosmosSession = token });

// Worker:
var readOpts = new ItemRequestOptions { SessionToken = msg.CosmosSession };
var order = await container.ReadItemAsync<Order>(msg.OrderId, pk, readOpts);
```

Without propagation, the worker reads may lag the web-tier write.

## Exam traps

- *"Strong is the default"* — no, **Session** is.
- *"Strong across all regions"* — not in multi-region-write topology.
- *"Eventual guarantees ordering"* — no; even prefix ordering needs **Consistent Prefix**.
- *"Consistency is per-query only"* — default is account-wide; per-request can only **weaken** it.
- RU cost doubles for Strong and Bounded Staleness on reads.

## Sources

- [Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [Choose the right consistency](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels-choosing)
- [Trade-offs per level](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels-tradeoffs)
