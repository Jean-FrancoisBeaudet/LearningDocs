# Configure and Manage Storage Accounts

> **Exam mapping:** AZ-104 -- "Implement and manage storage" (15-20%)
> **One-liner:** Storage accounts are the foundational container for all Azure Storage services (Blob, File, Queue, Table); mastering account types, redundancy, encryption, and data-movement tools is essential for the exam.
> **Related:** [04-storage-access (placeholder)](./04-storage-access.md) | [06-azure-files-and-blob (placeholder)](./06-azure-files-and-blob.md) | [AZURE/AZ-305/02-data-storage.md](../AZ-305/02-data-storage.md)

---

## Table of Contents

1. [Create and Configure Storage Accounts](#1-create-and-configure-storage-accounts)
2. [Configure Azure Storage Redundancy](#2-configure-azure-storage-redundancy)
3. [Configure Object Replication](#3-configure-object-replication)
4. [Configure Storage Account Encryption](#4-configure-storage-account-encryption)
5. [Azure Storage Explorer and AzCopy](#5-azure-storage-explorer-and-azcopy)
6. [Exam Traps](#exam-traps)
7. [Sources](#sources)

---

## 1. Create and Configure Storage Accounts

[Microsoft Learn -- Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)

A **storage account** provides a unique namespace in Azure for your data. Every object you store has an address that includes your unique account name. The combination of account name and the service endpoint forms the endpoints for your storage account.

### Account Types

| Account type | Supported services | Performance | Use case |
|---|---|---|---|
| **Standard general-purpose v2** | Blob (incl. Data Lake), Queue, Table, Azure Files | Standard (HDD-backed) | Default choice for most scenarios; supports all access tiers (Hot, Cool, Cold, Archive). |
| **Premium block blobs** | Block blobs and append blobs only | Premium (SSD-backed) | Low-latency workloads with high transaction rates (IoT ingestion, analytics). |
| **Premium file shares** | Azure Files only | Premium (SSD-backed) | Enterprise file shares that need IOPS-intensive or latency-sensitive performance. |
| **Premium page blobs** | Page blobs only | Premium (SSD-backed) | Unmanaged VM disks and custom scenarios requiring random read/write on page blobs. |

> **Exam note:** Legacy account types (Storage v1, BlobStorage) still exist but Microsoft recommends upgrading to **general-purpose v2**. The exam rarely tests v1 directly but may present it as a distractor.

### Naming Rules

- **3 to 24 characters** in length.
- Only **lowercase letters and numbers** -- no hyphens, underscores, or uppercase.
- Must be **globally unique** across all of Azure (the name becomes part of the DNS endpoint).

### Settings at Creation Time

| Setting | Notes |
|---|---|
| **Region** | Choose the region closest to consumers. Region determines available redundancy options (not all regions support ZRS/GZRS). |
| **Performance** | Standard (HDD) or Premium (SSD). Cannot be changed after creation -- you must create a new account and migrate. |
| **Redundancy** | LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS (see [Section 2](#2-configure-azure-storage-redundancy)). Can be changed later (with constraints). |
| **Access tier (default)** | Hot or Cool. Sets the default for blobs that don't have an explicit tier. Cold and Archive are set per-blob, not at account level. |
| **Networking** | Public endpoint (all networks), public endpoint (selected virtual networks), or private endpoint. |
| **Data protection** | Soft delete for blobs/containers/file shares, versioning, change feed, point-in-time restore. |
| **Encryption** | Microsoft-managed keys (default) or customer-managed keys (see [Section 4](#4-configure-storage-account-encryption)). Infrastructure encryption (double encryption) opt-in at creation. |

### CLI: Create a Storage Account

```bash
# Create a standard general-purpose v2 account with LRS
az storage account create \
  --name mystorageacct2026 \
  --resource-group rg-storage-demo \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

# Create a premium block-blob account
az storage account create \
  --name mypremiumblobs \
  --resource-group rg-storage-demo \
  --location eastus \
  --sku Premium_LRS \
  --kind BlockBlobStorage
```

**SKU values map to redundancy:** `Standard_LRS`, `Standard_ZRS`, `Standard_GRS`, `Standard_RAGRS`, `Standard_GZRS`, `Standard_RAGZRS`, `Premium_LRS`, `Premium_ZRS`.

> Premium accounts only support **LRS** and **ZRS** -- geo-redundant options (GRS, GZRS, RA-*) are not available for Premium.

---

## 2. Configure Azure Storage Redundancy

[Microsoft Learn -- Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)

Azure Storage always stores **multiple copies** of your data to protect against planned and unplanned events. Redundancy is the single biggest factor in durability and availability guarantees.

### The Six Redundancy Options

#### Local Redundancy -- Primary Region Only

| Option | How it works |
|---|---|
| **LRS (Locally Redundant Storage)** | Stores **3 copies** within a **single datacenter** (single availability zone) in the primary region. Protects against drive/server/rack failures. Does **not** protect against datacenter-level disasters (fire, flood). |
| **ZRS (Zone-Redundant Storage)** | Stores **3 copies** across **3 separate availability zones** in the primary region. Protects against zone-level outages. Data is accessible for read and write even if one zone goes down. |

#### Geo-Redundancy -- Primary + Secondary Region

| Option | How it works |
|---|---|
| **GRS (Geo-Redundant Storage)** | LRS in the primary region + **asynchronous** copy to a **paired region** where the data is stored with LRS (3 copies). Secondary data is **not readable** until a failover occurs. |
| **RA-GRS (Read-Access Geo-Redundant Storage)** | Same as GRS but the secondary endpoint is **always readable** (read-only). Access via `<accountname>-secondary.blob.core.windows.net`. |
| **GZRS (Geo-Zone-Redundant Storage)** | ZRS in the primary region (3 AZs) + asynchronous copy to a paired region stored with LRS. Combines zone protection in primary with geo protection. |
| **RA-GZRS (Read-Access Geo-Zone-Redundant Storage)** | Same as GZRS but the secondary endpoint is **always readable** (read-only). **Maximum protection** option. |

### Redundancy Comparison Table

| Redundancy | Copies | Durability (annual) | Write availability SLA | Read availability SLA | Read from secondary | Failover supported |
|---|---|---|---|---|---|---|
| **LRS** | 3 (1 DC) | 11 nines (99.999999999%) | 99.9% | 99.9% | No | N/A |
| **ZRS** | 3 (3 AZs) | 12 nines (99.9999999999%) | 99.9% | 99.9% | No | N/A |
| **GRS** | 6 (3 primary + 3 secondary) | 16 nines (99.99999999999999%) | 99.9% | 99.9% | No (until failover) | Yes |
| **RA-GRS** | 6 | 16 nines | 99.9% | 99.99% | **Yes** | Yes |
| **GZRS** | 6 (3 AZs + 3 secondary) | 16 nines | 99.9% | 99.9% | No (until failover) | Yes |
| **RA-GZRS** | 6 | 16 nines | 99.9% | 99.99% | **Yes** | Yes |

> **Exam note on SLA:** Read availability SLA is higher for RA-* variants (99.99% vs 99.9%) because the app can fall back to the secondary endpoint.

### Failover

[Microsoft Learn -- Storage account failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)

- **Customer-managed failover:** You initiate failover from the portal, CLI (`az storage account failover`), or PowerShell. The secondary region becomes the new primary. After failover, the account is **downgraded to LRS** in the new primary region -- you must reconfigure geo-redundancy afterward.
- **Microsoft-managed failover:** Microsoft initiates failover in extreme disaster scenarios. This is a last resort and you should not depend on it for your DR plan.
- **RPO:** Geo-replication is asynchronous; there is potential for data loss. You can check the `Last Sync Time` property to assess the RPO window.

### Changing Redundancy

[Microsoft Learn -- Change redundancy](https://learn.microsoft.com/en-us/azure/storage/common/redundancy-migration)

Some changes are live migrations (no downtime); others require a manual migration (create a new account and copy data). Key constraints:

| From | To | Method |
|---|---|---|
| LRS | ZRS | Live migration (request via support) or manual |
| LRS | GRS/RA-GRS | Portal toggle (live) |
| GRS | GZRS | Must go GRS -> ZRS first, then ZRS -> GZRS |
| Premium LRS | Premium ZRS | Supported |
| Any geo | LRS or ZRS | Portal toggle |

---

## 3. Configure Object Replication

[Microsoft Learn -- Object replication overview](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-overview) | [Configure object replication](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-configure)

Object replication **asynchronously copies block blobs** from a source container to a destination container, which may be in **different storage accounts, different regions, or different subscriptions**.

### Requirements

| Requirement | Source account | Destination account |
|---|---|---|
| **Blob versioning** | Must be enabled | Must be enabled |
| **Change feed** | Must be enabled | Not required |
| **Account type** | General-purpose v2 or Premium block blobs | General-purpose v2 or Premium block blobs |
| **Access tier** | Cannot be Archive | Cannot be Archive |
| **Hierarchical namespace (ADLS Gen2)** | Not supported | Not supported |
| **Customer-provided encryption keys** | Not supported | -- |
| **Customer-managed failover** | Not supported on either account in a replication policy | -- |

### How It Works

1. You create a **replication policy** on the destination account that specifies the source account.
2. Each policy contains one or more **rules**, each mapping a source container to a destination container.
3. You can filter which blobs replicate using **prefix filters** (e.g., only blobs matching `logs/`).
4. Replication is **asynchronous** -- there is no guaranteed RPO/RTO. You can check per-blob replication status.
5. **Deletions are not replicated** -- deleting a blob in the source does not delete it from the destination.

### Use Cases

- **Latency reduction:** Replicate blobs closer to consumers in different regions.
- **Analytics efficiency:** Process data in a secondary region without impacting production storage.
- **Data distribution:** Share data across teams in different subscriptions.
- **Disaster preparation:** Maintain a separate copy of critical blobs beyond geo-redundancy.

### Cross-Tenant Replication

Starting **December 15, 2023**, cross-tenant replication is **disabled by default** for new storage accounts. The `AllowCrossTenantReplication` property must be explicitly set to `true` to allow it. For security, Microsoft recommends keeping it disabled unless required.

### CLI: Configure Object Replication

```bash
# Enable versioning and change feed on the source account
az storage account blob-service-properties update \
  --account-name srcaccount \
  --resource-group rg-source \
  --enable-versioning true \
  --enable-change-feed true

# Enable versioning on the destination account
az storage account blob-service-properties update \
  --account-name destaccount \
  --resource-group rg-dest \
  --enable-versioning true

# Create a replication policy (JSON file defines rules)
az storage account or-policy create \
  --account-name destaccount \
  --resource-group rg-dest \
  --source-account srcaccount \
  --destination-account destaccount \
  --source-container src-container \
  --destination-container dest-container \
  --min-creation-time "2026-01-01T00:00:00Z"
```

---

## 4. Configure Storage Account Encryption

[Microsoft Learn -- Azure Storage encryption for data at rest](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption) | [Customer-managed keys overview](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)

### Storage Service Encryption (SSE)

- **Always on** -- SSE cannot be disabled. All data written to Azure Storage is automatically encrypted at rest using **256-bit AES** encryption.
- **Transparent** -- encryption/decryption is handled by the platform; no application code changes needed.
- All services (Blob, File, Queue, Table) and all tiers (Hot, Cool, Cold, Archive) are encrypted.

### Encryption Key Options

| Key type | Description | Key storage | Scope |
|---|---|---|---|
| **Microsoft-managed keys (MMK)** | Default. Microsoft generates, stores, and rotates the keys. | Microsoft-managed | Entire account |
| **Customer-managed keys (CMK)** | You create and manage the key in **Azure Key Vault** or **Key Vault Managed HSM**. You control rotation and access. | Azure Key Vault (or Managed HSM) | Entire account or per-scope |
| **Customer-provided keys** | Provide a key on each Blob service request. The key is **not stored** by Azure. | Your application | Per-request (Blob only) |

### Customer-Managed Keys -- Key Requirements

[Microsoft Learn -- Configure CMK for existing account](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-configure-existing-account)

1. **Key Vault must have soft-delete enabled** (enabled by default on new vaults).
2. **Key Vault must have purge protection enabled** -- this is a hard requirement.
3. The storage account's **managed identity** (system-assigned or user-assigned) must have the `Key Vault Crypto Service Encryption User` role (or equivalent Key Vault access policy: Get, Wrap Key, Unwrap Key).
4. Key Vault and storage account must be in the **same region** (but can be in different subscriptions within the same tenant).
5. For **Queue and Table services**, you must create the account with an account-level encryption key scope if you want CMK (infrastructure-level requirement at account creation).

### Encryption Scopes

[Microsoft Learn -- Encryption scopes](https://learn.microsoft.com/en-us/azure/storage/blobs/encryption-scope-manage)

Encryption scopes allow you to manage encryption at the **container or individual blob level**:

- Each scope can use either Microsoft-managed or customer-managed keys.
- You can assign a default encryption scope to a container so all blobs within it inherit that scope.
- You can require that all blobs in a container use the container's default scope (prevent overrides).
- Use cases: multi-tenant applications where each tenant's data is encrypted with a separate key.

### Infrastructure Encryption (Double Encryption)

- When enabled, data is encrypted **twice**: once at the **service level** and once at the **infrastructure level**, using two different algorithms and two different keys.
- The infrastructure-level key is always **Microsoft-managed** (even if the service-level key is CMK).
- Must be enabled **at storage account creation time** -- cannot be added later.
- Meets compliance requirements for double encryption (e.g., certain government/financial workloads).

### CLI: Configure Encryption

```bash
# Enable CMK on an existing storage account (user-assigned identity)
az storage account update \
  --name mystorageacct2026 \
  --resource-group rg-storage-demo \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "https://mykeyvault.vault.azure.net" \
  --encryption-key-name my-storage-key \
  --encryption-key-version "" \
  --identity-type UserAssigned \
  --user-identity-id "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{name}"

# Create an account with infrastructure encryption enabled (double encryption)
az storage account create \
  --name mydoubleencacct \
  --resource-group rg-storage-demo \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --require-infrastructure-encryption true

# Revert to Microsoft-managed keys
az storage account update \
  --name mystorageacct2026 \
  --resource-group rg-storage-demo \
  --encryption-key-source Microsoft.Storage
```

---

## 5. Azure Storage Explorer and AzCopy

### Azure Storage Explorer

[Microsoft Learn -- Azure Storage Explorer](https://learn.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer)

Storage Explorer is a **free, cross-platform GUI application** (Windows, macOS, Linux) for managing Azure Storage data.

**Capabilities:**

- Browse and manage **Blobs, Files, Queues, Tables**, and **Azure Data Lake Storage**.
- Upload, download, copy, move, and delete blobs and files.
- Manage blob access tiers, snapshots, leases, and metadata.
- Create and manage containers, file shares, queues, and tables.
- Generate and manage **Shared Access Signatures (SAS)**.
- View and edit blob properties and metadata.

**Authentication methods:**

| Method | Description |
|---|---|
| **Microsoft Entra ID (sign-in)** | Full RBAC-based access. Recommended for corporate environments. |
| **Shared Access Signature (SAS)** | Scoped, time-limited access. Good for granting temporary access without sharing keys. |
| **Storage account access key** | Full control. Use cautiously -- equivalent to root. |
| **Local emulator** | Connect to Azurite (local storage emulator) for dev/test. |

### AzCopy

[Microsoft Learn -- Get started with AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10)

AzCopy is a **command-line utility** optimized for high-performance data transfer to and from Azure Storage. It is the recommended tool for large-scale or scripted data movement.

**Authentication methods:**

| Method | Command |
|---|---|
| **Microsoft Entra ID** | `azcopy login` (opens browser for interactive auth) or `azcopy login --service-principal` |
| **SAS token** | Append `?sv=...` to the blob/container URL |

**Core commands:**

```bash
# Copy a local file to a blob container
azcopy copy "./data/report.csv" "https://mystorageacct.blob.core.windows.net/uploads/report.csv?<SAS>"

# Copy an entire local directory to a container (recursive)
azcopy copy "./data/" "https://mystorageacct.blob.core.windows.net/uploads/?<SAS>" --recursive

# Copy between two storage accounts (server-to-server, no local download)
azcopy copy "https://source.blob.core.windows.net/data/?<SAS>" \
            "https://dest.blob.core.windows.net/data/?<SAS>" --recursive

# Sync a local directory to a container (one-direction: source -> destination)
azcopy sync "./data/" "https://mystorageacct.blob.core.windows.net/uploads/?<SAS>"

# Sync a container to local (reverse direction)
azcopy sync "https://mystorageacct.blob.core.windows.net/uploads/?<SAS>" "./data/"

# Create a new container
azcopy make "https://mystorageacct.blob.core.windows.net/newcontainer?<SAS>"

# Login with Entra ID (interactive)
azcopy login

# Login with service principal (non-interactive, for CI/CD)
azcopy login --service-principal --application-id <app-id> --tenant-id <tenant-id>

# List jobs
azcopy jobs list

# Resume a failed job
azcopy jobs resume <job-id>

# Benchmark upload performance
azcopy benchmark "https://mystorageacct.blob.core.windows.net/benchcontainer?<SAS>"
```

### Storage Explorer vs AzCopy -- When to Use Each

| Scenario | Tool | Why |
|---|---|---|
| Browse storage visually, inspect metadata, one-off file management | **Storage Explorer** | GUI is faster for ad-hoc tasks. |
| Bulk copy thousands/millions of files | **AzCopy** | CLI performance is far superior; supports parallelism. |
| Scripted/automated transfers (CI/CD, scheduled jobs) | **AzCopy** | Command-line integrates into scripts and pipelines. |
| Sync a local folder with a container on a schedule | **AzCopy** (`sync`) | One-command incremental sync. |
| Server-to-server copy (no local machine in the path) | **AzCopy** | Data moves directly between storage endpoints. |
| Quick SAS generation or queue/table inspection | **Storage Explorer** | SAS wizard and table/queue editors are GUI-only features. |
| Grant temporary access to a non-technical user | **Storage Explorer** (with SAS) | No CLI knowledge required. |

> **Note:** Storage Explorer uses AzCopy under the hood for its upload/download operations.

---

## Exam Traps

1. **Performance tier is permanent.** You cannot convert a Standard account to Premium (or vice versa). You must create a new account and migrate data.

2. **Premium accounts only support LRS and ZRS.** GRS, RA-GRS, GZRS, and RA-GZRS are **not available** for any Premium account type.

3. **Premium block blobs do not support page blobs** (and vice versa). Know which account type supports which blob type -- do not confuse them.

4. **After a customer-managed failover, the account becomes LRS.** You must manually reconfigure geo-redundancy. Microsoft-managed failover does not have this limitation (but you cannot control when it happens).

5. **RA-GRS/RA-GZRS secondary is read-only.** You cannot write to the secondary endpoint. Read availability SLA is 99.99%, not 99.9%.

6. **CMK requires soft-delete AND purge protection on Key Vault.** Missing purge protection is a common exam trap. Both must be enabled before you can configure CMK.

7. **Infrastructure encryption (double encryption) must be enabled at account creation.** You cannot add it later. If the question says "enable double encryption on an existing account," the answer is "create a new account."

8. **Object replication requires versioning on BOTH accounts** and change feed on the **source** only. Replication does not copy deletions.

9. **Object replication does not support accounts with hierarchical namespace** (ADLS Gen2 with HNS enabled) or customer-managed failover accounts.

10. **AzCopy sync is one-direction only.** It compares source to destination and makes the destination match the source. It does **not** do bidirectional sync. Deleting a file at the destination is optional (`--delete-destination=true`).

11. **AzCopy copy vs sync:** `copy` always transfers all files. `sync` only transfers files that are new or modified (based on last-modified timestamp). The exam may present scenarios where `sync` is the correct answer for incremental transfers.

12. **Account naming: 3-24 chars, lowercase + numbers only.** Hyphens and underscores are **not allowed** (unlike resource group names which do allow them).

13. **Cross-tenant object replication is disabled by default** (since Dec 2023). If the question involves replicating across tenants, look for the `AllowCrossTenantReplication` property.

14. **Queue and Table CMK:** To use customer-managed keys for Queue and Table services, you must create the account with an **account-level encryption key scope** at creation time. This cannot be changed afterward.

---

## Sources

- [Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Storage account failover and disaster recovery](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)
- [Change how a storage account is replicated](https://learn.microsoft.com/en-us/azure/storage/common/redundancy-migration)
- [Object replication overview](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-overview)
- [Configure object replication](https://learn.microsoft.com/en-us/azure/storage/blobs/object-replication-configure)
- [Azure Storage encryption for data at rest](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)
- [Customer-managed keys for Azure Storage encryption](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview)
- [Configure CMK for existing storage account](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-configure-existing-account)
- [Encryption scopes for Blob storage](https://learn.microsoft.com/en-us/azure/storage/blobs/encryption-scope-manage)
- [Get started with AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10)
- [AzCopy login reference](https://learn.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-login)
- [AzCopy copy reference](https://learn.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-copy)
- [Azure Storage Explorer](https://learn.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104: Implement and manage storage (training path)](https://learn.microsoft.com/en-us/training/paths/az-104-manage-storage/)
