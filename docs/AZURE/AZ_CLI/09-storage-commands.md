# Azure CLI — Storage Commands

> **`az storage` across account, blob, file, queue, and table — including auth modes that trip up CI every time.**
> **Related:** [AZ-104 / 04 Storage access & security](../AZ-104/04-storage-access-and-security.md) · [AZ-104 / 05 Storage accounts](../AZ-104/05-storage-accounts.md) · [AZ-104 / 06 Azure Files and Blob](../AZ-104/06-azure-files-and-blob.md) · [14 Security & Key Vault](14-security-and-keyvault-commands.md)
> **Sources:** [`az storage`](https://learn.microsoft.com/en-us/cli/azure/storage) · [Auth options for `az storage`](https://learn.microsoft.com/en-us/azure/storage/common/authorize-data-operations-cli)

---

## Table of contents

1. [Auth modes (read this first)](#auth-modes-read-this-first)
2. [Storage accounts](#storage-accounts)
3. [Keys & SAS](#keys--sas)
4. [Blob](#blob)
5. [File shares](#file-shares)
6. [Queue](#queue)
7. [Table](#table)
8. [Lifecycle management](#lifecycle-management)
9. [Networking & firewall](#networking--firewall)

---

## Auth modes (read this first)

Every data-plane `az storage ...` command needs to authenticate. Precedence (first that resolves wins):

1. `--auth-mode login` — use current `az login` identity (Entra ID, requires **data-plane** RBAC role like `Storage Blob Data Contributor`).
2. `--account-key <key>` or env `AZURE_STORAGE_KEY`.
3. `--sas-token <token>` or env `AZURE_STORAGE_SAS_TOKEN`.
4. `--connection-string <cs>` or env `AZURE_STORAGE_CONNECTION_STRING`.
5. If `--account-name` only is provided, CLI will fetch a key via ARM (requires `Storage Account Contributor` or higher).

### Recommended in production: Entra ID

```bash
export AZURE_STORAGE_AUTH_MODE=login
az storage blob list --account-name stgdev1 --container-name logs -o table
```

- Works without shared keys — revocable via RBAC, audit-friendly.
- Requires a **data-plane RBAC role** on the account/container — [`Storage Blob Data Reader/Contributor/Owner`](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#storage), `Storage File Data SMB Share Reader/Contributor/Elevated Contributor`, `Storage Queue/Table Data Contributor`.
- *Storage Contributor* (control plane) does **not** include data-plane actions — common pitfall.

### Account key

```bash
KEY=$(az storage account keys list -g rg-dev -n stgdev1 --query "[0].value" -o tsv)
az storage blob list --account-name stgdev1 --container logs --account-key "$KEY" -o table

# Or via env
export AZURE_STORAGE_KEY="$KEY"
export AZURE_STORAGE_ACCOUNT=stgdev1
```

### SAS

```bash
END=$(date -u -d '2 hours' '+%Y-%m-%dT%H:%MZ')
SAS=$(az storage blob generate-sas \
  --account-name stgdev1 --container-name logs --name app.log \
  --permissions r --expiry "$END" --https-only --auth-mode login --as-user \
  -o tsv)

curl "https://stgdev1.blob.core.windows.net/logs/app.log?$SAS" -o app.log
```

- `--as-user` + `--auth-mode login` → **user-delegation SAS**, signed by an Entra ID key, safer than account-key SAS.

### Connection string

```bash
CS=$(az storage account show-connection-string -g rg-dev -n stgdev1 --query connectionString -o tsv)
az storage blob list --connection-string "$CS" --container logs
```

Use only when a tool demands the connection-string form.

## Storage accounts

```bash
# Create (general-purpose v2)
az storage account create \
  -g rg-dev -n stgdev1 \
  --location eastus --kind StorageV2 \
  --sku Standard_RAGRS \
  --access-tier Hot \
  --min-tls-version TLS1_2 \
  --allow-shared-key-access false \
  --allow-blob-public-access false \
  --enable-hierarchical-namespace false \
  --default-action Deny

# Inspect
az storage account list -o table
az storage account show -g rg-dev -n stgdev1
az storage account show-usage --location eastus        # subscription storage account count

# Update security properties
az storage account update -g rg-dev -n stgdev1 \
  --min-tls-version TLS1_2 --allow-shared-key-access false \
  --https-only true --allow-blob-public-access false

# ADLS Gen2 (hierarchical namespace) — must be set at create
az storage account create ... --enable-hierarchical-namespace true

# Failover (GRS/RAGRS only)
az storage account failover -g rg-dev -n stgdev1 --yes

# Delete
az storage account delete -g rg-dev -n stgdev1 --yes
```

SKU reference:

| SKU | Redundancy |
|---|---|
| `Standard_LRS` | 3 copies, single DC |
| `Standard_ZRS` | 3 copies across 3 AZs in region |
| `Standard_GRS` | LRS + async copy to paired region |
| `Standard_RAGRS` | GRS + readable secondary |
| `Standard_GZRS` | ZRS + async copy to paired region |
| `Standard_RAGZRS` | GZRS + readable secondary |
| `Premium_LRS` | SSD, low-latency, single DC |
| `Premium_ZRS` | SSD, zone-redundant |

## Keys & SAS

```bash
# Keys
az storage account keys list -g rg-dev -n stgdev1 -o table
az storage account keys renew -g rg-dev -n stgdev1 --key key1       # rotate
az storage account keys set-expiration-period -g rg-dev -n stgdev1 \
  --key-expiration-period-in-days 90

# Account-level SAS
EXP=$(date -u -d '1 day' '+%Y-%m-%dT%H:%MZ')
az storage account generate-sas \
  --account-name stgdev1 \
  --services b --resource-types sco \
  --permissions rl --expiry "$EXP" --https-only \
  --account-key "$KEY"
```

- `--services`: `b` blob, `f` file, `q` queue, `t` table.
- `--resource-types`: `s` service, `c` container/queue/table, `o` object.
- `--permissions` reads as a sequence: `r`=read, `w`=write, `d`=delete, `l`=list, `a`=add, `c`=create, `u`=update, `p`=process.

## Blob

### Containers

```bash
az storage container create \
  --account-name stgdev1 --name logs \
  --auth-mode login --public-access off

az storage container list --account-name stgdev1 --auth-mode login -o table
az storage container show --account-name stgdev1 --name logs --auth-mode login

az storage container delete --account-name stgdev1 --name logs --auth-mode login

# Immutable storage (legal hold / time-based retention)
az storage container immutability-policy create \
  --account-name stgdev1 -c archive --period 365
az storage container legal-hold set \
  --account-name stgdev1 -c archive --tags "legal-case-42"
```

### Upload & download

```bash
# Single file
az storage blob upload \
  --account-name stgdev1 --container-name logs \
  --name app.log --file ./app.log \
  --auth-mode login --overwrite

# Directory — many files in parallel
az storage blob upload-batch \
  --account-name stgdev1 --destination logs \
  --source ./logs --auth-mode login \
  --pattern "*.log" --overwrite

# Download
az storage blob download \
  --account-name stgdev1 --container-name logs \
  --name app.log --file ./app.log --auth-mode login

az storage blob download-batch \
  --account-name stgdev1 --source logs --destination ./logs \
  --auth-mode login --pattern "2024-*.log"
```

### Copy

```bash
# Server-side copy (supports across accounts/regions when both accessible)
az storage blob copy start \
  --source-uri "https://stgsrc.blob.core.windows.net/logs/app.log" \
  --destination-container logs-archive \
  --destination-blob app.log \
  --account-name stgdev1 --auth-mode login

# Sync a directory (one-way incremental)
az storage blob sync \
  --account-name stgdev1 --container logs \
  --source ./logs
```

- `blob sync` uses [AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-sync) under the hood — **AzCopy must be on PATH**.
- For very large migrations, use AzCopy directly — higher throughput, resumable.

### List / metadata / tiers

```bash
az storage blob list \
  --account-name stgdev1 --container-name logs \
  --auth-mode login --prefix "2024-" \
  --query "[].{name:name, tier:properties.blobTier, size:properties.contentLength}" \
  -o table

# Set access tier
az storage blob set-tier \
  --account-name stgdev1 --container-name logs \
  --name app.log --tier Cool --auth-mode login

# Batch tier change
az storage blob set-tier \
  --account-name stgdev1 --container-name logs \
  --name '2023-*' --tier Archive --auth-mode login

# Metadata
az storage blob metadata update \
  --account-name stgdev1 -c logs -b app.log \
  --metadata source=api env=prod --auth-mode login
```

### Versioning, soft-delete, snapshots

```bash
# Enable at account level
az storage account blob-service-properties update \
  -g rg-dev --account-name stgdev1 \
  --enable-versioning true \
  --enable-delete-retention true --delete-retention-days 30 \
  --enable-container-delete-retention true --container-delete-retention-days 30 \
  --enable-change-feed true

# Take a manual snapshot
az storage blob snapshot -c logs -b app.log --account-name stgdev1 --auth-mode login

# Undelete
az storage blob undelete -c logs -b app.log --account-name stgdev1 --auth-mode login
```

### Static website

```bash
az storage blob service-properties update \
  --account-name stgdev1 --static-website true \
  --404-document error.html --index-document index.html

# Upload a site
az storage blob upload-batch \
  --account-name stgdev1 --destination '$web' --source ./dist \
  --auth-mode login --overwrite
```

## File shares

Azure Files — SMB / NFS / REST file shares.

```bash
az storage share-rm create \
  -g rg-dev --storage-account stgdev1 \
  --name share-data \
  --quota 1024 \
  --access-tier TransactionOptimized

az storage share-rm list -g rg-dev --storage-account stgdev1 -o table
az storage share-rm show  -g rg-dev --storage-account stgdev1 --name share-data

# SMB: set up Entra-joined (Kerberos) access
az storage account update -g rg-dev -n stgdev1 \
  --enable-files-aadkerb true \
  --default-share-permission StorageFileDataSmbShareContributor

# Data-plane file ops (require auth — keys or SAS, not Entra for classic REST)
az storage file upload --account-name stgdev1 -s share-data --source ./config.ini
az storage file list --account-name stgdev1 -s share-data -o table
az storage file download --account-name stgdev1 -s share-data --path config.ini --dest ./config.ini
```

- Prefer `az storage share-rm` (control plane, ARM) over `az storage share` (data plane, requires keys).
- [NFS 4.1 shares](https://learn.microsoft.com/en-us/azure/storage/files/files-nfs-protocol) require Premium tier + `--enabled-protocols NFS`.

## Queue

```bash
az storage queue create --account-name stgdev1 --name jobs --auth-mode login
az storage queue list   --account-name stgdev1 --auth-mode login -o table

az storage message put  --account-name stgdev1 --queue-name jobs --content "process:42" --auth-mode login
az storage message peek --account-name stgdev1 --queue-name jobs --auth-mode login -o table
az storage message get  --account-name stgdev1 --queue-name jobs --num-messages 10 --auth-mode login
az storage message clear --account-name stgdev1 --queue-name jobs --auth-mode login

az storage queue delete --account-name stgdev1 --name jobs --auth-mode login
```

- Queue messages base64-encode by default when sent via CLI — set `--no-base64-encoding` if producing for a consumer that expects raw bytes.

## Table

```bash
az storage table create --account-name stgdev1 --name audit --auth-mode login
az storage table list --account-name stgdev1 --auth-mode login -o table

az storage entity insert \
  --account-name stgdev1 --table-name audit \
  --entity PartitionKey=2024 RowKey=evt-001 Action=login User=alice@contoso.com \
  --auth-mode login

az storage entity query \
  --account-name stgdev1 --table-name audit \
  --filter "PartitionKey eq '2024' and Action eq 'login'" \
  --auth-mode login -o table

az storage entity delete \
  --account-name stgdev1 --table-name audit \
  --partition-key 2024 --row-key evt-001 --auth-mode login
```

- Table API on a storage account is classic; for new work consider [Cosmos DB Table API](../COSMOSDB/01-apis-and-data-models.md) for elastic scale.

## Lifecycle management

Move cool/cold data to archive automatically.

```bash
cat > lifecycle.json <<'JSON'
{
  "rules": [
    {
      "enabled": true,
      "name": "archive-old-logs",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
            "delete":        { "daysAfterModificationGreaterThan": 2555 }
          }
        }
      }
    }
  ]
}
JSON

az storage account management-policy create \
  -g rg-dev --account-name stgdev1 \
  --policy @lifecycle.json
```

## Networking & firewall

```bash
# Default deny
az storage account update -g rg-dev -n stgdev1 --default-action Deny

# Allow an IP / CIDR
az storage account network-rule add \
  -g rg-dev --account-name stgdev1 --ip-address 203.0.113.10

# Allow a subnet
SUBNET_ID=$(az network vnet subnet show \
  -g rg-dev --vnet-name vnet-dev --name subnet-svc --query id -o tsv)
az storage account network-rule add \
  -g rg-dev --account-name stgdev1 --subnet "$SUBNET_ID"

# Private endpoint
az network private-endpoint create \
  -g rg-dev -n pe-stgdev1-blob \
  --vnet-name vnet-dev --subnet subnet-pe \
  --private-connection-resource-id $(az storage account show -g rg-dev -n stgdev1 --query id -o tsv) \
  --group-id blob \
  --connection-name stgdev1-blob

# Managed identity exception (let other Azure services reach you)
az storage account update -g rg-dev -n stgdev1 \
  --bypass AzureServices Logging Metrics
```

## Gotchas

- **`--auth-mode login` needs a data-plane RBAC role**, not the `Storage Account Contributor` control-plane one. First-time users commonly get 403.
- **Disabling `allow-shared-key-access`** breaks every script still using account keys — roll out in stages with a flight window.
- **`az storage blob upload --overwrite` defaults to `false`** — idempotent uploads require passing `--overwrite` explicitly.
- **Archive-tier blobs can't be read** until rehydrated (`az storage blob set-tier --tier Hot` + wait 1-15 hours).
- **Container names are case-sensitive and must be lowercase alphanumeric + hyphens** — the API rejects uppercase with a generic 400.
- **SAS tokens signed with an account key survive beyond the key rotation** unless you rotate both keys (`az storage account keys renew --key key1`, then `--key key2`). User-delegation SAS invalidates on the signing user's token expiry.
- **`az storage blob sync`** requires AzCopy installed separately — it's a wrapper, not a native CLI function.
- **Table entities with integer `RowKey`** are still strings under the hood; filters use `eq '123'` not `eq 123`.
- **Azure Files Entra-Kerberos** requires both `--enable-files-aadkerb true` **and** configuring a Kerberos-joined identity on the client — CLI only handles the server side.

## Sources

- [`az storage`](https://learn.microsoft.com/en-us/cli/azure/storage) · [`az storage account`](https://learn.microsoft.com/en-us/cli/azure/storage/account)
- [Authorize with the CLI](https://learn.microsoft.com/en-us/azure/storage/common/authorize-data-operations-cli)
- [Storage redundancy options](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Blob access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [SAS overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Azure Files](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)
