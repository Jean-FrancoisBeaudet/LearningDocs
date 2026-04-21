# Configure Access to Storage

> **Exam mapping:** AZ-104 → "Implement and manage storage" (15–20%)
> **One-liner:** Lock down *who* and *how* data is reached — firewalls, SAS tokens, access keys, stored policies, and identity-based SMB access are the five control planes the exam tests.
> **Related:** [05-storage-accounts.md](05-storage-accounts.md) | [06-azure-files-and-blob.md](06-azure-files-and-blob.md) | [AZ-305 / 02-data-storage.md](../AZ-305/02-data-storage.md)

---

## 1. Storage Firewalls and Virtual Networks

[Microsoft Learn — Configure Azure Storage firewalls and virtual networks](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security)

### Default behaviour

By default every storage account has its **public endpoint** open to **all networks**. The first hardening step is switching the default rule to **Deny** and then explicitly whitelisting allowed sources.

### Configuring deny-by-default + allowed VNets

1. **Enable a service endpoint** on the target subnet (`Microsoft.Storage`).
2. Add a **virtual network rule** on the storage account pointing to that VNet/subnet.
3. Set the default action to **Deny**.

```bash
# Enable service endpoint on a subnet
az network vnet subnet update \
  --resource-group rg-storage \
  --vnet-name vnet-main \
  --name snet-app \
  --service-endpoints Microsoft.Storage

# Add the VNet rule to the storage account
az storage account network-rule add \
  --resource-group rg-storage \
  --account-name stcontoso \
  --vnet-name vnet-main \
  --subnet snet-app

# Set default action to Deny
az storage account update \
  --resource-group rg-storage \
  --name stcontoso \
  --default-action Deny
```

### Allowed IP ranges

You can add individual public IPs or CIDR ranges (IPv4 only, no private ranges like 10.x or 172.16.x):

```bash
az storage account network-rule add \
  --resource-group rg-storage \
  --account-name stcontoso \
  --ip-address 203.0.113.0/24
```

### Trusted Microsoft services exception

Some Azure services (Backup, Site Recovery, Event Grid, Monitor, Microsoft Entra ID, etc.) operate from unpredictable IPs. The **"Allow trusted Microsoft services to access this storage account"** exception lets them bypass firewall rules. This is enabled by default and should almost always stay enabled.

```bash
az storage account update \
  --resource-group rg-storage \
  --name stcontoso \
  --bypass AzureServices
```

### Service endpoints vs Private Endpoints

| Aspect | Service Endpoint | Private Endpoint |
|---|---|---|
| **Traffic path** | Optimised route over Microsoft backbone, but still uses the **public endpoint** IP of the storage account | Traffic flows over **Azure Private Link** to a **private IP** inside your VNet |
| **DNS** | No DNS change; public IP is used | Private DNS zone maps `<account>.blob.core.windows.net` → private IP |
| **Cross-region** | Same region only (endpoint policies can enforce this) | Works **cross-region** and from on-premises via VPN/ExpressRoute |
| **Firewall interaction** | You still need a VNet rule on the storage firewall | Firewall rules **do not apply** — traffic never hits the public endpoint |
| **Cost** | Free | Per-hour + per-GB data-processed charge |

> **Exam note:** When a Private Endpoint is configured, the storage firewall only governs the *public* endpoint. Private Endpoint traffic bypasses the firewall entirely. ([Learn more](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints))

---

## 2. Shared Access Signatures (SAS)

[Microsoft Learn — Grant limited access to data with SAS](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)

A SAS is a signed URI that grants **constrained, time-limited** access to storage resources without exposing the account key.

### Three types of SAS

| Type | Signed with | Scope | Stored access policy support | Security ranking |
|---|---|---|---|---|
| **User delegation SAS** | Microsoft Entra ID credentials (OAuth token → user delegation key) | Blob Storage, Queue Storage, Table Storage, Azure Files | No (ad hoc only) | Best — no account key involved |
| **Service SAS** | Storage account key | Single service (blobs **or** queues **or** tables **or** files) | Yes | Medium |
| **Account SAS** | Storage account key | One or more services, service-level operations (e.g., `Get Blob Service Properties`) | No (ad hoc only) | Lowest — broadest scope + key-based |

> **Microsoft recommends user delegation SAS whenever possible** because it avoids storing or distributing the account key and leverages Entra ID RBAC + conditional access.

### SAS token components

A SAS token (appended as query parameters) typically includes:

| Parameter | Meaning | Example |
|---|---|---|
| `sv` | Signed storage service version | `2024-11-04` |
| `ss` | Signed services (account SAS) | `bfqt` (blob, file, queue, table) |
| `srt` | Signed resource types (account SAS) | `sco` (service, container, object) |
| `sp` | Signed permissions | `rwdlacup` |
| `st` | Start time (UTC) | `2026-04-16T00:00:00Z` |
| `se` | Expiry time (UTC) | `2026-04-17T00:00:00Z` |
| `sip` | Allowed IP range | `203.0.113.0-203.0.113.255` |
| `spr` | Allowed protocol | `https` |
| `sig` | Signature (HMAC-SHA256 over the string-to-sign) | Base64 string |

### SAS URL format

```
https://<account>.blob.core.windows.net/<container>/<blob>?sv=2024-11-04&st=...&se=...&sp=r&sig=<signature>
```

### Generating SAS tokens — CLI

```bash
# User delegation SAS for a blob (preferred)
az storage blob generate-sas \
  --account-name stcontoso \
  --container-name data \
  --name report.csv \
  --permissions r \
  --expiry 2026-04-17T00:00:00Z \
  --auth-mode login \
  --as-user \
  --output tsv

# Account SAS (key-based)
az storage account generate-sas \
  --account-name stcontoso \
  --services b \
  --resource-types sco \
  --permissions rwl \
  --expiry 2026-04-17T00:00:00Z \
  --output tsv
```

The `--as-user` flag combined with `--auth-mode login` tells the CLI to request a **user delegation key** from Entra ID instead of using the account key.

---

## 3. Stored Access Policies

[Microsoft Learn — Define a stored access policy](https://learn.microsoft.com/en-us/rest/api/storageservices/define-stored-access-policy)

A stored access policy is a **named, server-side policy** attached to a container, queue, table, or file share. A **service SAS** can reference the policy by name instead of embedding all constraints in the token itself.

### Why use them?

- **Revocability:** Delete or modify the policy → every SAS that references it is immediately invalidated. Without a stored access policy, the only way to revoke a service SAS is to **rotate the account key** (which breaks everything signed with that key).
- **Centralized control:** Change start/expiry/permissions in one place.

### Limits

- **Maximum 5 stored access policies** per container (or queue, table, share).
- Stored access policies are **not supported** for user delegation SAS or account SAS — only service SAS.

### CLI operations

```bash
# Create a stored access policy on a blob container
az storage container policy create \
  --account-name stcontoso \
  --container-name data \
  --name read-policy \
  --permissions rl \
  --start 2026-04-16T00:00:00Z \
  --expiry 2026-07-16T00:00:00Z

# List policies on a container
az storage container policy list \
  --account-name stcontoso \
  --container-name data

# Generate a service SAS linked to the stored access policy
az storage blob generate-sas \
  --account-name stcontoso \
  --container-name data \
  --name report.csv \
  --policy-name read-policy \
  --output tsv

# Revoke: delete the policy (invalidates all SAS referencing it)
az storage container policy delete \
  --account-name stcontoso \
  --container-name data \
  --name read-policy
```

> **Propagation delay:** Creating or updating a stored access policy can take **up to 30 seconds** to take effect. During this window, requests using SAS tokens linked to the policy may return **403 Forbidden**.

---

## 4. Access Keys

[Microsoft Learn — Manage storage account access keys](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage)

### Basics

- Every storage account has **two 512-bit access keys** (`key1` and `key2`).
- Either key grants **full, unrestricted** access to the entire storage account (equivalent to root).
- Two keys exist so you can **rotate without downtime**.

### Rotation strategy

The standard pattern:

1. Update all apps/connection strings to use **key2**.
2. **Regenerate key1** (`az storage account keys renew --key key1`).
3. Update all apps/connection strings to use the new **key1**.
4. **Regenerate key2** (`az storage account keys renew --key key2`).

```bash
# List current keys
az storage account keys list \
  --resource-group rg-storage \
  --account-name stcontoso \
  --output table

# Regenerate key1
az storage account keys renew \
  --resource-group rg-storage \
  --account-name stcontoso \
  --key key1
```

> **Impact of rotation:** Regenerating a key **immediately invalidates** every SAS (service SAS or account SAS) that was signed with that key. Plan accordingly.

### Key Vault integration for automatic rotation

Rather than manual rotation, store access keys in **Azure Key Vault** and configure automatic rotation using the [dual-credential rotation pattern](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation-dual):

1. Store the key as a Key Vault **secret** with a defined expiration (e.g., 90 days).
2. Key Vault publishes a **Near Expiry** event to **Event Grid** 30 days before expiry.
3. An **Azure Function** (triggered by Event Grid) calls the storage API to regenerate the alternate key, then updates the Key Vault secret with the new value.

This removes human error from the rotation cycle and is the Microsoft-recommended approach for production.

---

## 5. Identity-Based Access for Azure Files

[Microsoft Learn — Azure Files identity-based authentication overview](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-active-directory-overview)

Azure Files supports **identity-based authentication over SMB** so that users mount shares with their domain credentials rather than a storage account key.

### Supported identity sources (pick one per account)

| Source | Use case | Requirements |
|---|---|---|
| **Microsoft Entra Domain Services** | Cloud-only or hybrid orgs already using Entra DS | Entra DS deployed in the same subscription; users synced to Entra ID |
| **On-premises AD DS** | Hybrid orgs with existing AD | Domain-joined machines; AD DS synced to Entra ID via Entra Connect Sync or Cloud Sync; storage account joined to AD |
| **Microsoft Entra Kerberos** (for hybrid identities) | Hybrid users accessing from Entra-joined or hybrid-joined devices | Entra ID Kerberos enabled; no domain controller needed for cloud-joined devices |

### Two-tier permission model

1. **Share-level permissions** — assigned via Azure RBAC roles on the file share resource.
2. **Directory / file-level permissions** — standard **NTFS ACLs** configured from a domain-joined Windows machine using `icacls` or Windows Explorer.

Both tiers are enforced: RBAC controls *whether* you can connect; NTFS ACLs control *what* you can see and modify once connected.

### Built-in RBAC roles for Azure Files SMB

| Role | Permissions |
|---|---|
| **Storage File Data SMB Share Reader** | Read access to files and directories |
| **Storage File Data SMB Share Contributor** | Read, write, delete access to files and directories |
| **Storage File Data SMB Share Elevated Contributor** | Read, write, delete **+ modify NTFS ACLs** |

```bash
# Assign share-level RBAC role
az role assignment create \
  --assignee user@contoso.com \
  --role "Storage File Data SMB Share Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/stcontoso/fileServices/default/fileshares/share01"
```

### Enabling identity-based access on the storage account

```bash
# Enable Entra Domain Services authentication
az storage account update \
  --resource-group rg-storage \
  --name stcontoso \
  --azure-files-identity-based-authentication \
    directoryServiceOptions=AADDS

# Enable on-premises AD DS authentication (requires additional domain-join steps)
az storage account update \
  --resource-group rg-storage \
  --name stcontoso \
  --azure-files-identity-based-authentication \
    directoryServiceOptions=AD
```

> **2026 note:** A Windows Server update scheduled for July 2026 changes the default Kerberos encryption type from RC4 to AES-256. Storage accounts not upgraded to support AES-256 may experience mount failures for SMB shares. Plan to verify encryption settings before that date. ([Learn more](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable))

---

## Exam Traps

| Trap | What to remember |
|---|---|
| **User delegation SAS vs service SAS** | User delegation SAS is signed with Entra ID credentials (no key exposure) and is the **recommended** type. Service SAS uses the account key. The exam loves asking which is "more secure." |
| **Stored access policy limits** | Max **5 per container** (or queue/table/share). Only works with **service SAS** — not user delegation or account SAS. |
| **Stored access policy propagation** | Up to **30 seconds** for changes to take effect. During that window, linked SAS tokens may return 403. |
| **Service endpoint vs private endpoint** | Service endpoints still use the **public IP** of the storage account (just an optimised route). Private endpoints give a **private IP** inside the VNet. Firewall rules **do not apply** to private endpoint traffic. |
| **Private endpoints and cross-region** | Service endpoints are **same-region only**; private endpoints work **cross-region** and from on-premises. |
| **Access key rotation invalidates SAS** | Regenerating a key breaks every SAS signed with it. Always rotate the key that apps are **not** currently using. |
| **Trusted Microsoft services** | The bypass exception must be enabled for services like Azure Backup, Event Grid, and Monitor to reach storage behind a firewall. |
| **Azure Files identity source** | Only **one** identity source per storage account (Entra DS *or* on-prem AD DS *or* Entra Kerberos). You cannot mix them. |
| **Two-tier Azure Files permissions** | Share-level RBAC alone is not enough — NTFS ACLs must also be configured for directory/file-level control. Both are enforced. |
| **SAS best practice** | Always use **HTTPS-only** (`spr=https`), shortest possible expiry, and least-privilege permissions. |

---

## Sources

- [Configure Azure Storage firewalls and virtual networks](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security)
- [Virtual network service endpoints overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)
- [Use private endpoints for Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
- [Grant limited access to data with shared access signatures (SAS)](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Create a user delegation SAS](https://learn.microsoft.com/en-us/rest/api/storageservices/create-user-delegation-sas)
- [Define a stored access policy](https://learn.microsoft.com/en-us/rest/api/storageservices/define-stored-access-policy)
- [Manage storage account access keys](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage)
- [Key Vault dual-credential rotation tutorial](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation-dual)
- [Azure Files identity-based authentication overview](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-active-directory-overview)
- [Assign share-level permissions for Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-assign-share-level-permissions)
- [Microsoft Entra Kerberos authentication for Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable)
- [On-premises AD DS authentication for Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-identity-ad-ds-overview)
