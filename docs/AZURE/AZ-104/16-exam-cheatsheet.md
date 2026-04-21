# AZ-104 Exam Cheatsheet

> **Purpose:** Last-minute review sheet -- scan this before walking into the exam.
> **Full notes:** [00-overview.md](00-overview.md) and individual deep-dives 01-15.
> **Passing score:** 700 / 1000 | **Duration:** 100 minutes | ~40-60 items

---

## 1. Identity & Governance (20-25%)

### Entra ID essentials

- **User types:** Member (full access) vs Guest (limited). Cloud-only vs synced (Entra Connect Sync / Cloud Sync).
- **Group types:**

| Type | Membership | Use |
|---|---|---|
| **Security** (assigned) | Manual add | RBAC, resource access |
| **Security** (dynamic) | Rule-based (`user.department -eq "IT"`) | Auto-membership; **requires P1+** |
| **Microsoft 365** (assigned/dynamic) | Manual or rule-based | Shared mailbox, Teams, SharePoint |

- **SSPR:** requires at least **1 auth method** (Security Q not valid for admins). Can scope to a group. Reset requires P1 for writeback to on-prem.
- **License assignment:** Direct (per user) vs Group-based (auto-assign by group membership; **requires P1+**).
- **Bulk operations:** CSV upload for create/invite/delete users.

### RBAC

| Built-in Role | Can manage resources? | Can grant access? |
|---|---|---|
| **Owner** | Yes | Yes |
| **Contributor** | Yes | No |
| **Reader** | Read-only | No |
| **User Access Administrator** | No | Yes |

**Top service-specific roles:**

| Role | Scope |
|---|---|
| Storage Blob Data Contributor/Reader/Owner | Blob data plane |
| Virtual Machine Contributor | VM management (not access) |
| Network Contributor | Network resources |
| Key Vault Secrets User/Officer | Key Vault data plane |
| Monitoring Contributor/Reader | Azure Monitor |

- **Scope hierarchy:** Management Group > Subscription > Resource Group > Resource (inherited downward, **cannot break inheritance**).
- **Azure RBAC vs Entra roles:** RBAC = Azure resource access; Entra roles = directory admin (Global Admin, User Admin). Global Admin does NOT auto-grant Azure RBAC -- must elevate at root scope.
- **Limits:** 4,000 role assignments per subscription; 5,000 custom roles per tenant.

### Governance

**Policy effects evaluation order:**

```
Disabled --> Append/Modify --> Deny --> Audit --> DeployIfNotExists / AuditIfNotExists
```

- `DeployIfNotExists` (DINE) requires a **managed identity** on the policy assignment for remediation.
- **Exemptions:** Waiver (temporary) vs Mitigated (compensating control).

**Resource locks:**

| Lock | Delete? | Modify? | Data-plane writes? |
|---|---|---|---|
| **CanNotDelete** | Blocked | Allowed | Allowed |
| **ReadOnly** | Blocked | Blocked | Allowed (locks are management-plane only) |

- Locks **inherit** down the scope hierarchy.

**Tags:** max **50** tag name-value pairs per resource. Tags do **not** inherit by default -- use Azure Policy (`Inherit a tag from the resource group`) to enforce.

**Management groups:** up to **6 levels** of depth (excluding root and subscription level). One subscription = one parent only.

**Cost management:** Budgets + alerts (actual & forecasted) + Azure Advisor cost recommendations + spending limits (dev/test only).

---

## 2. Storage (15-20%)

### Storage account types & redundancy

| Account type | Services | Performance |
|---|---|---|
| **Standard general-purpose v2** | Blob, File, Queue, Table | Standard (HDD) |
| **Premium block blobs** | Block/Append blobs | Premium (SSD) |
| **Premium file shares** | Azure Files | Premium (SSD) |
| **Premium page blobs** | Page blobs | Premium (SSD) |

- **Naming:** 3-24 chars, lowercase + numbers only, globally unique.
- **Performance tier cannot be changed** after creation.
- Premium supports only **LRS and ZRS** (no geo-redundancy).

| Redundancy | Copies | Regions | Durability | Read from secondary? |
|---|---|---|---|---|
| **LRS** | 3 | 1 (single DC) | 11 nines | No |
| **ZRS** | 3 | 1 (3 AZs) | 12 nines | No |
| **GRS** | 6 | 2 | 16 nines | No |
| **RA-GRS** | 6 | 2 | 16 nines | **Yes** |
| **GZRS** | 6 | 2 (primary ZRS) | 16 nines | No |
| **RA-GZRS** | 6 | 2 (primary ZRS) | 16 nines | **Yes** |

### SAS types (security order)

1. **User delegation SAS** -- signed with Entra credentials (most secure, **exam favourite**)
2. **Service SAS** -- signed with storage account key, scoped to one service
3. **Account SAS** -- signed with storage account key, scoped to one or more services

- **Stored access policies:** only on **service SAS** (not account SAS). Max **5** per container/queue/table/share.

### Access tiers

| Tier | Min retention | Availability SLA | Online? |
|---|---|---|---|
| **Hot** | None | 99.9% (RA: 99.99%) | Yes |
| **Cool** | **30 days** | 99% (RA: 99.9%) | Yes |
| **Cold** | **90 days** | 99% (RA: 99.9%) | Yes |
| **Archive** | **180 days** | N/A | **No** (offline) |

- **Rehydration:** Standard (up to 15 hours), High priority (< 1 hour for objects < 10 GB).
- Default account tier is Hot or Cool. Cold/Archive set **per blob**.

### Data protection

- **Soft delete:** independently configured for blobs, containers, and file shares (1-365 days).
- **Versioning:** automatic previous version on overwrite (Blob only).
- **Lifecycle management:** tiering and deletion rules. Actions: `tierToCool`, `tierToCold`, `tierToArchive`, `delete`. Applied once per day.

### Azure Files

- **Protocols:** SMB (2.1/3.0/3.1.1) and NFS 4.1 (Premium only, Linux only).
- **Tiers:** Transaction Optimized, Hot, Cool (Standard); Premium (SSD, provisioned IOPS).
- **Identity-based access:** on-prem AD DS, Entra Domain Services, Entra Kerberos (hybrid users).
- **Share snapshots:** read-only, incremental, max 200 per share.

---

## 3. Compute (20-25%)

### ARM vs Bicep

| Feature | ARM JSON | Bicep |
|---|---|---|
| Syntax | Verbose JSON | Concise DSL |
| Modules | Linked/nested templates | `module` keyword |
| Dependencies | Explicit `dependsOn` | **Auto-detected** (implicit via `reference()`) |
| Parameter files | `.parameters.json` | `.bicepparam` |
| Decompile | N/A | `az bicep decompile` |

**Deployment modes:**

| Mode | Existing resources NOT in template | Use case |
|---|---|---|
| **Incremental** (default) | Left unchanged | Normal deployments |
| **Complete** | **Deleted** | Full-state enforcement (dangerous) |

- `what-if` flag previews changes before deploying.

### VM size families (quick reference)

| Series | Optimized for | Example |
|---|---|---|
| **B** | Burstable, dev/test | B2s |
| **D** | General purpose | D4s_v5 |
| **E** | Memory | E8s_v5 |
| **F** | Compute | F4s_v2 |
| **N** | GPU | NC6s_v3 |
| **L** | Storage (high IOPS) | L8s_v3 |
| **M** | Memory-intensive (SAP) | M128s |

### Disk types

| Disk | Max IOPS | Max size | Use case |
|---|---|---|---|
| **Standard HDD** | 500 | 32 TiB | Backup, dev/test |
| **Standard SSD** | 6,000 | 32 TiB | Web servers, light workloads |
| **Premium SSD** | 20,000 | 32 TiB | Production workloads |
| **Premium SSD v2** | 80,000 | 64 TiB | Performance-sensitive |
| **Ultra** | 400,000 | 64 TiB | SAP HANA, databases |

### Encryption types

| Type | Encrypts | Method |
|---|---|---|
| **SSE** (Server-Side Encryption) | Data at rest on disk | Automatic, always on (Microsoft-managed or customer-managed keys) |
| **Encryption at Host** | Temp disk + OS/data disk caches | Enabled on VM; data encrypted before leaving host |
| **ADE** (Azure Disk Encryption) | OS + data volumes inside VM | BitLocker (Windows) / DM-Crypt (Linux) |

- SSE and Encryption at Host are **not** ADE. Exam loves this distinction.

### Availability

| Strategy | SLA | What it protects against |
|---|---|---|
| **Availability Zone** | **99.99%** | Datacenter failure (VMs across 3 zones) |
| **Availability Set** | **99.95%** | Rack/update failure (fault domains + update domains) |
| **Single VM (Premium SSD)** | **99.9%** | N/A |

- Cannot combine AZ + AS for the same VM.
- **VMSS Uniform:** identical VMs, legacy mode. **VMSS Flexible:** mixed sizes, zone-spread, recommended for new deployments.

### Containers

**ACR SKUs:**

| SKU | Storage | Geo-replication | Private endpoint |
|---|---|---|---|
| **Basic** | 10 GiB | No | No |
| **Standard** | 100 GiB | No | No |
| **Premium** | 500 GiB | **Yes** | **Yes** |

**ACI vs ACA:**

| Feature | ACI | ACA |
|---|---|---|
| Use case | Simple, single-job containers | Microservices, APIs |
| Scaling | Manual | **Auto** (KEDA-based) |
| Dapr | No | **Yes** |
| Ingress / revisions | No | **Yes** |
| Billing | Per-second (CPU+RAM) | Per-second (CPU+RAM) |

### App Service tiers

| Tier | Slots | Autoscale | VNet integration | Backup | Custom domain |
|---|---|---|---|---|---|
| **Free / Shared** | 0 | No | No | No | Shared only |
| **Basic** | 0 | No | No | No | Yes (SSL) |
| **Standard** | **5** | **Yes** | **Yes** | **Yes** | Yes |
| **Premium** | **20** | Yes | Yes | Yes (50/day) | Yes |
| **Isolated** | 20 | Yes | ASE (full isolation) | Yes | Yes |

- Slot swap = DNS swap (no cold start if warmed). **Sticky settings** do NOT swap.
- **Always On** requires Basic+. **Autoscale** requires Standard+.

---

## 4. Networking (15-20%)

### VNet fundamentals

- **5 reserved IPs** per subnet: .0 (network), .1 (gateway), .2-.3 (DNS), last (broadcast).
- Smallest subnet: **/29** (3 usable).
- **Peering is non-transitive:** A<->B and B<->C does NOT give A<->C. Use hub-spoke + gateway/NVA.
- **Gateway transit:** hub VNet shares its VPN/ER gateway with spokes.

**Public IP SKU:**

| | Basic | Standard |
|---|---|---|
| Allocation | Static or Dynamic | **Static only** |
| Security | Open by default | **Secure by default** (needs NSG) |
| AZ support | No | **Yes** (zone-redundant) |
| LB compatibility | Basic LB | **Standard LB** |

### NSG processing

- Rules evaluated by **priority** (lowest number = highest priority). First match wins.
- Applied at **NIC and/or subnet** level -- both are evaluated (inbound: subnet NSG first, then NIC NSG).
- **Default rules** (priority 65000-65500): AllowVNetInBound, AllowAzureLoadBalancerInBound, DenyAllInBound (similar for outbound). Cannot be deleted.
- **ASG** constraints: all NICs in an ASG must be in the **same VNet**. NSG rules with ASGs as source/destination must have all ASGs in NICs on the same VNet.

### Azure Bastion

| SKU | Features |
|---|---|
| **Developer** | Free, single connection, no subnet needed |
| **Basic** | Multi-session, native client, requires **AzureBastionSubnet** (**/26** min) |
| **Standard** | + file transfer, IP-based connect, shareable link, scaling |
| **Premium** | + session recording, private-only |

### Service endpoint vs Private endpoint

| | Service Endpoint | Private Endpoint |
|---|---|---|
| Traffic path | Microsoft backbone | **Private IP in your VNet** |
| PaaS public IP | Still exists | Effectively disabled |
| DNS | No change | Requires private DNS zone |
| Cross-region | No (same region) | **Yes** |
| Cost | Free | Per-hour + per-GB |

### Azure DNS

- **Alias record:** points to an Azure resource (LB, Traffic Manager, CDN, public IP). Auto-updates on IP change. Supports zone apex (@).
- **CNAME:** cannot be at zone apex. Points to another DNS name.
- **Private DNS zones:** require a **virtual network link**. **Auto-registration** creates A records for VMs automatically (limit: **one** private zone with auto-registration per VNet).

### Load Balancer

| | Basic | Standard |
|---|---|---|
| Cost | Free | Paid |
| Backend pool | Availability set or single VMSS | Any VMs/VMSS in single VNet |
| AZ support | No | **Yes** |
| HA ports | No | **Yes** (internal LB only) |
| Outbound rules | No | **Yes** |
| SLA | None | **99.99%** |

- **Health probe source:** `168.63.129.16` (must be allowed in NSG).
- **HA ports:** internal Standard LB only; load-balances **all protocols on all ports**.

---

## 5. Monitor & Maintain (10-15%)

### Metrics vs Logs

| | Metrics | Logs |
|---|---|---|
| Type | Numeric time-series | Text / structured records |
| Latency | Near-real-time | Minutes |
| Retention | **93 days** default | **30 days** default (up to **730 days** interactive, **12 years** archive) |
| Query | Metrics Explorer | **KQL** in Log Analytics |

- **Diagnostic settings:** route platform logs/metrics to Log Analytics, Storage, or Event Hub.
- **Data Collection Rules (DCR)** replace legacy agents -- the exam tests DCR-based collection.

### Alert types

| Type | Fires on | Evaluation frequency |
|---|---|---|
| **Metric alert** | Threshold on metric values | 1-15 min |
| **Log search alert** | KQL query result | 5-15 min |
| **Activity log alert** | Control-plane events (e.g., VM deallocated) | Near-real-time |

- **Severity:** 0 (Critical) through 4 (Verbose).
- **Action groups:** email, SMS, webhook, Logic App, Azure Function, ITSM.
- **Alert processing rules** != alert rules. They suppress/route notifications **after** an alert fires (e.g., maintenance windows).

### KQL essentials

```kusto
// Filter
AzureMetrics | where TimeGenerated > ago(1h)

// Aggregate
Perf | summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer

// Join
Heartbeat | join kind=inner (Perf) on Computer
```

Key functions: `where`, `summarize`, `ago()`, `bin()`, `project`, `extend`, `count()`, `distinct`.

### Insights & Network Watcher

- **VM Insights:** performance + map (dependencies). Requires DCR + Azure Monitor Agent.
- **Storage Insights:** availability, latency, capacity dashboards.
- **Network Insights:** topology view.
- **Network Watcher tools:**
  - **IP flow verify** -- test if a packet is allowed/denied by NSG.
  - **Next hop** -- determine next hop for a packet.
  - **Connection troubleshoot** -- end-to-end connectivity test.
  - **NSG diagnostics** -- evaluate effective NSG rules.
  - **Packet capture** -- capture traffic on a VM NIC.
  - **Connection monitor** -- ongoing monitoring of connectivity.

### Backup & Recovery

| Vault type | Workloads |
|---|---|
| **Recovery Services vault** | Azure VMs, SQL in VM, Azure Files, SAP HANA |
| **Backup vault** | Azure Blobs, Azure Disks, Azure Database for PostgreSQL |

- **Soft delete:** enabled by default on Recovery Services vaults. Deleted data retained **14 extra days**.
- **Cross-region restore:** available with GRS vaults.
- **GFS retention:** daily + weekly + monthly + yearly schedules.

**Azure Site Recovery (ASR) failover workflow:**

```
Enable replication --> Continuous sync --> Test failover (isolated, no impact)
--> Planned/Unplanned failover (stops replication) --> Reprotect --> Failback
```

- Test failover does NOT affect source VM or replication.
- Real failover STOPS replication until you reprotect.

---

## Key Numbers to Memorize

| Number | What it means |
|---|---|
| **5** | Reserved IPs per subnet |
| **5** | Max stored access policies per container/share |
| **6** | Max management group depth (excl. root + subscription) |
| **14 days** | Backup soft-delete retention |
| **30 days** | Cool tier min retention |
| **50** | Max tags per resource |
| **90 days** | Cold tier min retention |
| **93 days** | Default metric retention |
| **180 days** | Archive tier min retention |
| **200** | Max file share snapshots |
| **730 days** | Max interactive Log Analytics retention |
| **4,000** | Max RBAC role assignments per subscription |
| **5,000** | Max custom roles per Entra tenant |
| **99.9%** | Single VM SLA (Premium SSD) |
| **99.95%** | Availability Set SLA |
| **99.99%** | Availability Zone SLA / Standard LB SLA |
| **/26** | Minimum AzureBastionSubnet size |
| **/29** | Smallest Azure subnet (3 usable IPs) |
| **168.63.129.16** | Azure health probe / DHCP / DNS source IP |
| **3-24 chars** | Storage account name length |
| **1-64 chars** | VM name length |

---

## Common "vs" Comparisons

| Comparison | Key differentiator |
|---|---|
| **Owner vs Contributor** | Owner can assign roles; Contributor cannot |
| **Owner vs User Access Administrator** | Owner manages resources + access; UAA manages access only |
| **CanNotDelete vs ReadOnly lock** | ReadOnly blocks modifications too (management plane only) |
| **Azure RBAC vs Entra roles** | RBAC = resource access; Entra = directory admin |
| **LRS vs ZRS** | LRS = single DC; ZRS = 3 AZs in one region |
| **GRS vs RA-GRS** | RA-GRS allows read from secondary region |
| **User delegation SAS vs Service SAS** | UD SAS uses Entra creds (more secure); Service SAS uses account key |
| **Service endpoint vs Private endpoint** | SE = backbone route, still public IP; PE = private IP in VNet |
| **Availability Zone vs Availability Set** | AZ = cross-datacenter (99.99%); AS = cross-rack (99.95%) |
| **VMSS Uniform vs Flexible** | Flexible = mixed sizes, zone-spread, recommended |
| **ACI vs ACA** | ACI = simple jobs; ACA = microservices with autoscale + Dapr |
| **Complete vs Incremental deploy** | Complete deletes resources not in template |
| **SSE vs ADE vs Encryption at Host** | SSE = at-rest auto; ADE = in-guest BitLocker; EaH = temp/cache on host |
| **Metrics vs Logs** | Metrics = numeric, 93d; Logs = structured text, KQL |
| **Recovery Services vault vs Backup vault** | RSV = VMs/SQL/Files; BV = Blobs/Disks/PostgreSQL |
| **Alert rules vs Alert processing rules** | Alert rules fire alerts; processing rules suppress/route after firing |
| **Basic vs Standard public IP** | Basic = open by default, no zones; Standard = secure, zones |
| **Basic vs Standard LB** | Basic = free, no zones, no SLA; Standard = paid, zones, 99.99% |
| **NSG at subnet vs NIC** | Inbound: subnet NSG evaluated first, then NIC NSG |
| **Alias record vs CNAME** | Alias works at zone apex, auto-updates; CNAME cannot be at apex |
| **Test failover vs Planned failover (ASR)** | Test = isolated, no impact; Planned = stops replication |

---

## Sources

All detailed coverage lives in the individual deep-dive notes:

- [01-entra-users-and-groups.md](01-entra-users-and-groups.md) | [02-access-to-azure-resources.md](02-access-to-azure-resources.md) | [03-subscriptions-and-governance.md](03-subscriptions-and-governance.md)
- [04-storage-access-and-security.md](04-storage-access-and-security.md) | [05-storage-accounts.md](05-storage-accounts.md) | [06-azure-files-and-blob.md](06-azure-files-and-blob.md)
- [07-arm-and-bicep.md](07-arm-and-bicep.md) | [08-virtual-machines.md](08-virtual-machines.md) | [09-containers.md](09-containers.md) | [10-app-service.md](10-app-service.md)
- [11-virtual-networks.md](11-virtual-networks.md) | [12-network-security.md](12-network-security.md) | [13-dns-and-load-balancing.md](13-dns-and-load-balancing.md)
- Cross-links: [AZURE/ENTRAID/](../ENTRAID/00-overview.md) | [AZURE/MONITOR/](../MONITOR/00-overview.md)
- [Microsoft Learn -- AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 free practice assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/practice/assessment?assessment-type=practice&assessmentId=21)
