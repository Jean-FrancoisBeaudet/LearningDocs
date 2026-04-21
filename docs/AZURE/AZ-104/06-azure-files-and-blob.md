# Azure Files and Azure Blob Storage

> **Exam mapping:** AZ-104 --> "Implement and manage storage" (15-20%)
> **One-liner:** Configure file shares (SMB/NFS), blob containers, access tiers, soft delete, snapshots, lifecycle policies, and versioning to manage data cost-effectively in Azure Storage.
> **Related:** [04-storage-access](./04-storage-access.md) | [05-storage-accounts](./05-storage-accounts.md) | [AZ-305 Data Storage](../AZ-305/02-data-storage.md)

---

## 1. Azure Files

Azure Files provides fully managed file shares in the cloud accessible via **SMB 3.x** and **NFS 4.1** protocols. Shares are backed by Azure Storage and can be mounted concurrently by cloud or on-premises deployments.

**Reference:** [Azure Files overview](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)

### 1.1 Protocols

| Protocol | Storage Account Kind | Auth | OS Support | Notes |
|----------|---------------------|------|------------|-------|
| **SMB 3.x** | GPv2 (Standard) or FileStorage (Premium) | Identity-based (Entra DS, AD DS, Entra Kerberos) or storage key | Windows, Linux, macOS | Default protocol; encryption in transit required |
| **NFS 4.1** | **FileStorage (Premium) only** | Network-based (private endpoint / VNet) | Linux | No SMB on the same share; no Windows support; no identity auth |

### 1.2 Performance and Access Tiers

Azure Files tiers split into two categories: **Premium** (provisioned, SSD) and **Standard** (pay-as-you-go, HDD).

| Tier | Storage Account Kind | Media | Billing Model | Best For |
|------|---------------------|-------|---------------|----------|
| **Premium** | FileStorage | SSD | Provisioned (pay for capacity reserved) | IO-intensive: databases, dev environments, HPC |
| **Transaction Optimized** | GPv2 | HDD | Pay-as-you-go | Transaction-heavy workloads; default for standard shares |
| **Hot** | GPv2 | HDD | Pay-as-you-go | General-purpose: team shares, Azure File Sync |
| **Cool** | GPv2 | HDD | Pay-as-you-go | Online archive: infrequently accessed data |

> Standard tiers can be changed on the fly (no data movement). Premium cannot be converted to standard or vice versa -- you must copy data.

### 1.3 Creating a File Share

**Portal:** Storage account --> File shares --> + File share --> set name, tier, quota (GiB).

**Azure CLI:**

```bash
# Create a standard file share (Transaction Optimized tier, 100 GiB quota)
az storage share-rm create \
  --resource-group myRG \
  --storage-account mystorageacct \
  --name myshare \
  --access-tier "TransactionOptimized" \
  --quota 100

# Create a premium file share (requires FileStorage account)
az storage share-rm create \
  --resource-group myRG \
  --storage-account mypremiumsa \
  --name premshare \
  --quota 1024   # provisioned capacity in GiB
```

### 1.4 Quota Management

- Standard shares: quota sets the **maximum size** of the share (up to 100 TiB with large file shares enabled). You pay only for data stored.
- Premium shares: quota sets the **provisioned size** -- you pay for this amount regardless of usage, and IOPS/throughput scale with provisioned capacity.
- Quota can be increased or decreased at any time (decreasing below current usage is blocked).

### 1.5 Mounting File Shares

**Windows (SMB):**

```powershell
# Map drive letter Z: to the share
net use Z: \\mystorageacct.file.core.windows.net\myshare /u:AZURE\mystorageacct <storage-account-key>

# Persistent mount (survives reboot)
cmdkey /add:mystorageacct.file.core.windows.net /user:AZURE\mystorageacct /pass:<key>
New-PSDrive -Name Z -PSProvider FileSystem -Root "\\mystorageacct.file.core.windows.net\myshare" -Persist
```

**Linux (SMB):**

```bash
sudo mount -t cifs //mystorageacct.file.core.windows.net/myshare /mnt/myshare \
  -o vers=3.0,username=mystorageacct,password=<key>,dir_mode=0777,file_mode=0777,serverino

# For persistent mount, add to /etc/fstab (use credentials file for security)
```

**Linux (NFS) -- Premium only:**

```bash
sudo mount -t nfs mystorageacct.file.core.windows.net:/mystorageacct/premshare /mnt/premshare \
  -o vers=4,minorversion=1,sec=sys
```

> Port 445 (SMB) must be open. Many ISPs and corporate firewalls block it -- use VPN, ExpressRoute, or Azure File Sync as workarounds.

### 1.6 Share Snapshots

- **Share-level, read-only, incremental** point-in-time copies of the entire file share.
- Up to **200 snapshots** per share; retained for up to **10 years**.
- Created via Portal, CLI (`az storage share snapshot`), REST API, or Azure Backup.
- Use cases: restore individual files, restore full share, protect against application errors.
- Snapshots are differential -- only changes since the last snapshot are stored (cost-efficient).
- Deleting a share requires deleting all its snapshots first (unless using soft delete).

**Reference:** [Azure Files share snapshots](https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files)

### 1.7 Soft Delete for File Shares

Soft delete protects file shares from **accidental deletion** at the share level. It does **not** protect individual files/folders within a share -- use snapshots or Azure Backup for that.

- **Enabled at the storage account level** (applies to all file shares in the account).
- Retention period: **1 to 365 days** (default: 7 days; Microsoft recommends >= 14 days for backed-up shares).
- Deleted shares (and their snapshots) are retained in a soft-deleted state during the retention window.
- Recover via Portal (show deleted shares) or CLI.

```bash
# Enable soft delete (14-day retention)
az storage account file-service-properties update \
  --resource-group myRG \
  --account-name mystorageacct \
  --enable-delete-retention true \
  --delete-retention-days 14
```

**Reference:** [Soft delete for Azure file shares](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-prevent-file-share-deletion)

### 1.8 Azure File Sync (Brief Overview)

Azure File Sync allows you to cache Azure file shares on **Windows Server** for local-speed access while keeping a full copy in Azure. Key points for AZ-104:

- **Cloud tiering**: frequently accessed files cached locally; cold files replaced with pointers (stubs) and recalled on demand.
- **Components**: Storage Sync Service (Azure resource) --> Sync group --> Cloud endpoint (Azure file share) + Server endpoints (paths on registered Windows Servers).
- **Multi-site sync**: multiple server endpoints can sync with the same cloud endpoint, enabling branch-office scenarios.
- Requires **Azure File Sync agent** on Windows Server 2016+.

---

## 2. Blob Containers

Azure Blob Storage organises objects into **containers** (flat namespace unless hierarchical namespace is enabled for Data Lake Storage Gen2).

### 2.1 Creating a Container

**Portal:** Storage account --> Containers --> + Container --> name + access level.

**Azure CLI:**

```bash
az storage container create \
  --name mycontainer \
  --account-name mystorageacct \
  --public-access off          # Private (default)
```

### 2.2 Container Naming Rules

- Lowercase letters, numbers, and hyphens only.
- Must start with a letter or number.
- Length: **3 to 63 characters**.
- No consecutive hyphens.

### 2.3 Public Access Levels

| Level | Portal Label | Description |
|-------|-------------|-------------|
| **Private** | `off` | No anonymous access (default). All requests require authorisation. |
| **Blob** | `blob` | Anonymous read access to **blobs only**. Cannot list blobs in the container. |
| **Container** | `container` | Anonymous read access to blobs **and** the ability to list all blobs in the container. |

> Public access must also be **allowed at the storage account level** (`--allow-blob-public-access true`). If disabled at the account level, per-container settings are irrelevant. Microsoft recommends disabling public access for production accounts.

### 2.4 Metadata

- Each container and each blob can store **user-defined metadata** as name-value pairs (up to 8 KiB total per resource).
- Set via Portal, CLI (`az storage container metadata update`), or SDK.
- Metadata names are case-insensitive and must be valid C# identifiers.

---

## 3. Storage Tiers (Blob Access Tiers)

Azure Blob Storage supports four access tiers to optimise cost based on data access patterns.

**Reference:** [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)

### 3.1 Tier Comparison

| Tier | Online? | Storage Cost | Access Cost | Min Storage Duration | Best For |
|------|---------|-------------|-------------|---------------------|----------|
| **Hot** | Yes | Highest | Lowest | None | Frequently accessed data |
| **Cool** | Yes | Lower | Higher | **30 days** | Infrequent access (>= 30 days) |
| **Cold** | Yes | Lower still | Higher still | **90 days** | Rare access (>= 90 days) |
| **Archive** | **No (offline)** | Lowest | Highest | **180 days** | Long-term retention, compliance |

### 3.2 Default Account Tier vs Per-Blob Tier

- **Default access tier** is set at the storage account level: **Hot** or **Cool** (Cold and Archive cannot be set as default).
- Individual blobs can have their tier set explicitly, overriding the account default.
- When a blob has no explicit tier, it inherits the account default.
- Changing the account default tier re-tiers all blobs that do not have an explicitly set tier.

```bash
# Set account default tier to Cool
az storage account update \
  --name mystorageacct \
  --resource-group myRG \
  --access-tier Cool

# Set individual blob tier to Archive
az storage blob set-tier \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myblob.zip \
  --tier Archive
```

### 3.3 Early Deletion Penalties

If a blob is deleted or moved to a different tier before the minimum storage duration, you are charged for the **remaining days**.

Example: a blob moved to Cool and deleted after 10 days incurs a penalty for the remaining 20 days (30 - 10).

### 3.4 Rehydration from Archive

Archive blobs are **offline** -- they cannot be read or modified until rehydrated to an online tier (Hot, Cool, or Cold).

| Rehydration Priority | Typical Duration | Notes |
|---------------------|-----------------|-------|
| **Standard** | Up to **15 hours** | Default; lower cost |
| **High** | Under **1 hour** for objects < 10 GB | Higher cost; not guaranteed for large objects |

**Two rehydration methods:**

1. **Copy Blob** (recommended): copies the archived blob to a new blob in an online tier. Original stays in Archive. Use `az storage blob copy start --tier Hot --rehydrate-priority High`.
2. **Set Blob Tier**: changes the tier of the existing blob in place. Blob remains unavailable until rehydration completes.

> You **cannot** rehydrate from Archive using lifecycle management policies. Lifecycle policies can only move blobs *into* Archive, not out.

**Reference:** [Blob rehydration from the archive tier](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)

---

## 4. Soft Delete for Blobs and Containers

Soft delete provides a safety net against accidental or malicious deletion. **Blob soft delete** and **container soft delete** are separate features and must be enabled independently.

**Reference:** [Soft delete for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview) | [Soft delete for containers](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview)

### 4.1 Blob Soft Delete

- Enabled at the **storage account level** under Data Protection.
- Retention: **1 to 365 days** (default: 7 days).
- When a blob (or blob version/snapshot) is deleted, it transitions to a **soft-deleted** state instead of being permanently removed.
- Soft-deleted blobs are listed in the Portal when "Show deleted blobs" is enabled.
- Recover with `Undelete Blob` operation.
- Overwritten blobs: if versioning is enabled, the previous version is preserved automatically. If only soft delete is enabled, an overwrite creates a soft-deleted snapshot of the prior state.

```bash
# Enable blob soft delete (14-day retention)
az storage account blob-service-properties update \
  --account-name mystorageacct \
  --resource-group myRG \
  --enable-delete-retention true \
  --delete-retention-days 14
```

### 4.2 Container Soft Delete

- Also enabled at the **storage account level** under Data Protection.
- Retention: **1 to 365 days** (default: 7 days).
- When a container is deleted, it and all its blobs (including snapshots and versions) enter a soft-deleted state.
- Recover via Portal or REST API (`Restore Container`).
- A deleted container's name cannot be reused until it is permanently purged or the retention period expires.

```bash
# Enable container soft delete (30-day retention)
az storage account blob-service-properties update \
  --account-name mystorageacct \
  --resource-group myRG \
  --enable-container-delete-retention true \
  --container-delete-retention-days 30
```

### 4.3 Interaction with Versioning

| Scenario | Blob Soft Delete Only | Versioning Only | Both Enabled |
|----------|----------------------|-----------------|--------------|
| Blob deleted | Soft-deleted snapshot created | Previous version retained; current version deleted | Previous version retained + soft-deleted current version |
| Blob overwritten | Soft-deleted snapshot of previous data | New version created automatically | New version created; previous version retained |
| Recovery | Undelete restores blob | Promote previous version (copy) | Undelete + version promotion for full recovery |

> Microsoft recommends enabling **both** blob soft delete and versioning for comprehensive data protection.

---

## 5. Azure Files Snapshots and Soft Delete

### 5.1 Share Snapshots

- **Read-only, incremental** copies of the entire file share taken at a point in time.
- Created at the **share level** (not per-file).
- Limit: **200 snapshots** per share.
- Maximum retention: **10 years**.
- Store only the delta from the previous snapshot (storage-efficient).
- You can restore **individual files** from a snapshot (via Portal: browse snapshot --> select file --> Restore) or restore the **entire share**.
- Snapshots are deleted when the share is deleted -- you must delete snapshots before deleting a share (unless soft delete is active, in which case the share + snapshots enter soft-deleted state).

```bash
# Create a share snapshot
az storage share snapshot \
  --name myshare \
  --account-name mystorageacct
```

### 5.2 Soft Delete for File Shares (Recap)

- Protects against accidental **share-level** deletion (not individual file deletion).
- Enabled at the storage account level; retention 1-365 days.
- Soft-deleted shares retain all snapshots.
- Azure Backup enforces a minimum of 14 days if it detects a lower setting.

### 5.3 Combined Data Protection Strategy for Azure Files

| Threat | Protection |
|--------|-----------|
| Accidental share deletion | Soft delete for file shares |
| Accidental file deletion/corruption | Share snapshots (manual or via Azure Backup) |
| Ransomware / bulk corruption | Azure Backup with Recovery Services vault (point-in-time restore with retention policies) |

**Reference:** [Data protection overview for Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/files-data-protection-overview)

---

## 6. Blob Lifecycle Management

Lifecycle management policies automate **tiering** and **deletion** of blobs based on rules you define. This is critical for controlling costs at scale.

**Reference:** [Lifecycle management overview](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview) | [Policy structure](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)

### 6.1 Policy Structure (JSON)

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "move-to-cool-and-archive",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToCold": {
              "daysAfterModificationGreaterThan": 90
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          },
          "version": {
            "tierToCool": {
              "daysAfterCreationGreaterThan": 30
            },
            "delete": {
              "daysAfterCreationGreaterThan": 180
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["container1/logs/", "container2/archive/"],
          "blobIndexMatch": [
            {
              "name": "Project",
              "op": "==",
              "value": "Contoso"
            }
          ]
        }
      }
    }
  ]
}
```

### 6.2 Available Actions

| Action | Applies To | Description |
|--------|-----------|-------------|
| `tierToCool` | baseBlob, snapshot, version | Move to Cool tier |
| `tierToCold` | baseBlob, snapshot, version | Move to Cold tier |
| `tierToArchive` | baseBlob, snapshot, version | Move to Archive tier |
| `delete` | baseBlob, snapshot, version | Permanently delete |
| `enableAutoTierToHotFromCool` | baseBlob only | Automatically move back to Hot when accessed (requires [last access time tracking](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview#move-data-based-on-last-accessed-time)) |

> **Action priority**: if multiple actions match, lifecycle management applies the **least expensive** action. Delete < tierToArchive < tierToCold < tierToCool < tierToHot.

### 6.3 Conditions

| Condition | Applies To | Notes |
|-----------|-----------|-------|
| `daysAfterModificationGreaterThan` | baseBlob | Most common; based on `Last-Modified` property |
| `daysAfterCreationGreaterThan` | snapshot, version | Based on creation timestamp |
| `daysAfterLastAccessTimeGreaterThan` | baseBlob | Requires **last access time tracking** to be enabled on the storage account |
| `daysAfterLastTierChangeGreaterThan` | baseBlob | Useful to avoid re-tiering recently changed blobs |

### 6.4 Filters

- **blobTypes**: `blockBlob` or `appendBlob`.
- **prefixMatch**: up to 10 case-sensitive prefix strings (e.g., `container1/logs/`).
- **blobIndexMatch**: up to 10 blob index tag conditions.

### 6.5 CLI: Create a Lifecycle Policy

```bash
# Apply policy from a JSON file
az storage account management-policy create \
  --account-name mystorageacct \
  --resource-group myRG \
  --policy @lifecycle-policy.json

# View current policy
az storage account management-policy show \
  --account-name mystorageacct \
  --resource-group myRG
```

### 6.6 Important Constraints

- Lifecycle management runs **once per day** (not real-time).
- It can move blobs **into** Archive but **cannot rehydrate** from Archive.
- Policies apply to **GPv2** and **Blob Storage** accounts (not GPv1).
- Premium block blob storage accounts support lifecycle management for deletion but not tiering (only one tier exists).

---

## 7. Blob Versioning

Blob versioning automatically maintains **previous versions** of a blob whenever it is overwritten or deleted.

**Reference:** [Blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview) | [Enable and manage versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-enable)

### 7.1 How It Works

- Enabled at the **storage account level** (Data Protection settings).
- Every **write** (Put Blob, Put Block List, Copy Blob) or **delete** operation creates a new version with a unique **version ID** (a UTC timestamp, e.g., `2026-04-16T15:30:00.1234567Z`).
- The most recent write is the **current version**. Previous writes become **previous versions**.
- Each version is immutable (read-only) once superseded.

```bash
# Enable versioning
az storage account blob-service-properties update \
  --account-name mystorageacct \
  --resource-group myRG \
  --enable-versioning true
```

### 7.2 Accessing Versions

- Access a specific version by appending `?versionid=<ID>` to the blob URI.
- List versions: Portal (enable version display) or SDK (`list_blobs(include=['versions'])`).
- Current version: no version ID required in the URI.

### 7.3 Promoting a Previous Version

To restore a previous version as the current version:

1. **Copy Blob** (recommended): copy the desired previous version over the current blob. The previous version remains unchanged; a new current version is created.
2. **Portal**: select the version --> "Promote Version" from the toolbar.

> The base blob must be in an **online tier** (Hot, Cool, or Cold) to promote a version over it.

### 7.4 Interaction with Soft Delete

- When versioning and blob soft delete are both enabled, deleting a blob soft-deletes the current version and retains all previous versions.
- Undeleting restores the soft-deleted current version.
- Versions themselves can also be soft-deleted if individually removed.

### 7.5 Interaction with Lifecycle Management

- Lifecycle policies can target **versions** with `tierToCool`, `tierToCold`, `tierToArchive`, and `delete` actions using `daysAfterCreationGreaterThan`.
- This is essential for cost control: without lifecycle policies, old versions accumulate indefinitely and increase storage costs.

### 7.6 Cost Implications

- **Each version is billed as a separate object** based on its size and tier.
- Frequent overwrites on large blobs can significantly increase costs.
- Use lifecycle management to automatically tier or delete old versions.
- Consider whether **soft delete alone** (cheaper for simple recovery) is sufficient versus **full versioning** (better for audit trails and granular recovery).

---

## Exam Traps

1. **Archive is offline.** You cannot read, modify, or download an archived blob. You must rehydrate it first (Standard: up to 15 hours; High priority: <1 hour for blobs <10 GB). Expect questions testing whether you know Archive blobs are inaccessible.

2. **Lifecycle management cannot rehydrate from Archive.** Policies can move blobs *to* Archive, Cool, or Cold, but cannot move them *out of* Archive to an online tier. Rehydration requires Copy Blob or Set Blob Tier.

3. **Public access levels on containers.** "Blob" level allows anonymous read of individual blobs but not listing. "Container" level allows both. Both require public access to be allowed at the **account level** first. If the account blocks public access, container-level settings are ignored.

4. **Soft delete vs versioning -- different protections.** Soft delete protects against *deletion*. Versioning protects against *overwrites*. They are complementary, not interchangeable. Container soft delete is a separate toggle from blob soft delete.

5. **NFS shares require Premium FileStorage + no SMB.** NFS 4.1 is only available on Premium (FileStorage) accounts. An NFS share cannot coexist as SMB on the same share. No identity-based authentication for NFS -- network-level security only (private endpoints, VNet).

6. **Azure Files soft delete is share-level only.** It protects against deleting the entire share. It does **not** recover individual deleted files -- use snapshots or Azure Backup for file-level recovery.

7. **Early deletion penalties.** Deleting or re-tiering a Cool blob before 30 days, Cold before 90 days, or Archive before 180 days incurs charges for the remaining days.

8. **Account default tier is Hot or Cool only.** You cannot set Cold or Archive as the default account access tier. Those tiers must be set per-blob.

9. **Lifecycle policy action priority.** If multiple rules match, the least expensive action wins (delete is cheapest). A delete rule takes precedence over a tiering rule for the same blob.

10. **Versioning cost accumulation.** Each version is billed separately. Without lifecycle management to clean up old versions, costs can grow rapidly on frequently overwritten blobs.

---

## Sources

- [Study guide for AZ-104](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [Azure Files overview](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)
- [Plan for an Azure Files deployment](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-planning)
- [Understand Azure Files billing](https://learn.microsoft.com/en-us/azure/storage/files/understanding-billing)
- [Azure Files share snapshots](https://learn.microsoft.com/en-us/azure/storage/files/storage-snapshots-files)
- [Soft delete for Azure file shares](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-prevent-file-share-deletion)
- [Data protection overview for Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/files-data-protection-overview)
- [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Blob rehydration from the archive tier](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)
- [Lifecycle management overview](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Lifecycle management policy structure](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)
- [Configure a lifecycle management policy](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-configure)
- [Soft delete for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview)
- [Soft delete for containers](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview)
- [Blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview)
- [Enable and manage blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-enable)
- [Best practices for blob soft delete vs versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-vs-versioning-options)
