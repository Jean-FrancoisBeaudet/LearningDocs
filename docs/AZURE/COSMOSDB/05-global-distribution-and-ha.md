# Global Distribution & High Availability

> **Exam mapping:** AZ-305 *(multi-region design, DR, SLA composition)* · AZ-204 *(multi-region read/write SDK behavior)*
> **One-liner:** Turn-key global replication — add a region and the SDK routes to the nearest one; upgrade to multi-region writes for active/active.
> **Related:** [Consistency](04-consistency-levels.md) · [Overview / SLA](00-overview.md)

## Topology options

| Topology | Writes | Reads | Multi-region write conflicts? | Read SLA | Use when |
|----------|--------|-------|------------------------------|----------|----------|
| Single-region | 1 region | Same region only | n/a | 99.99% | Dev, regional apps |
| Multi-region, single-write | 1 "hub" region | All regions (nearest-read) | n/a | **99.999%** | Global reads, writes OK in one DC |
| **Multi-region, multi-write** | All regions | All regions | Yes — resolution policy required | **99.999% write** too | Truly global apps, active/active |

Add/remove regions online — no downtime.

## SLA composition

| Config | Read | Write |
|--------|-----:|------:|
| Single-region | 99.99 % | 99.99 % |
| Multi-region, single-write | **99.999 %** | 99.99 % |
| Multi-region, multi-write | 99.999 % | **99.999 %** |

SLA only applies when the SDK is in **direct mode** and the region list is correctly configured (`PreferredLocations`).

## Conflict resolution (multi-region writes)

Three policies configurable per container:

1. **Last-Writer-Wins (default)** — highest `_ts` wins; tie-broken by `_etag`.
2. **Custom property LWW** — use a property you control (e.g. `version` integer).
3. **Custom merge via stored procedure** — you write JS that reads all conflicting versions from the **Conflicts feed** and emits the survivor.

Bicep snippet:

```bicep
conflictResolutionPolicy: {
  mode: 'LastWriterWins'
  conflictResolutionPath: '/version'   // integer property
}
```

Unresolved conflicts land in a per-container **conflict feed** you can drain.

## Failover: automatic vs manual

- **Service-managed failover** (on by default when enabled): Cosmos promotes a read region to write on regional outage.
- **Manual failover**: you trigger via portal/CLI for planned DR drills.
- RTO: single-digit minutes. RPO: < 15 min (single-region write) / ~0 (multi-region write).

## Zone redundancy

- Optional per-region: replicas across 3 AZs within the region.
- Adds cost (~1.25×), raises in-region availability to 99.995 %.
- **Required** for 99.999% SLA in some configs — check current docs before committing.

## SDK region preferences

```csharp
var options = new CosmosClientOptions
{
    ApplicationPreferredRegions = new List<string>
    {
        "West Europe",      // primary for this tenant
        "North Europe",     // failover
        "East US"
    },
    ConnectionMode = ConnectionMode.Direct
};
```

Without preferences, SDK uses the account's default region — latency-critical workloads should always set this.

## Backups

| Policy | Retention | PITR granularity | Restore target |
|--------|-----------|-------------------|----------------|
| **Periodic** (default) | 2 copies, 4 h interval (configurable up to 30 days) | full restore only | new account |
| **Continuous 7-day** | Rolling 7 days | second-level | new account, any point |
| **Continuous 30-day** | Rolling 30 days | second-level | new account |

Switching periodic → continuous is one-way. Restore is always to a **new account** — you cannot overwrite in place.

## Disaster-recovery playbook

1. Multi-region writes + zone redundancy in the primary region = baseline.
2. Session consistency in most user-facing apps; Bounded Staleness if ordering matters globally.
3. Use continuous-7-day backup for anything with PITR requirements (GDPR, financial).
4. Periodic **DR drill**: trigger a manual failover in non-prod; measure time to promote and app-tier reconnection.

## Exam traps

- *"Strong consistency with multi-region writes"* — unsupported.
- *"Backup restore overwrites the account"* — never; always new account.
- 99.999 % SLA requires **multi-region + preferred-regions + direct mode**.
- Adding a region rebuilds the read index there — expect minutes-to-hours catch-up for large containers.

## Sources

- [Distribute your data globally](https://learn.microsoft.com/en-us/azure/cosmos-db/distribute-data-globally)
- [High availability](https://learn.microsoft.com/en-us/azure/cosmos-db/high-availability)
- [Conflict resolution](https://learn.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies)
- [Continuous backup / PITR](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)
- [SLA details](https://learn.microsoft.com/en-us/azure/cosmos-db/high-availability#slas)
