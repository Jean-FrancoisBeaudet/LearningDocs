# Backup and Recovery

> **Exam mapping:** AZ-104 -- "Monitor and maintain Azure resources" (10--15 %)
> **One-liner:** Protect Azure workloads with Recovery Services vaults (VMs, SQL, Files) and Backup vaults (Blobs, Disks, AKS), orchestrate disaster recovery with Azure Site Recovery, and surface it all through Backup Center reports and Azure Monitor alerts.
> **Related:** 14-monitoring (when created) | [AZURE/AZ-305/03-business-continuity.md](../AZ-305/03-business-continuity.md) (architect-level DR design)

---

## 1 Recovery Services Vault

A [Recovery Services vault](https://learn.microsoft.com/en-us/azure/backup/backup-azure-recovery-services-vault-overview) is an Azure Resource Manager entity that stores **backup data, recovery points, and backup policies**. It is also the management container for **Azure Site Recovery** replication metadata.

### 1.1 Creating a vault

```bash
# Create a Recovery Services vault
az backup vault create \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --location eastus
```

Portal path: **Recovery Services vaults > + Create** -- select subscription, resource group, vault name, and region.

### 1.2 Storage redundancy

Set **before the first backup item is registered** -- once a protected item exists, the setting is locked.

| Option | Copies | Use case |
|--------|--------|----------|
| **LRS** (Locally-redundant) | 3 copies in one data centre | Dev/test, cost-sensitive workloads |
| **ZRS** (Zone-redundant) | 3 copies across availability zones | Production in regions with AZs |
| **GRS** (Geo-redundant) | 6 copies (3 primary + 3 paired region) | Business-critical workloads requiring cross-region restore |

```bash
# Change redundancy (only before first backup)
az backup vault backup-properties set \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --backup-storage-redundancy GeoRedundant
```

> **Exam trap:** You **cannot** change redundancy from GRS to LRS (or vice-versa) after the first backup item is configured. The portal greys out the option.

### 1.3 Soft delete

[Soft delete](https://learn.microsoft.com/en-us/azure/backup/backup-azure-security-feature-cloud) is **enabled by default** and retains deleted backup data for **14 additional days** at no extra cost. During those 14 days you can undelete the item; after expiry the data is permanently purged.

- **Always-on soft delete** (immutable): cannot be disabled -- available as a vault-level setting.
- Multi-user authorization (MUA) can protect soft-delete settings from being turned off by a single admin.

### 1.4 Immutability

Vault-level [immutability](https://learn.microsoft.com/en-us/azure/backup/backup-azure-immutable-vault-concept) prevents operations that could lead to data loss (reducing retention, disabling soft delete). States: **Disabled → Enabled (reversible) → Locked (irreversible)**.

### 1.5 Cross-region restore (CRR)

Available only for **GRS vaults**. Lets you restore backup data in the **Azure paired secondary region** even when the primary region is healthy -- useful for audits and DR drills.

```bash
# Enable CRR on a GRS vault
az backup vault backup-properties set \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --cross-region-restore-flag true
```

### 1.6 Supported workloads

| Workload | Agent / method |
|----------|---------------|
| **Azure VMs** (Windows/Linux) | VM backup extension (agentless) |
| **SQL Server in Azure VM** | Workload backup extension (stream backup to vault) |
| **SAP HANA in Azure VM** | Workload backup extension |
| **Azure Files** (file shares) | Snapshot-based, no agent |
| **On-premises files/folders** | [MARS agent](https://learn.microsoft.com/en-us/azure/backup/backup-azure-about-mars) |
| **On-premises VMs / Hyper-V / VMware** | DPM or [MABS](https://learn.microsoft.com/en-us/azure/backup/backup-mabs-whats-new) (Microsoft Azure Backup Server) |

---

## 2 Azure Backup Vault

A [Backup vault](https://learn.microsoft.com/en-us/azure/backup/backup-vault-overview) is a **newer vault type** built on the Azure Data Protection service. It is used for workloads that Recovery Services vaults do not cover.

```bash
# Create a Backup vault
az dataprotection backup-vault create \
  --resource-group rg-backup \
  --vault-name bv-prod-eastus \
  --location eastus \
  --storage-setting "[{type:LocallyRedundant,datastore-type:VaultStore}]"
```

### 2.1 Recovery Services vault vs Backup vault

| Feature | Recovery Services vault | Backup vault |
|---------|------------------------|-------------|
| **Azure VMs** | Yes | No |
| **SQL / SAP HANA in VM** | Yes | No |
| **Azure Files** | Yes | No |
| **Azure Blobs** | No | Yes (operational + vaulted) |
| **Azure Managed Disks** | No | Yes |
| **Azure Database for PostgreSQL** | No | Yes (Single Server & Flexible) |
| **Azure Kubernetes Service (AKS)** | No | Yes |
| **On-premises (MARS / DPM / MABS)** | Yes | No |
| **Site Recovery support** | Yes | No |
| **Soft delete** | 14-day default | 14-day default |
| **Cross-region restore** | GRS vaults | Select workloads |
| **CLI namespace** | `az backup vault` | `az dataprotection backup-vault` |

> **Exam trap:** Know which workload maps to which vault type. Azure VMs and SQL-in-VM always use **Recovery Services vault**; Azure Blobs, Disks, and AKS always use **Backup vault**.

---

## 3 Backup Policy

A [backup policy](https://learn.microsoft.com/en-us/azure/backup/backup-azure-arm-vms-prepare) defines **when backups run** (schedule) and **how long recovery points are kept** (retention).

### 3.1 Schedule options

| Policy tier | Minimum frequency | Granularity |
|-------------|-------------------|------------|
| **Standard** (VMs) | Once per day | Daily or weekly |
| **Enhanced** (VMs) | Every 4 hours | Hourly (4h, 6h, 8h, 12h), daily, weekly |
| **Azure Files** | Multiple per day (every 4h, 6h, 8h, 12h, daily) | Hourly/daily |
| **SQL / SAP HANA** | Every 15 minutes (log) | Log + full schedule |

### 3.2 Retention

Configure independent retention for each tier of recovery points:

| Tier | Retention range |
|------|----------------|
| **Daily** | 7--9999 days |
| **Weekly** | 1--5163 weeks |
| **Monthly** | 1--1188 months |
| **Yearly** | 1--99 years |

### 3.3 Instant restore snapshots

[Instant Restore](https://learn.microsoft.com/en-us/azure/backup/backup-instant-restore-capability) keeps a local VM snapshot on managed disks for fast recovery before the data is transferred to the vault.

| Policy | Default retention | Configurable range |
|--------|-------------------|-------------------|
| **Standard** | 2 days | 1--5 days |
| **Enhanced** | 7 days | 1--30 days |

> Snapshots are incremental (delta only after the first full copy), so longer retention adds modest cost.

### 3.4 CLI -- create and update

```bash
# List built-in policies
az backup policy list \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus

# Export a policy to JSON, tweak, and apply
az backup policy show \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --name DefaultPolicy \
  -o json > policy.json

# Create a custom policy from JSON
az backup policy create \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --name DailyVMPolicy \
  --policy policy.json

# Update an existing policy
az backup policy set \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --policy policy.json
```

---

## 4 Backup and Restore Operations

### 4.1 Azure VM backup

**How it works** -- the backup extension takes a snapshot of the VM disks, then transfers the snapshot data to the vault. The two phases run independently; you can restore from the local snapshot (instant restore) even before vault transfer completes.

#### Snapshot consistency types

| Type | Description | When it happens |
|------|-------------|-----------------|
| **Application-consistent** | Flushes in-memory data and pending I/O (uses VSS on Windows, pre/post scripts on Linux) | Default for running VMs with VSS/scripts |
| **File-system-consistent** | File system is consistent, but in-memory data may be lost | Linux VMs without pre/post scripts |
| **Crash-consistent** | Captures data on disk at the point in time (equivalent to power failure) | VM is off, or snapshot cannot quiesce the OS |

#### Enabling backup (CLI)

```bash
# Enable backup for a VM with the default policy
az backup protection enable-for-vm \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --vm myVM \
  --policy-name DefaultPolicy
```

#### On-demand backup

```bash
az backup protection backup-now \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-compute;myVM" \
  --item-name "VM;iaasvmcontainerv2;rg-compute;myVM" \
  --retain-until 2026-05-16
```

#### VM restore options

| Option | What happens |
|--------|-------------|
| **Create new VM** | Deploys a new VM from the recovery point |
| **Replace existing** | Swaps the disks on the existing VM (same availability set / zone required) |
| **Restore disks** | Restores managed disks to a storage account; you attach them manually or via template |
| **Cross-region restore** | Restore in the paired secondary region (GRS + CRR enabled) |
| **File-level recovery** | Mounts the recovery point disks as **iSCSI volumes** on a target machine; browse and copy individual files |

```bash
# Restore disks to a storage account
az backup restore restore-disks \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-compute;myVM" \
  --item-name "VM;iaasvmcontainerv2;rg-compute;myVM" \
  --rp-name <recovery-point-id> \
  --storage-account mystorageacct \
  --target-resource-group rg-restore
```

> **Exam trap:** File-level recovery works by downloading a script that **mounts iSCSI volumes** -- know this mechanism for the exam.

### 4.2 Azure Files backup

- Backed up via **share-level snapshots** (no agent, no vault transfer overhead).
- **Share-level restore**: restore the entire share to a point in time (original or alternate location).
- **File-level restore**: restore individual files or folders from a snapshot.
- Instant restore from snapshots -- restores are fast because data stays in the storage account.

```bash
# Enable backup for an Azure File share
az backup protection enable-for-azurefileshare \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --storage-account mystorageacct \
  --azure-file-share myshare \
  --policy-name AzureFilesPolicy
```

---

## 5 Azure Site Recovery (ASR) for Azure Resources

[Azure Site Recovery](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-tutorial-failover-failback) is an **orchestration service for disaster recovery** -- it replicates VMs to a secondary Azure region and automates failover / failback.

> **Key distinction:** Azure Backup = protect data (recovery points). Azure Site Recovery = protect workloads (entire VMs replicated for near-zero downtime DR).

### 5.1 Azure-to-Azure replication -- how it works

1. **Enable replication** on the source VM -- ASR installs the Mobility service extension.
2. Disk writes are **continuously replicated** to a **cache storage account** in the source region.
3. Data is sent from the cache to **managed disks (replica disks)** in the target region.
4. **Recovery points** are generated per the replication policy.

#### Replication policy settings

| Setting | Default | Description |
|---------|---------|-------------|
| **Recovery point retention** | 24 hours | How long recovery points are kept |
| **App-consistent snapshot frequency** | 4 hours | How often app-consistent recovery points are created |
| **RPO** (Recovery Point Objective) | Minutes (not zero) | Maximum acceptable data loss |

#### Requirements for Azure-to-Azure replication

- **Target region** -- must be different from source; typically the paired region.
- **Cache storage account** -- in the source region, stores writes before forwarding.
- **Target resource group** -- to hold the replica disks and NIC.
- **Target VNet** (or create one automatically).
- **Target availability set / zone** (optional, matched to source).
- **Sufficient quota** in the target region for VM size.

```bash
# Enable replication (simplified -- usually done via portal or PowerShell)
az site-recovery protected-item create \
  --resource-group rg-dr \
  --vault-name rsv-dr-eastus \
  --fabric-name "azure-eastus" \
  --protection-container "asr-a2a-default-eastus" \
  --name "myVM-replication" \
  --provider-specific-details '{a2a specific config JSON}'
```

> In practice, most admins enable ASR replication through the **portal** (VM blade > Disaster recovery) or **PowerShell** (`New-AzRecoveryServicesAsrReplicationProtectedItem`). The CLI `az site-recovery` namespace is available but less commonly used for this workflow.

---

## 6 Failover to a Secondary Region

### 6.1 Failover types

| Type | Impact on production | Use case |
|------|---------------------|----------|
| [**Test failover**](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-test-failover-to-azure) | **None** -- creates VMs in an isolated VNet | DR drills, compliance validation |
| **Failover** | Stops replication from source; target VM becomes live | Actual disaster or planned migration |

### 6.2 Failover workflow

```
1. Select recovery point
   ├── Latest processed (low RTO)
   ├── Latest (lowest RPO -- processes all pending data)
   ├── Latest app-consistent
   └── Custom
2. Failover executes → replica VM created in target region
3. **Commit** the failover (deletes earlier recovery points)
4. Verify application health
```

### 6.3 Re-protect (reverse replication)

After failover, the target-region VM is **not protected**. You must **re-protect** it -- this sets up replication back to the original (or a new) primary region.

### 6.4 Failback

Once re-protection replication is healthy, perform another failover back to the primary region (the "failback"). Then re-protect again to restore the original DR posture.

### 6.5 Recovery plans

A [recovery plan](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-create-recovery-plans) groups VMs into ordered steps:

- **Groups** -- VMs in the same group fail over in parallel.
- **Sequencing** -- Group 1 completes before Group 2 starts (e.g. database tier before app tier).
- **Custom actions** -- add Azure Automation runbooks, PowerShell scripts, or manual approval steps between groups.

```
Recovery Plan: "ERP-DR"
├── Group 1: SQL VMs (fail over first)
│   └── Post-action: run script to verify DB connectivity
├── Group 2: App-tier VMs
│   └── Pre-action: manual step "confirm DB is online"
└── Group 3: Web-tier VMs
    └── Post-action: run Traffic Manager failover script
```

> **Exam trap:** A **test failover** does **not** affect production replication and creates VMs in an isolated network. A real **failover** stops replication and you must **commit** then **re-protect**.

---

## 7 Reports and Alerts for Backups

### 7.1 Backup Center

[Backup Center](https://learn.microsoft.com/en-us/azure/backup/backup-center-overview) is the **single-pane-of-glass** for managing all backup resources across vaults, subscriptions, and regions. It surfaces:

- Backup instances, policies, and vaults
- Jobs and alerts
- Built-in Backup Reports workbooks

### 7.2 Backup reports

[Backup Reports](https://learn.microsoft.com/en-us/azure/backup/configure-reports) use **Azure Monitor workbooks** powered by **Log Analytics** queries.

**Setup steps:**

1. Configure **Diagnostic settings** on each Recovery Services vault -- send data to a **Log Analytics workspace**.
2. Wait up to 24 hours for the initial data push (ongoing lag is ~20--30 minutes).
3. Open **Backup Center > Backup Reports** -- the built-in workbooks render automatically.

```bash
# Configure diagnostic settings via CLI
az monitor diagnostic-settings create \
  --resource "/subscriptions/{sub}/resourceGroups/rg-backup/providers/Microsoft.RecoveryServices/vaults/rsv-prod-eastus" \
  --name "send-to-la" \
  --workspace "/subscriptions/{sub}/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/la-backup" \
  --logs '[{"category":"AzureBackupReport","enabled":true},{"category":"CoreAzureBackup","enabled":true},{"category":"AddonAzureBackupJobs","enabled":true},{"category":"AddonAzureBackupPolicy","enabled":true},{"category":"AddonAzureBackupStorage","enabled":true},{"category":"AddonAzureBackupProtectedInstance","enabled":true}]'
```

> Use the **Resource specific** destination table mode (not **Azure diagnostics** legacy mode) for better query performance and schema clarity.

Report categories include: backup items, usage, jobs, policies, and compliance/optimization recommendations.

### 7.3 Backup alerts

| Alert framework | Status | Scope |
|----------------|--------|-------|
| **Classic alerts** | **Deprecated** (March 31, 2026) | Per-vault, no integration with Azure Monitor |
| [**Azure Monitor-based alerts**](https://learn.microsoft.com/en-us/azure/backup/backup-azure-monitoring-alerts) | **Current (recommended)** | Centralized, action groups, at-scale management |

#### Built-in alert types (Azure Monitor-based)

| Alert | Severity | Trigger |
|-------|----------|---------|
| Backup job failure | Sev 1 | A scheduled or on-demand backup fails |
| Restore job failure | Sev 1 | A restore operation fails |
| Delete backup data | Sev 2 | Backup data is deleted (soft-delete kicks in) |
| Soft delete expiry approaching | Sev 2 | Soft-deleted data nearing permanent purge |
| Security alert | Sev 0 | Suspicious activity (e.g. backup disabled, data deleted from unusual IP) |

#### Configuring notifications

Alerts fire automatically, but **notifications** (email, SMS, webhook) require an **action group**:

1. Create an action group with desired notification targets.
2. Create an **alert processing rule** in Azure Monitor that applies the action group to backup alerts.

```bash
# Create an action group
az monitor action-group create \
  --resource-group rg-monitoring \
  --name ag-backup-alerts \
  --short-name BackupAG \
  --action email backupAdmin admin@contoso.com
```

> **Exam note:** Alerts are generated automatically by Azure Backup, but you do **not** receive emails/SMS unless you configure an action group and alert processing rule.

---

## Exam Traps

| Trap | Detail |
|------|--------|
| **RSV vs Backup vault workloads** | Azure VMs / SQL / SAP HANA / Files / on-prem = **Recovery Services vault**. Blobs / Disks / PostgreSQL / AKS = **Backup vault**. |
| **Soft delete 14-day default** | Enabled by default on both vault types. Deleted data is retained 14 days. Can be made always-on (irreversible). |
| **Can't change redundancy after first backup** | GRS/LRS/ZRS must be set **before** the first protected item. Portal locks the setting afterwards. |
| **ASR RPO is not zero** | ASR provides near-continuous replication but **RPO is measured in minutes**, not zero. Data in the replication pipeline can be lost. |
| **Test failover vs failover** | Test failover creates VMs in an **isolated VNet** with **no impact on production replication**. A real failover stops replication and requires commit + re-protect. |
| **File-level recovery = iSCSI mount** | VM file-level restore downloads a script that mounts recovery point disks as **iSCSI volumes** on a target machine. |
| **Cross-region restore requires GRS** | CRR is only available on vaults configured with **GRS** redundancy. LRS/ZRS vaults do not support it. |
| **Classic backup alerts deprecated** | Classic (per-vault) alerts were deprecated March 2026. Use **Azure Monitor-based alerts** with action groups. |
| **Backup reports need Log Analytics** | Reports will not show data until you configure **Diagnostic settings** pointing to a **Log Analytics workspace** (up to 24-hour initial delay). |

---

## Sources

- [Overview of Recovery Services vaults](https://learn.microsoft.com/en-us/azure/backup/backup-azure-recovery-services-vault-overview)
- [Overview of Backup vaults](https://learn.microsoft.com/en-us/azure/backup/backup-vault-overview)
- [What is Azure Backup?](https://learn.microsoft.com/en-us/azure/backup/backup-overview)
- [Azure Backup support matrix](https://learn.microsoft.com/en-us/azure/backup/backup-support-matrix)
- [Instant restore capability](https://learn.microsoft.com/en-us/azure/backup/backup-instant-restore-capability)
- [Enhanced policy for Azure VMs](https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-enhanced-policy)
- [Soft delete for Azure Backup](https://learn.microsoft.com/en-us/azure/backup/backup-azure-security-feature-cloud)
- [Azure Site Recovery: Azure-to-Azure failover/failback tutorial](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-tutorial-failover-failback)
- [Test failover to Azure](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-test-failover-to-azure)
- [About failover and failback (Modernized)](https://learn.microsoft.com/en-us/azure/site-recovery/failover-failback-overview-modernized)
- [Configure Azure Backup reports](https://learn.microsoft.com/en-us/azure/backup/configure-reports)
- [Monitoring and alerting for Azure Backup](https://learn.microsoft.com/en-us/azure/backup/monitoring-and-alerts-overview)
- [Manage Azure Monitor-based alerts](https://learn.microsoft.com/en-us/azure/backup/backup-azure-monitoring-alerts)
- [AZ-104 training path: Monitor and back up Azure resources](https://learn.microsoft.com/en-us/training/paths/az-104-monitor-backup-resources/)
