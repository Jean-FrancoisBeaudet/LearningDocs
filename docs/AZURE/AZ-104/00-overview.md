# AZ-104 — Microsoft Azure Administrator

**Core study overview and deep-dive launchpad**

> Target exam: **AZ-104** (Microsoft Certified: Azure Administrator Associate)
> Prerequisite for: **AZ-305** (Azure Solutions Architect Expert) — see [AZURE/AZ-305/](../AZ-305/00-overview.md)
> Passing score: **700 / 1000**
> Duration: **100 minutes**
> Current English exam update: **April 17, 2026** (localized versions follow ~8 weeks later)
> Sources: [Exam page](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/) · [Official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104) · [Certification path](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/)

---

## Exam shape

- ~40–60 scored items: multiple choice, drag-and-drop, hot-area (click-on-diagram), build-list/reorder, and short case-study scenarios.
- The exam is **admin-focused, not design-focused**: questions test "how do you configure X" and "which tool/blade/command accomplishes Y," not architectural trade-offs (that is AZ-305's territory).
- Expect portal-screenshot questions, Azure CLI / PowerShell snippet interpretation, ARM template / Bicep file interpretation, and "what happens if you do X" operational scenarios.
- No formal case studies like AZ-305, but multi-part scenarios (a shared scenario text with 2–3 questions) are common.
- **Recurring decision axes**: correct scope for RBAC assignments, storage redundancy & access tier choices, NSG rule evaluation order, backup vs Site Recovery, and metric/log query interpretation.

## Skill areas & weightings (April 2026 update)

| # | Domain | Weight |
|---|---|---|
| 1 | Manage Azure identities and governance | **20–25 %** |
| 2 | Implement and manage storage | **15–20 %** |
| 3 | Deploy and manage Azure compute resources | **20–25 %** |
| 4 | Implement and manage virtual networking | **15–20 %** |
| 5 | Monitor and maintain Azure resources | **10–15 %** |

Domains 1 and 3 are jointly the heaviest, together accounting for up to **50 %** of the exam. Domain 5 is the smallest but delivers easy marks if you know [Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/overview) and [Azure Backup](https://learn.microsoft.com/en-us/azure/backup/backup-overview) well.

## How to use this overview

Each domain section below contains:
1. **Scope** — what the objective domain covers.
2. **Core services & concepts** — the must-know items.
3. **Exam traps** — distinctions the exam loves to test.
4. **Deep-dive prompt** — paste into a new Claude session with `/azure-expert` to generate the corresponding topical note in this folder.

> **Style convention for all deep-dive notes:** embed inline `learn.microsoft.com` hyperlinks on the first mention of every Azure service, feature, and limit — not only a trailing Sources list. Each prompt below already carries this instruction.

---

## 1. Manage Azure identities and governance (20–25 %)

### Scope
Identity lifecycle in [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/whatis) (users, groups, licenses, external identities), role-based access control, subscriptions, management groups, policy, locks, tags, and cost governance.

### Core services & concepts
- **Users & groups**: cloud-only vs synced, dynamic vs assigned membership groups, bulk operations, [self-service password reset (SSPR)](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-howitworks).
- **Licenses**: Entra ID Free vs P1 vs P2 feature gates, group-based license assignment.
- **External users**: [B2B guest invitations](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b), redemption flow, guest vs member user type.
- **Azure RBAC**: [built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) (Owner, Contributor, Reader, User Access Administrator), custom roles, scope hierarchy (management group → subscription → resource group → resource), deny assignments.
- **Subscriptions & management groups**: [management group](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview) hierarchy, moving subscriptions, subscription limits.
- **Azure Policy**: [policy definitions, initiatives, assignment scopes, effects](https://learn.microsoft.com/en-us/azure/governance/policy/overview) (`Deny`, `Audit`, `DeployIfNotExists`, `Modify`), exemptions, compliance dashboard.
- **Resource locks**: `CanNotDelete` vs `ReadOnly` — scope inheritance.
- **Tags**: tag inheritance via Policy, cost allocation tags.
- **Cost management**: [budgets, alerts, Azure Advisor cost recommendations](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management).

### Exam traps
- **RBAC scope inheritance** flows down — a role at subscription scope applies to every resource group and resource below it. You *cannot* break inheritance; you can only add a deny assignment at a lower scope.
- **Owner vs User Access Administrator**: Owner can manage resources *and* access; User Access Administrator can only manage access, not resources.
- **Dynamic group** membership requires Entra ID P1 or higher.
- **Policy `DeployIfNotExists`** requires a [managed identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) on the assignment for remediation — often a distractor in wrong answers.
- **Resource locks** apply to the management plane, not the data plane: a `ReadOnly` lock on a storage account blocks key rotation but *not* blob uploads.
- **Management groups** cannot contain the same subscription in two branches — a subscription has exactly one parent.

### Deep-dive prompts
```
/azure-expert Deep-dive AZ-104 "Manage Microsoft Entra users and groups" (part of 20–25 %).
Produce AZURE/AZ-104/01-entra-users-and-groups.md covering:
User creation (cloud vs synced), bulk operations, group types (security/M365,
assigned/dynamic), dynamic membership rules syntax, license assignment (direct vs
group-based), SSPR configuration, external users (B2B guest invite, redemption,
user type). Include portal walkthrough steps, Azure CLI / PowerShell examples,
and "Exam traps". Cross-link to AZURE/ENTRAID/ for deeper Entra ID coverage.
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Manage access to Azure resources" (part of 20–25 %).
Produce AZURE/AZ-104/02-access-to-azure-resources.md covering:
Azure RBAC model (scope hierarchy, built-in roles, custom roles, deny assignments),
role assignment interpretation (effective access), classic subscription administrator
roles vs RBAC, Entra ID roles vs Azure roles distinction. Include CLI/PowerShell
role assignment examples and "Exam traps". Cross-link to AZURE/ENTRAID/05-rbac-and-authorization.md.
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Manage Azure subscriptions and governance" (part of 20–25 %).
Produce AZURE/AZ-104/03-subscriptions-and-governance.md covering:
Management groups (hierarchy, moving subs), Azure Policy (definitions, initiatives,
effects, remediation tasks, exemptions), resource locks (CanNotDelete vs ReadOnly,
scope behaviour), tags (enforcement via Policy, inheritance, cost allocation),
resource groups (moving resources, deletion behaviour), cost management (budgets,
alerts, Advisor recommendations, spending limits). Include Bicep/CLI snippets
and "Exam traps". Embed inline Microsoft Learn hyperlinks on first mention of
every service and feature.
```

---

## 2. Implement and manage storage (15–20 %)

### Scope
[Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-introduction) account configuration, access control (firewalls, SAS, keys, identity-based), redundancy, and blob/file lifecycle.

### Core services & concepts
- **Storage account types**: Standard general-purpose v2 vs Premium (block blob, file shares, page blobs), [performance tiers](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview).
- **Redundancy**: LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS — know the durability nines and read-access behaviour.
- **Network access**: [storage firewalls & virtual network rules](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security), private endpoints, service endpoints, trusted Azure services exception.
- **Access keys** and key rotation; [shared access signatures (SAS)](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview) — account SAS vs service SAS vs user delegation SAS, stored access policies.
- **Identity-based access for Azure Files**: Entra DS, on-prem AD DS, Entra Kerberos (for hybrid identities).
- **Object replication**, **AzCopy**, and **Storage Explorer**.
- **Blob Storage**: [access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) (Hot / Cool / Cold / Archive), lifecycle management policies, soft delete, versioning, snapshots, immutable blobs.
- **Azure Files**: [SMB / NFS file shares](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction), share snapshots, soft delete.

### Exam traps
- **ZRS vs GRS**: ZRS keeps 3 copies across availability zones *within one region*; GRS replicates to a paired *secondary region* but secondary is not readable unless RA-GRS or RA-GZRS.
- **Archive tier** is *offline* — rehydration takes hours (standard) or up to 15 hours (high priority). Minimum storage duration: 180 days; early deletion fees apply.
- **User delegation SAS** is the most secure SAS type (signed with Entra credentials, not access keys) — a frequent "best practice" distractor.
- **Stored access policy** can only be created on a *service SAS* (not account SAS).
- **Storage account firewall "Allow trusted Microsoft services"** bypass does *not* cover all Azure services — only a defined list (e.g., Azure Backup, Azure Monitor, Event Grid).
- **Moving a storage account** to another resource group or subscription does *not* change the endpoint URL; moving to another region requires data migration.
- **Soft delete** for blobs vs containers vs file shares are independently configured — enabling one does not enable the others.

### Deep-dive prompts
```
/azure-expert Deep-dive AZ-104 "Configure access to storage" (part of 15–20 %).
Produce AZURE/AZ-104/04-storage-access-and-security.md covering:
Storage firewalls & VNet rules, service endpoints vs private endpoints, access keys
& rotation, SAS types (account/service/user delegation) with decision matrix, stored
access policies, identity-based access for Azure Files (AD DS, Entra DS, Entra Kerberos).
Include CLI/PowerShell examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Configure and manage storage accounts" (part of 15–20 %).
Produce AZURE/AZ-104/05-storage-accounts.md covering:
Storage account creation & settings, performance tiers (Standard vs Premium),
redundancy options (LRS/ZRS/GRS/RA-GRS/GZRS/RA-GZRS) with durability table,
encryption (Microsoft-managed vs customer-managed keys), object replication,
AzCopy and Storage Explorer. Include CLI examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Configure Azure Files and Azure Blob Storage" (part of 15–20 %).
Produce AZURE/AZ-104/06-azure-files-and-blob.md covering:
Azure Files (SMB/NFS, tiers, share snapshots, soft delete), Blob Storage (containers,
access tiers Hot/Cool/Cold/Archive, lifecycle management rules, soft delete, versioning,
snapshots, immutable storage WORM). Include lifecycle policy JSON examples and
"Exam traps". Embed inline Microsoft Learn hyperlinks on first mention of every
service and feature.
```

---

## 3. Deploy and manage Azure compute resources (20–25 %)

### Scope
IaC with [ARM templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) and [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview), virtual machines, containers ([Azure Container Instances](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview), [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview)), and [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/overview).

### Core services & concepts
- **ARM / Bicep**: interpret & modify templates, deploy via CLI/PowerShell/portal, export deployments, convert ARM JSON to Bicep.
- **Virtual machines**: VM creation, [sizes (families)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview), disk types (Standard HDD, Standard SSD, Premium SSD, Ultra), encryption at host, [availability zones & availability sets](https://learn.microsoft.com/en-us/azure/virtual-machines/availability), moving VMs across RG/subscription/region, [VM Scale Sets](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) (Uniform vs Flexible orchestration).
- **Containers**: [Azure Container Registry (ACR)](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro), ACI (quick single-container workloads), Azure Container Apps (microservices with scaling/Dapr), sizing & scaling for both.
- **App Service**: [App Service plans](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans) (Free/Shared/Basic/Standard/Premium/Isolated), scaling (up & out, autoscale rules), TLS/SSL certificates, custom DNS mapping, deployment slots (swap, slot settings), backup/restore, VNet integration, access restrictions.

### Exam traps
- **Availability set vs availability zone**: sets distribute across fault/update domains *within a datacenter*; zones distribute across *physically separate datacenters* in a region. You cannot combine both for the same VM.
- **VMSS Uniform vs Flexible**: Flexible is the recommended mode for new deployments (supports mixed VM sizes, zone spreading) — exam questions increasingly reference Flexible.
- **Deployment slots** are available only at **Standard tier and above** in App Service. Slot-specific settings ("sticky settings") do *not* swap.
- **Azure Container Apps** vs **ACI**: ACA provides built-in scaling (KEDA), Dapr, revisions, and ingress — ACI is simpler but lacks auto-scaling and microservice features.
- **ARM template `dependsOn`** vs implicit dependencies via `reference()` — a template question may require you to identify missing dependencies.
- **Encryption at host** encrypts temp disk and OS/data disk caches — it is *not* the same as Azure Disk Encryption (ADE), which uses BitLocker/DM-Crypt inside the VM.
- **Moving a VM** to another region requires Azure Resource Mover or re-creation; you cannot simply move the resource.

### Deep-dive prompts
```
/azure-expert Deep-dive AZ-104 "Automate deployment of resources by using ARM templates or Bicep files" (part of 20–25 %).
Produce AZURE/AZ-104/07-arm-and-bicep.md covering:
ARM template structure (schema, parameters, variables, resources, outputs), Bicep
syntax equivalents, interpreting & modifying templates, deploying (CLI/PowerShell/portal),
exporting deployments, ARM-to-Bicep conversion, linked/nested templates, what-if.
Include annotated template examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Create and configure virtual machines" (part of 20–25 %).
Produce AZURE/AZ-104/08-virtual-machines.md covering:
VM creation, sizes & families, disk types & management, encryption at host vs ADE,
availability sets vs zones, VM Scale Sets (Uniform vs Flexible), moving VMs (RG,
subscription, region), custom script extension, boot diagnostics, serial console.
Include CLI/PowerShell examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Provision and manage containers" (part of 20–25 %).
Produce AZURE/AZ-104/09-containers.md covering:
Azure Container Registry (SKUs, tasks, geo-replication), Azure Container Instances
(container groups, restart policies, environment variables, mounted volumes),
Azure Container Apps (environments, revisions, scaling rules, Dapr, ingress),
sizing & scaling for both ACI and ACA. Include CLI examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Create and configure Azure App Service" (part of 20–25 %).
Produce AZURE/AZ-104/10-app-service.md covering:
App Service plans (tiers, scaling up & out, autoscale), app creation & configuration,
TLS/SSL certificates, custom DNS mapping (CNAME vs A record, domain verification),
deployment slots (swap behaviour, slot settings, traffic routing), backup & restore,
VNet integration, hybrid connections, access restrictions. Include CLI examples
and "Exam traps". Embed inline Microsoft Learn hyperlinks on first mention of every
service and feature.
```

---

## 4. Implement and manage virtual networking (15–20 %)

### Scope
[Virtual networks](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview), subnets, peering, public IPs, routing, network security, service/private endpoints, DNS, and load balancing.

### Core services & concepts
- **VNets & subnets**: address spaces, subnet delegation, reserved addresses (5 per subnet).
- **VNet peering**: regional vs [global peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview), non-transitive nature, gateway transit.
- **Public IP addresses**: Basic vs Standard SKU, static vs dynamic, zone-redundant.
- **User-defined routes (UDR)**: [route tables](https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table), next-hop types (Virtual appliance, VNet gateway, Internet, None), BGP route propagation.
- **NSGs & ASGs**: [network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) (rules, priority, default rules, effective rules), [application security groups](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups) for logical grouping.
- **Azure Bastion**: [secure RDP/SSH without public IPs](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview), SKU tiers (Developer, Basic, Standard, Premium).
- **Service endpoints** vs **[private endpoints](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)**: connectivity to PaaS over Microsoft backbone vs private IP in your VNet.
- **Azure DNS**: [public DNS zones, private DNS zones](https://learn.microsoft.com/en-us/azure/dns/dns-overview), record types, alias records.
- **Load balancing**: [Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview) (L4, Basic vs Standard SKU, internal vs public, health probes, rules, HA ports).

### Exam traps
- **VNet peering is non-transitive**: VNet A ↔ B and B ↔ C does *not* give A ↔ C connectivity. Use hub-spoke with a gateway or NVA.
- **NSG rule evaluation**: rules processed in priority order (lowest number = highest priority). Once a match is found, processing stops. Default rules (65000–65500) cannot be deleted.
- **Basic vs Standard public IP**: Basic allows dynamic or static, is open by default, no zone support. Standard is always static, secure-by-default (requires NSG to allow traffic), supports zones.
- **Basic vs Standard Load Balancer**: Basic is free but supports only availability sets; Standard supports zones, HA ports, outbound rules, and is required for cross-zone scenarios.
- **Service endpoint** keeps traffic on the Microsoft backbone but the PaaS resource still has a *public* IP — it is not a *private* connection. **Private endpoint** assigns a private IP in your VNet.
- **Azure DNS private zones** need a [virtual network link](https://learn.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links) (with or without auto-registration) to resolve records from a VNet.
- **UDR next hop "None"** drops traffic — used to black-hole routes (e.g., block Internet egress from a subnet).

### Deep-dive prompts
```
/azure-expert Deep-dive AZ-104 "Configure and manage virtual networks in Azure" (part of 15–20 %).
Produce AZURE/AZ-104/11-virtual-networks.md covering:
VNet creation & address spaces, subnets (reserved IPs, delegation), VNet peering
(regional/global, non-transitive, gateway transit, allow forwarded traffic),
public IP addresses (Basic vs Standard, static vs dynamic), user-defined routes
(route tables, next-hop types, BGP propagation), network troubleshooting
(effective routes, next hop, IP flow verify). Include CLI examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Configure secure access to virtual networks" (part of 15–20 %).
Produce AZURE/AZ-104/12-network-security.md covering:
NSGs (rules, priority, default rules, effective security rules, NIC vs subnet association),
ASGs, Azure Bastion (SKUs, deployment), service endpoints (configuration, policies),
private endpoints (Private Link, DNS integration, approval workflow). Include CLI
examples and "Exam traps". Embed inline Microsoft Learn hyperlinks on first mention
of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Configure name resolution and load balancing" (part of 15–20 %).
Produce AZURE/AZ-104/13-dns-and-load-balancing.md covering:
Azure DNS (public zones, private zones, virtual network links, auto-registration,
record types, alias records), Azure Load Balancer (Standard vs Basic, internal vs
public, health probes, load-balancing rules, inbound NAT rules, outbound rules,
HA ports), troubleshooting load balancing. Include CLI examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

---

## 5. Monitor and maintain Azure resources (10–15 %)

### Scope
[Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/overview) (metrics, logs, alerts), [Azure Backup](https://learn.microsoft.com/en-us/azure/backup/backup-overview), and [Azure Site Recovery](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview). See also [AZURE/MONITOR/](../MONITOR/00-overview.md) for deeper monitoring notes.

### Core services & concepts
- **Azure Monitor metrics**: platform metrics (automatic), custom metrics, metrics explorer, [Data Collection Rules (DCR)](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview).
- **Azure Monitor logs**: [Log Analytics workspaces](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview), KQL queries, diagnostic settings (route to LA, Storage, Event Hub).
- **Alerts**: [alert rules](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview) (metric, log search, activity log), action groups (email, SMS, webhook, Logic App, Azure Function, ITSM), alert processing rules (suppression, routing).
- **Insights**: VM insights, Storage insights, Network insights, [Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview) (Connection monitor, IP flow verify, NSG diagnostics, next hop, packet capture).
- **Azure Backup**: [Recovery Services vault vs Backup vault](https://learn.microsoft.com/en-us/azure/backup/backup-azure-recovery-services-vault-overview), backup policies (frequency, retention), VM backup, file-level restore, cross-region restore.
- **Azure Site Recovery**: Azure-to-Azure replication, recovery plans, test failover vs planned failover, reprotect & failback.
- **Backup reports & alerts**: configuring diagnostic settings on vaults, built-in backup alerts.

### Exam traps
- **Metrics vs Logs**: metrics are numeric time-series (near-real-time, 93 days retention by default); logs are text/structured records in a Log Analytics workspace (configurable retention up to 730 days in archive).
- **Data Collection Rules** replace the legacy Azure Diagnostics extension and Log Analytics agent — the exam now tests DCR-based collection.
- **Alert processing rules** are *not* alert rules: they suppress or modify notifications *after* an alert fires (e.g., suppress during maintenance windows).
- **Recovery Services vault vs Backup vault**: Backup vault is the newer resource that handles Azure Blobs, Azure Disks, Azure Database for PostgreSQL; Recovery Services vault handles VMs, SQL in VM, Azure Files, SAP HANA.
- **Site Recovery test failover** does *not* impact the source VM or ongoing replication — it creates resources in an isolated VNet. A real failover *stops* replication.
- **Azure Backup** soft delete is enabled by default on Recovery Services vaults — deleted backup data is retained for 14 extra days.

### Deep-dive prompts
```
/azure-expert Deep-dive AZ-104 "Monitor resources in Azure" (part of 10–15 %).
Produce AZURE/AZ-104/14-monitoring.md covering:
Azure Monitor metrics (platform/custom, metrics explorer), log settings (diagnostic
settings, DCRs, Log Analytics workspace), KQL query basics for the exam, alert rules
(metric/log/activity log), action groups, alert processing rules, VM/Storage/Network
insights, Azure Network Watcher tools (connection monitor, IP flow verify, NSG
diagnostics, next hop, packet capture). Include KQL examples and "Exam traps".
Cross-link to AZURE/MONITOR/ for deeper coverage.
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

```
/azure-expert Deep-dive AZ-104 "Implement backup and recovery" (part of 10–15 %).
Produce AZURE/AZ-104/15-backup-and-recovery.md covering:
Recovery Services vault vs Backup vault (workload support matrix), backup policies
(frequency, retention, GFS), VM backup (consistent snapshots, file-level restore,
cross-region restore), Azure Files backup, Azure Site Recovery (A2A replication,
recovery plans, test/planned/unplanned failover, reprotect & failback), backup
reports & alerts configuration. Include CLI examples and "Exam traps".
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

---

## Cross-cutting foundations

These skills do not map to a single domain but show up across the entire exam:

| Foundation | Why it matters |
|---|---|
| **PowerShell (`Az` module)** | Many questions present a PowerShell snippet and ask you to complete or interpret it. Know `New-Az*`, `Get-Az*`, `Set-Az*`, `Remove-Az*` patterns. |
| **Azure CLI (`az`)** | Same as above but with `az group`, `az vm`, `az network`, `az storage`, etc. Know the `--query` JMESPath flag. |
| **Azure portal** | Screenshot-based questions: identify the correct blade, setting, or option. |
| **ARM templates / Bicep** | Interpret `parameters`, `variables`, `resources`, `outputs` blocks. Know `dependsOn`, `reference()`, and what `[resourceId()]` resolves to. |
| **Microsoft Entra ID basics** | Users, groups, RBAC, managed identities — see [AZURE/ENTRAID/](../ENTRAID/00-overview.md) for comprehensive coverage. |
| **Tagging & naming conventions** | Cost allocation, Policy enforcement, automation filtering. |
| **Azure Resource Manager (ARM)** | Understand control plane vs data plane — many "why is this failing" questions hinge on this distinction. |

### Deep-dive prompt (cheatsheet)
```
/azure-expert Produce AZURE/AZ-104/16-exam-cheatsheet.md — a rapid-reference
cheatsheet for AZ-104 covering: CLI vs PowerShell command equivalents for common
operations (VM, storage, network, RBAC, policy), ARM template vs Bicep syntax
side-by-side, key Azure limits/quotas (VNets per subscription, subnets per VNet,
NSG rules, storage account limits, VM sizes), quick-reference tables for redundancy
options, access tier minimum durations, RBAC built-in roles, NSG default rules,
and a "last-day cram" section with the top 30 gotchas. Cross-link to AZURE/ENTRAID/
and AZURE/MONITOR/ for identity and monitoring details.
Embed inline Microsoft Learn hyperlinks on first mention of every service and feature.
```

---

## Suggested study path

1. Read this overview end-to-end and bookmark the [official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104).
2. Generate the **cheatsheet** first (section 16 prompt) — it gives you a rapid-reference anchor.
3. Generate deep-dives in exam-weight order: **Identities & Governance (01–03) → Compute (07–10) → Storage (04–06) → Networking (11–13) → Monitoring & Backup (14–15)**.
4. Hands-on labs: use [Microsoft Learn sandbox exercises](https://learn.microsoft.com/en-us/training/browse/?products=azure&resource_type=module&expanded=azure&roles=administrator) or an Azure free account to practice every CLI/portal operation.
5. Take the [free practice assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/practice/assessment?assessment-type=practice&assessmentId=21) and review wrong answers against the deep-dive notes.
6. Cross-reference [AZURE/AZ-305/](../AZ-305/00-overview.md) for architect-level context — understanding *why* a design is chosen helps you pick the correct admin action.
7. Re-check the [study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104) shortly before exam day — Microsoft publishes change logs within it.

---

## Existing cross-links

| Topic | Deeper notes in this repo |
|---|---|
| Microsoft Entra ID (identity, RBAC, Conditional Access, PIM) | [AZURE/ENTRAID/](../ENTRAID/00-overview.md) |
| Azure Monitor (metrics, logs, KQL, alerts, insights) | [AZURE/MONITOR/](../MONITOR/00-overview.md) |
| AZ-305 architect-level context | [AZURE/AZ-305/](../AZ-305/00-overview.md) |

---

## Sources

- [Microsoft Learn — Exam AZ-104](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/)
- [Microsoft Learn — AZ-104 official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [Microsoft Certified: Azure Administrator Associate](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/)
- [Exam readiness: Manage Azure identities and governance (1/5)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-104-manage-azure-identities-and-governance-1-of-5)
- [Exam readiness: Monitor and maintain Azure resources (5/5)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-104-monitor-and-maintain-azure-resources-5-of-5)
- [AZ-104 free practice assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/practice/assessment?assessment-type=practice&assessmentId=21)
- [Azure documentation](https://learn.microsoft.com/en-us/azure/)
