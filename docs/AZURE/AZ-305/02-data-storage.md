# AZ-305 — Design Data Storage Solutions (20–25%)

> Exam alignment: **[AZ-305](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/) → "Design data storage solutions"** (≈24% of exam; [study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305) updated April 17, 2026). Covers relational, semi-structured/unstructured, and data integration design. Related: [AZ-204](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-204/) for developer-facing SDK/config details.

This note is structured as **decision frameworks + concrete trade-offs + Bicep/CLI snippets + Exam traps**. It assumes you already understand basic Azure resource model and RBAC.

---

## 1. Relational data — [Azure SQL DB](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview) vs [SQL MI](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview) vs [PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)/[MySQL Flexible Server](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/overview)

### 1.1 Decision matrix

| Dimension | **[Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)** | **[Azure SQL Managed Instance](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview)** | **[PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) / [MySQL Flexible Server](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/overview)** |
|---|---|---|---|
| Engine | Latest SQL Server (evergreen) | ~100% SQL Server instance surface ([SQL Agent](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/job-automation-managed-instance), cross-DB queries, CLR, [Service Broker](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-service-broker/sql-server-service-broker), linked servers) | Community PostgreSQL / MySQL |
| Lift-and-shift SQL Server | Medium (per-DB) | **High** — closest to on-prem SQL Server | N/A |
| Networking | Public endpoint + [Private Endpoint](https://learn.microsoft.com/en-us/azure/azure-sql/database/private-endpoint-overview); no native VNet injection | **[VNet-injected](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/connectivity-architecture-overview)** (subnet delegated to MI) | VNet-injected ([Flexible Server networking](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-networking)) |
| Max size | [128 TB (Hyperscale)](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale) | 16 TB (Business Critical), 32 TB+ (GP next-gen) | 32 TB ([PG Flexible](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-compute-storage)) |
| HA model | GP: remote storage + failover; BC: [AlwaysOn](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server) local SSD, 3 replicas; Hyperscale: log service + page servers | GP or BC (same models as SQL DB) | [Zone-redundant HA](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-high-availability) (sync replica in another AZ) |
| Read scale-out | HS: up to 4 HA replicas + [30 named replicas](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-replicas); BC: 1 read replica | BC: 1 read replica | [Read replicas](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-read-replicas) (async) |
| Serverless / auto-pause | [Yes](https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview) (single DB, HS serverless GA) | No | No ([burstable tier](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-compute) instead) |
| Elastic Pools | [Yes](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-pool-overview) (GP, BC, HS) | N/A (instance is the pool) | No |
| Cross-DB queries / 3-part names | No ([Elastic Query](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-query-overview) limited) | **Yes** | PG: cross-DB via FDW; MySQL: no |
| Best for | Modern cloud-native apps, multi-tenant SaaS, HTAP | Legacy SQL Server migration, packaged apps needing SQL Agent/linked servers | OSS-friendly workloads, Django/Rails/Node, cost-sensitive |

**Architect rule of thumb (AZ-305):**
- Existing SQL Server estate with instance-level features → **[SQL MI](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview)**.
- New cloud-native app or multi-tenant SaaS → **[Azure SQL DB](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)** (Hyperscale if >4 TB or fast restore needed).
- OSS stack, non-Microsoft ORM, cheaper TCO → **[PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)** (prefer over [deprecated Single Server](https://learn.microsoft.com/en-us/azure/postgresql/migrate/whats-happening-to-postgresql-single-server)).
- Spiky/dev/test workload → **[Serverless](https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview)** SQL DB (auto-pause).

### 1.2 Azure SQL DB — [service tiers](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-sql-database-vcore)

| Tier | Storage layer | Max size | HA | Latency | Pick when |
|---|---|---|---|---|---|
| **[General Purpose (GP)](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-general-purpose)** | Remote premium/standard storage | 4 TB | Stateless compute + storage failover (~30–120 s) | Moderate | Budget, most OLTP |
| **[Business Critical (BC)](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-business-critical)** | Local SSD + 3 AlwaysOn replicas | 4 TB | Sub-second failover; readable secondary | Lowest | Low-latency OLTP, in-memory OLTP, ~2.7× cost of GP |
| **[Hyperscale (HS)](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale)** | Log service + page servers, distributed | **128 TB** (10 GB → 128 TB auto-grow) | 0–4 HA replicas + 30 named replicas; fast restore regardless of size | Low | VLDB, fast restore, read scale-out, serverless at scale |
| **[Serverless (compute tier)](https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview)** | Works with GP or HS | See tier | Auto-pause + per-second billing | Cold-start delay | Intermittent, dev/test |

**Hyperscale specifics** ([FAQ](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-frequently-asked-questions-faq)):
- Backup/restore is **near-constant time** regardless of DB size (snapshot-based on page servers).
- [Reverse migration GP ↔ HS](https://learn.microsoft.com/en-us/azure/azure-sql/database/manage-hyperscale-database) is possible but time-boxed after the move; plan before converting.
- **Fsv2 hardware retires Oct 1, 2026** → migrate to [**premium-series / Standard-series (Gen5)**](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-sql-database-vcore#hardware-configuration).

### 1.3 [Elastic Pools](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-pool-overview)

- Share DTU/vCore budget across many databases with bursty, non-correlated usage.
- Per-DB min/max caps prevent noisy neighbors.
- Great for **SaaS multi-tenant** (database-per-tenant). Not for steady high-utilization DBs — provisioned single DB is cheaper.

### 1.4 [Always Encrypted (AE)](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)

- **Client-side** column-level encryption; keys never leave the client.
- Two modes: **Deterministic** (supports `=`, joins, indexes; vulnerable to frequency analysis) vs **Randomized** (stronger, no server-side predicates).
- **[AE with secure enclaves](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves)** (VBS/Intel SGX on DC-series): allows `LIKE`, range queries, and in-place encryption — closes AE's biggest practical gap.
- Distinguish from **[TDE](https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-tde-overview)** (at-rest on disk, transparent to app) and **[SSE](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)** (storage-level).

### 1.5 Failover groups & auto-failover

- **[Auto-failover group](https://learn.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-sql-db)** = paired read-write + read-only listener DNS across two regions, with automated geo-failover for one or many DBs at once.
- Replication is **async** ([geo-replica](https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview)) → non-zero RPO under stress.
- [SQL MI also supports failover groups](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/auto-failover-group-sql-mi) (one-to-one per instance pair).
- Prefer failover groups over bare [active geo-replication](https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview) for DR, because they give you **stable listener endpoints** and coordinated multi-DB failover.

### 1.6 RTO / RPO cheat sheet

| Capability | RTO | RPO |
|---|---|---|
| LRS zone failure (GP) | Minutes | ~0 (sync) within region |
| [Zone-redundant SQL DB](https://learn.microsoft.com/en-us/azure/azure-sql/database/high-availability-sla) | Seconds–minutes | ~0 |
| BC with AlwaysOn | **Seconds** | ~0 |
| [Active geo-replication](https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview) | Seconds–minutes (manual) | < 5 s typical |
| [Auto-failover group](https://learn.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-sql-db) | Automated within grace period (1h default) | **< 5 s** under normal load |
| [PITR (backup restore)](https://learn.microsoft.com/en-us/azure/azure-sql/database/recovery-using-backups) | Minutes–hours | Up to 10 min (log backups) |
| [LTR restore](https://learn.microsoft.com/en-us/azure/azure-sql/database/long-term-retention-overview) | Hours | Weekly/monthly granularity |

### 1.7 [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview) — [Failover group](https://learn.microsoft.com/en-us/azure/templates/microsoft.sql/servers/failovergroups) (SQL DB)

```bicep
param primaryServerName string
param secondaryServerName string
param dbName string
param fogName string = 'fog-${uniqueString(resourceGroup().id)}'

resource primary 'Microsoft.Sql/servers@2023-08-01-preview' existing = {
  name: primaryServerName
}

resource fog 'Microsoft.Sql/servers/failoverGroups@2023-08-01-preview' = {
  parent: primary
  name: fogName
  properties: {
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60
    }
    readOnlyEndpoint: {
      failoverPolicy: 'Disabled'
    }
    partnerServers: [
      { id: resourceId('Microsoft.Sql/servers', secondaryServerName) }
    ]
    databases: [
      resourceId('Microsoft.Sql/servers/databases', primaryServerName, dbName)
    ]
  }
}
```

---

## 2. [Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/introduction)

### 2.1 API selection

| API | Pick when |
|---|---|
| **[NoSQL (Core)](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/)** | New greenfield; best feature velocity ([vector](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/vector-search), [full-text](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/full-text-search), change feed all-mode) |
| **[MongoDB](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/) (RU or [vCore](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/vcore/introduction))** | Existing Mongo driver/app; vCore tier for Mongo parity + Atlas-like pricing |
| **[Cassandra](https://learn.microsoft.com/en-us/azure/cosmos-db/cassandra/)** | CQL workloads; wide-column |
| **[Gremlin](https://learn.microsoft.com/en-us/azure/cosmos-db/gremlin/)** | Graph workloads |
| **[Table](https://learn.microsoft.com/en-us/azure/cosmos-db/table/)** | Upgrade path from [Azure Table Storage](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-overview) (better SLA, global dist.) |
| **[PostgreSQL (Citus)](https://learn.microsoft.com/en-us/azure/cosmos-db/postgresql/introduction)** | Distributed Postgres / HTAP — note: billed and managed separately from classic Cosmos |

### 2.2 [Partition key](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview) design (most tested concept)

- Choose a key with **high cardinality**, **even access distribution**, and **query affinity**.
- Logical partition limit: **20 GB** + 10k RU/s. Physical partitions scale horizontally; you cannot shrink them.
- **[Hot partition](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/troubleshoot-request-rate-too-large)** = write/read skew → throttles even with high provisioned RU/s.
- **Cross-partition queries** fan-out and cost more RU. Design queries to include partition key in `WHERE`.
- **[Synthetic keys](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/synthetic-partition-keys)**: concatenate tenant + hash suffix (`tenantId_01`..`_NN`) to spread writes when natural key is skewed.
- **[Hierarchical partition keys](https://learn.microsoft.com/en-us/azure/cosmos-db/hierarchical-partition-keys)** (GA on NoSQL): up to 3 levels, useful for multi-tenant (`/tenantId/userId/sessionId`).

### 2.3 Throughput models

| Model | When |
|---|---|
| **[Manual (provisioned) RU/s](https://learn.microsoft.com/en-us/azure/cosmos-db/set-throughput)** | Steady predictable workload; cheapest per RU |
| **[Autoscale](https://learn.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autoscale)** | Variable workload; billed at 1.5× of max used hour; scales 10%→100% of ceiling |
| **[Serverless](https://learn.microsoft.com/en-us/azure/cosmos-db/serverless)** | Dev/test, spiky low volume, small dataset (<1 TB, <5k RU/s peak); per-operation billing |

Rule: autoscale is cheaper than provisioned whenever **avg utilization < 66%** of the manual ceiling.

### 2.4 Five [consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)

Linearizability strength ↓ · Throughput/cost ↑ · Latency ↓

| Level | Guarantee | RU cost (read) | Multi-region write? | Practical use |
|---|---|---|---|---|
| **[Strong](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#strong-consistency)** | Linearizable; reads see latest commit | ~2× | **No** (single write region only) | Financial ledgers needing global linearizability |
| **[Bounded Staleness](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#bounded-staleness-consistency)** | Reads lag by ≤ K versions or T seconds; monotonic, consistent prefix | ~2× | **Yes** (acts like strong in write region, bounded in readers) | Global apps needing predictable staleness window |
| **[Session](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#session-consistency)** (default) | Read-your-writes within a session token | ~1× | Yes | Most user-facing apps — best RU/latency balance |
| **[Consistent Prefix](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#consistent-prefix-consistency)** | Reads never see out-of-order writes | ~1× | Yes | Social feeds, chat (no gaps, but may be stale) |
| **[Eventual](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#eventual-consistency)** | No ordering; convergence only | ~1× | Yes | Analytics, counters, non-critical telemetry |

Stronger-than-account consistency at request time is **not allowed**; you can only **relax** per request.

### 2.5 [Multi-region writes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-multi-master) (multi-master)

- Active-active across all regions; [conflict resolution](https://learn.microsoft.com/en-us/azure/cosmos-db/conflict-resolution-policies) via **Last-Writer-Wins** (LWW on a numeric/timestamp property) or **custom stored-procedure merge**.
- **Cannot** be combined with Strong consistency.
- Premium AZ surcharge **waived** when multi-region writes or autoscale are enabled.

### 2.6 [Change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed)

- Ordered log of inserts/updates per logical partition.
- Two [modes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-modes):
  - **Latest version** (classic): latest state only, no deletes.
  - **[All versions and deletes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-modes#all-versions-and-deletes-change-feed-mode)** (GA on NoSQL): full CDC including intermediate versions and tombstones — for audit, sync, event-sourcing.
- Consumers: [Change Feed processor SDK](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor), [Azure Functions Cosmos trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2-trigger), [Synapse Link](https://learn.microsoft.com/en-us/azure/cosmos-db/synapse-link) (zero-ETL HTAP).

### 2.7 Backups

| Mode | Retention | Restore granularity |
|---|---|---|
| **[Periodic](https://learn.microsoft.com/en-us/azure/cosmos-db/periodic-backup-restore-introduction)** (default) | 8 h – 30 d; 2 free copies | Account-level, to new account |
| **[Continuous 7-day](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)** (free) | 7 d | Point-in-time, to new account |
| **[Continuous 30-day](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)** | 30 d | Point-in-time, DB/container scope |

Continuous mode is required for PITR. Cannot switch continuous → periodic.

### 2.8 RTO / RPO

| Scenario | RTO | RPO |
|---|---|---|
| Single-region + AZ | < minutes | 0 (sync) |
| Multi-region, single write | Automatic failover ≈ 15 min (service-managed) or manual | Typically < 15 s |
| Multi-region writes | Near-zero (apps re-route) | Seconds (depending on replication lag) |
| PITR restore | Minutes–hours | Up to the chosen second (continuous) |

---

## 3. [Azure Storage accounts](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)

### 3.1 [Redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)

| Option | Scope | Copies | Primary region failure | Regional disaster | Durability (annual) |
|---|---|---|---|---|---|
| **[LRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#locally-redundant-storage)** | Single DC | 3 (same rack-spread) | Data lost if DC burns | Lost | 11 nines |
| **[ZRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#zone-redundant-storage)** | 3 AZs in region | 3 (cross-AZ sync) | Survives AZ | Lost | 12 nines |
| **[GRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#geo-redundant-storage)** | LRS + async to paired region | 3 + 3 | Survives | Survives (after failover) | 16 nines |
| **[RA-GRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#read-access-to-data-in-the-secondary-region)** | GRS + secondary read endpoint | 3 + 3 | Survives; read during outage | Survives | 16 nines |
| **[GZRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#geo-zone-redundant-storage)** | ZRS + async to paired region | 3 AZ + 3 | Survives AZ & region | Survives | 16 nines |
| **[RA-GZRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#read-access-to-data-in-the-secondary-region)** | GZRS + secondary read endpoint | 3 AZ + 3 | Survives + read during outage | Survives | 16 nines |

**[Geo-failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance):** customer-initiated **or** Microsoft-initiated. After failover, the account becomes LRS in the new primary; you must re-enable geo-redundancy, which triggers a full resync. **RPO typically < 15 min**, **no SLA** on that number.

[SLA](https://learn.microsoft.com/en-us/azure/reliability/reliability-storage-blob) read availability (GZRS/GRS): **99.9%**; **RA-GZRS/RA-GRS: 99.99%**. Cool/Cold tiers drop one nine.

### 3.2 [Access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) & [lifecycle](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)

| Tier | Min retention | Use |
|---|---|---|
| **[Hot](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#hot-tier)** | None | Frequent access |
| **[Cool](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#cool-tier)** | 30 d | Infrequent (monthly-ish) |
| **[Cold](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#cold-tier)** | 90 d | Rare access, lower storage $, higher read $ |
| **[Archive](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#archive-tier)** | 180 d | [Rehydration](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview) hours; backups, compliance |

[Lifecycle policy](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview) (JSON, applied at account scope) to auto-tier:

```json
{
  "rules": [{
    "name": "age-out",
    "enabled": true,
    "type": "Lifecycle",
    "definition": {
      "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
      "actions": {
        "baseBlob": {
          "tierToCool":    { "daysAfterModificationGreaterThan":  30 },
          "tierToCold":    { "daysAfterModificationGreaterThan":  90 },
          "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
          "delete":        { "daysAfterModificationGreaterThan": 2555 }
        }
      }
    }
  }]
}
```

**Gotcha:** Lifecycle **delete** action is **blocked on immutable containers**. [Early tier-down penalty](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#early-deletion) applies if you re-tier before min-retention.

### 3.3 [Immutable blobs (WORM)](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview)

- **[Time-based retention](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-time-based-retention-policy-overview)** (legal-hold-like) or **[legal hold](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-legal-hold-overview)**.
- **[Container-scope](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-policy-configure-container-scope)** or **[version-scope](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-policy-configure-version-scope)** (requires versioning enabled).
- Policy can be **locked** → cannot reduce retention; meets SEC 17a-4(f), FINRA, CFTC.
- **Not supported when [SFTP](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) or [NFS 3.0](https://learn.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support) protocol is enabled on the account.**

### 3.4 [SFTP](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) & [NFS 3.0](https://learn.microsoft.com/en-us/azure/storage/blobs/network-file-system-protocol-support) on Blob

Both require **[hierarchical namespace (ADLS Gen2)](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)**. Exclusions:
- No account-key-free identity for SFTP users ([local users with SSH keys](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support-how-to)).
- NFS 3.0 requires VNet access (no public NFS).
- **Mutually exclusive** with: soft delete for blobs/containers at certain settings, immutable policies, lifecycle-to-archive, customer-provided keys, and some encryption scopes.

### 3.5 [ADLS Gen2](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)

- **[Hierarchical namespace](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-namespace) (HNS)** on top of Blob → true POSIX-like dirs, atomic rename, [ACLs](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-access-control).
- Native for [Databricks](https://learn.microsoft.com/en-us/azure/databricks/), [Synapse](https://learn.microsoft.com/en-us/azure/synapse-analytics/overview-what-is), [Fabric](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview), [HDInsight](https://learn.microsoft.com/en-us/azure/hdinsight/).
- Cannot be converted back to flat namespace — decision is permanent.
- Use **managed identity + RBAC + ACLs** combo; avoid account keys.

---

## 4. File shares — [Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction) vs [Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-introduction)

[Side-by-side comparison ↗](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-netapp-comparison)

| Feature | **[Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)** | **[Azure NetApp Files (ANF)](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-introduction)** |
|---|---|---|
| Backing | Microsoft-managed storage | Bare-metal NetApp ONTAP in Azure |
| Protocols | [SMB](https://learn.microsoft.com/en-us/azure/storage/files/files-smb-protocol) 2.1/3, [NFS 4.1](https://learn.microsoft.com/en-us/azure/storage/files/files-nfs-protocol) (premium only), REST | NFS 3, NFS 4.1, SMB 3, **[dual-protocol](https://learn.microsoft.com/en-us/azure/azure-netapp-files/create-volumes-dual-protocol)** |
| Tiers | [Standard (HDD), Premium (SSD), provisioned v2](https://learn.microsoft.com/en-us/azure/storage/files/understanding-billing) | [Standard, Premium, Ultra](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-service-levels) (QoS by capacity pool) |
| Latency | Low (premium ~ms) | **Sub-ms**, consistent |
| Snapshots | [Share-level](https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files) | [Volume-level](https://learn.microsoft.com/en-us/azure/azure-netapp-files/snapshots-introduction), writable clones |
| Cross-region replication | Limited (sync via [Azure File Sync](https://learn.microsoft.com/en-us/azure/storage-sync/storage-sync-files-planning)) | Built-in **[CRR](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cross-region-replication-introduction)** |
| AD integration | [Entra ID / AD DS / hybrid](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable) | AD DS (SMB) |
| Typical cost | $ | $$$ (Ultra ≈ $0.39/GB/mo) |
| Use cases | Lift-and-shift SMB, app settings, [FSLogix](https://learn.microsoft.com/en-us/fslogix/overview-what-is-fslogix) basic | SAP HANA, Oracle, HPC, VDI at scale, EDA |

**Architect rule:** Default to Azure Files; escalate to ANF only when you need sub-ms latency, NFS 4.1 at scale, SAP HANA certification, or cross-region volume replication with snapshots.

---

## 5. [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview) — tier selection

| Tier | Scale | HA | Zone redundancy | Persistence | Modules | Pick when |
|---|---|---|---|---|---|---|
| **[Basic](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview#service-tiers)** | 250 MB – 53 GB, single node | **No** | No | No | No | Dev only |
| **Standard** | Same, replicated primary/replica | Yes | Optional | No | No | Small production cache |
| **[Premium](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview#service-tiers)** | Up to 1.2 TB, sharding, [VNet injection](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-how-to-premium-vnet) | Yes | Yes | **[AOF/RDB](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-how-to-premium-persistence)** | No | Enterprise OSS Redis with VNet |
| **[Enterprise](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-enterprise-tiers)** | 1 GB – 2 TB, [active-active geo-replication](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-how-to-active-geo-replication) | Yes | Yes | RDB | **RediSearch, JSON, Bloom, TimeSeries** | Modules, geo-replication, 99.999% |
| **[Enterprise Flash](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-enterprise-tiers)** | 300 GB – 4.5 TB (NVMe-tiered) | Yes | Yes | Yes | Modules | Cost-effective very large caches |

Enterprise/Flash tiers are billed as **infra (Azure) + software IP (Redis Labs via Marketplace)**. They **do not yet support [scale-down/scale-in](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-how-to-scale)**.

VNet injection is Premium-only on OSS tiers; **[Private Endpoint](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-private-link)** is the modern alternative and works on Standard/Premium/Enterprise.

---

## 6. Encryption

| Layer | Mechanism | Key ownership |
|---|---|---|
| **[At-rest default (SSE)](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)** | AES-256, platform-managed keys (PMK) | Microsoft |
| **[SSE + CMK](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)** | Customer-managed key in **[Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview) / [Managed HSM](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/overview)** | You (rotation, revoke) |
| **[Double encryption (infra)](https://learn.microsoft.com/en-us/azure/storage/common/infrastructure-encryption-enable)** | Second layer at infra level (hardware/service) | Microsoft |
| **[Customer-provided keys (CPK)](https://learn.microsoft.com/en-us/azure/storage/blobs/encryption-customer-provided-keys)** | Blob only — key passed per request | You (not stored) |
| **[Always Encrypted](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)** | Client-side column encryption (SQL DB/MI) | You (outside DB) |
| **In-transit** | [TLS 1.2+](https://learn.microsoft.com/en-us/azure/storage/common/transport-layer-security-configure-minimum-version) enforced; 1.3 where supported | — |

### 6.1 Bicep — Storage account with [CMK](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview) + [user-assigned MI](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)

```bicep
param name string
param location string = resourceGroup().location
param kvName string
param keyName string
param uamiId string
param uamiPrincipalId string

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' existing = { name: kvName }

// Grant the MI access via Key Vault RBAC (Key Vault Crypto Service Encryption User)
resource ra 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: kv
  name: guid(kv.id, uamiPrincipalId, 'kv-crypto-enc')
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      'e147488a-f6f5-4113-8e2d-b22465e65bf6') // Key Vault Crypto Service Encryption User
    principalId: uamiPrincipalId
    principalType: 'ServicePrincipal'
  }
}

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: name
  location: location
  sku: { name: 'Standard_GZRS' }
  kind: 'StorageV2'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: { '${uamiId}': {} }
  }
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: false
    publicNetworkAccess: 'Disabled'
    encryption: {
      keySource: 'Microsoft.Keyvault'
      identity: { userAssignedIdentity: uamiId }
      keyvaultproperties: {
        keyvaulturi: kv.properties.vaultUri
        keyname: keyName
      }
      services: {
        blob: { enabled: true, keyType: 'Account' }
        file: { enabled: true, keyType: 'Account' }
      }
      requireInfrastructureEncryption: true  // double encryption
    }
  }
}
```

Key rotation = update the key version (or use [autorotation on the key](https://learn.microsoft.com/en-us/azure/key-vault/keys/how-to-configure-key-rotation)); the storage account picks up new versions automatically when configured without a pinned version.

---

## 7. [Private Endpoints](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) & data exfiltration

- **[Private Endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) (PE)** = NIC in your VNet with a private IP mapped to a sub-resource (`blob`, `sqlServer`, `dfs`, `table`, `vault`, `mongo`, etc.). Traffic stays on Microsoft backbone via [Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview).
- **Disable public access** on data services (`publicNetworkAccess: Disabled`) once PE is wired.
- **[Service Endpoints](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)** are cheaper but grant subnet-wide service access — **not** anti-exfiltration. Prefer PE.
- **Exfiltration hardening stack:**
  1. PE + public access off.
  2. **[Storage firewall](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security)** with **[Resource Instance rules](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security#grant-access-from-azure-resource-instances)** (allow only specific Synapse/Data Factory).
  3. **[Entra ID auth](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory)** only (`allowSharedKeyAccess: false`) → no account key to leak.
  4. **[Azure Firewall with FQDN filtering](https://learn.microsoft.com/en-us/azure/firewall/fqdn-filtering-network-rules)** on egress so compromised VMs can't upload to attacker's storage account.
  5. **Cosmos/SQL: VNet firewall + PE + [disable key auth](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac) (Entra-only where supported)**.
  6. **[Defender for Storage](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-storage-introduction) + [Defender for SQL](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-databases-introduction)** for anomaly detection.
- **Cross-tenant exfil risk (Cosmos/Storage)**: attacker uploads to *their* account. Mitigations = AFW FQDN allow-list + [Private DNS zone](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) locked to your account's PE.

---

## 8. Integration-friendly patterns (quick reference)

- **[Synapse Link](https://learn.microsoft.com/en-us/azure/cosmos-db/synapse-link)**: zero-ETL from Cosmos DB / SQL DB / Dataverse to Synapse analytical store. Near-real-time HTAP without impacting OLTP.
- **[Microsoft Fabric OneLake shortcuts](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts)**: mount ADLS Gen2 into Fabric without copying.
- **Event-driven CDC**: Cosmos change feed → [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2-trigger) → [Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview) / [Event Grid](https://learn.microsoft.com/en-us/azure/event-grid/overview).
- **Data Lake design**: bronze/silver/gold zones as containers on ADLS Gen2 with [directory ACLs](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-access-control) + Fabric/Databricks on top.

---

## 9. Exam traps (memorize these)

- **Strong consistency ≠ compatible with multi-region writes.** If a question mentions multi-master + strong → wrong answer.
- **[Session](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels#session-consistency) is Cosmos default** and the right answer for "user sees their own writes" at minimum cost.
- **[Hyperscale](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale)** = fast restore regardless of DB size and up to 128 TB. BC ≠ big, BC = low-latency.
- **BC has ~2.7× GP cost** and uses local SSD with 3 replicas.
- **[Elastic Pool](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-pool-overview)** is about **many DBs with uncorrelated spikes**, not high-throughput single DB.
- **[SQL MI](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview)** is the answer whenever the question mentions **SQL Agent, linked servers, cross-DB queries, Service Broker, CLR, or lift-and-shift of on-prem SQL Server**.
- **[Failover groups](https://learn.microsoft.com/en-us/azure/azure-sql/database/auto-failover-group-sql-db)** > [active geo-replication](https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview) for architect-level DR answers (stable listener, grouped failover).
- **LRS ≠ ZRS.** LRS is single datacenter; if the question says "survive a datacenter/AZ outage" → [ZRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#zone-redundant-storage) (or higher).
- **GZRS ≠ GRS.** [GZRS](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy#geo-zone-redundant-storage) uses ZRS in primary; pick it when both AZ and region resilience are required.
- **RA-\*** variants only differ by **read from secondary** during outage. If the question says "read during failover" → RA-GZRS/RA-GRS.
- **[Archive tier](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview#archive-tier)** has **180-day min retention**, **hours rehydration**, **per-GB read cost**. Not for anything latency-sensitive.
- **Immutable blob + SFTP/NFS** = **not supported together**. Same for lifecycle *delete* in an immutable container.
- **[ADLS Gen2](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction) is one-way:** you cannot convert HNS off.
- **[Always Encrypted](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine)** is **client-side**; [TDE](https://learn.microsoft.com/en-us/azure/azure-sql/database/transparent-data-encryption-tde-overview) is at-rest server-side. Exam loves conflating them.
- **[Secure enclaves](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-enclaves)** enable rich queries (range, LIKE) on AE-encrypted columns.
- **[Redis Enterprise tiers](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-enterprise-tiers)** are needed for **modules (RediSearch/JSON)** and **active-active geo**. Premium is VNet-injection for OSS tier.
- **[Service Endpoint](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview) ≠ [Private Endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview).** Exam often uses SE as a distractor when the real answer is PE + disable public access.
- **[Partition key](https://learn.microsoft.com/en-us/azure/cosmos-db/partitioning-overview)** is immutable after container creation in Cosmos (NoSQL) — if the question implies "need to change key," the answer is **copy to new container**.
- **[Autoscale RU](https://learn.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autoscale)** bills at 1.5× of max used hour — worth it only when utilization < ~66%.
- **[PostgreSQL Single Server is retired/legacy](https://learn.microsoft.com/en-us/azure/postgresql/migrate/whats-happening-to-postgresql-single-server)** — choose **[Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)**.
- **Backup modes on Cosmos** — PITR requires **[continuous](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)**, and you **can't switch back** from continuous to [periodic](https://learn.microsoft.com/en-us/azure/cosmos-db/periodic-backup-restore-introduction).

---

## Sources

- [Microsoft Learn — AZ-305 exam page](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/)
- [AZ-305 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305)
- [Exam Readiness Zone — AZ-305 Design data storage (Part 2)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-305-02-fy25)
- [Azure SQL Database Hyperscale overview](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale)
- [Azure SQL Database Hyperscale FAQ](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tier-hyperscale-frequently-asked-questions-faq)
- [vCore purchasing model — service tiers](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-sql-database-vcore)
- [Single database vCore resource limits](https://learn.microsoft.com/en-us/azure/azure-sql/database/resource-limits-vcore-single-databases)
- [Azure Database for PostgreSQL — Flexible Server overview](https://learn.microsoft.com/en-us/azure/postgresql/overview)
- [Cosmos DB consistency level choices](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [Cosmos DB autoscale FAQ](https://learn.microsoft.com/en-us/azure/cosmos-db/autoscale-faq)
- [Cosmos DB for NoSQL reliability](https://learn.microsoft.com/en-us/azure/reliability/reliability-cosmos-db-nosql)
- [Azure Storage data redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Azure Blob Storage reliability](https://learn.microsoft.com/en-us/azure/reliability/reliability-storage-blob)
- [Azure Blob lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Immutable storage for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview)
- [Azure Files vs Azure NetApp Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-netapp-comparison)
- [Azure Cache for Redis — Enterprise tier best practices](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices-enterprise-tiers)
- [Azure Cache for Redis overview](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview)
