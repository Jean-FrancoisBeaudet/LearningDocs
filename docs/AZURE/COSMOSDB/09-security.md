# Security & Identity

> **Exam mapping:** AZ-204 *(Managed Identity, RBAC)* · AZ-305 *(network isolation, CMK)* · AI-102 *(Key Vault for AI secrets)*
> **One-liner:** Defaults are secure-ish; the exam-worthy answer is always **Managed Identity + data-plane RBAC + Private Endpoint + CMK**.
> **Related:** [APIs](01-apis-and-data-models.md) · [Global distribution & backup](05-global-distribution-and-ha.md)

## Authentication options

| Method | Surface | Verdict |
|--------|---------|---------|
| Primary / secondary **account keys** | Full admin | ❌ avoid for apps; OK for emergency / ops |
| **Resource tokens** (user + permission docs) | Fine-grained, time-limited | Legacy; largely replaced by RBAC |
| **Entra ID RBAC — control plane** | ARM-level (create/delete/configure) | Standard |
| **Entra ID RBAC — data plane** | Read/write items in containers | ✅ use this; works with Managed Identity |

Disable keys entirely once RBAC is rolled out:

```bash
az resource update --resource-type Microsoft.DocumentDB/databaseAccounts \
  --name $acct --resource-group $rg \
  --set properties.disableLocalAuth=true
```

## Data-plane RBAC

Two built-in roles on NoSQL API:

| Role | Permissions |
|------|-------------|
| **Cosmos DB Built-in Data Reader** | Read items + metadata |
| **Cosmos DB Built-in Data Contributor** | Read + write + manage stored procs/UDFs/triggers |

Assignments are **data-plane**, not ARM — use `az cosmosdb sql role assignment create` (NOT `az role assignment create`):

```bash
az cosmosdb sql role assignment create \
  --account-name $acct --resource-group $rg \
  --scope "/" \
  --principal-id $(az ad sp show --id $appId --query id -o tsv) \
  --role-definition-id 00000000-0000-0000-0000-000000000002   # Data Contributor
```

Narrow the scope (`/dbs/{db}/colls/{c}`) for least privilege.

## Managed Identity + SDK

```csharp
var client = new CosmosClient(
    accountEndpoint,
    new DefaultAzureCredential());
```

Works in Functions, App Service, Container Apps, AKS (with workload identity). No keys in app settings, no Key Vault round trip.

> Note: Some older tooling (e.g. Functions Cosmos trigger before extension v4.7) required keys. Use the **extension v4.7+** which supports Managed Identity via `__accountEndpoint` / `__credential` suffixes on the connection name.

## Network isolation

| Control | What it does |
|---------|--------------|
| **Firewall IP rules** | Allowlist public IPs (dev workstations, etc.) |
| **Virtual network service endpoints** | Restrict to selected subnets |
| **Private Endpoint** | Private IP in your VNet; disable public network access | ✅ preferred |
| **Dedicated gateway** | Integrated cache; private endpoint support |

Disable public access once private endpoint is in:

```bicep
publicNetworkAccess: 'Disabled'
```

For multi-region accounts, create a private endpoint **per region** if apps need region-local access.

## Encryption

- **At rest**: always encrypted by Microsoft-managed keys.
- **Customer-managed keys (CMK)**: via Azure Key Vault with a System-Assigned or User-Assigned Managed Identity on the account. Enables key rotation, revocation (soft-delete scope).
- **Always Encrypted (client-side)**: deterministic/randomized encryption of individual properties using a CMK; Cosmos engine never sees plaintext. Good for PII columns.
- **In transit**: TLS 1.2+ enforced.

## Dedicated-gateway integrated cache

Sits in front of the account; caches point-reads and query results for configured TTL.

- Cheap read-heavy RAG / catalog workloads: serve from cache at much lower RU cost.
- Cache invalidation is **TTL-based** — don't use for data where staleness matters.
- Deploy multiple gateway instances per region for HA.

## Defender for Cosmos DB

Microsoft Defender plan adds:
- Anomalous data-access alerts (unusual location, extraction volume).
- Credential-leak detection (account key spotted in public repos).
- SQL-injection style attempt alerts on query text.

Enable at the account or subscription level. Priced per 100 RU/s of provisioned throughput.

## Backup & restore (security lens)

- **Continuous backup (PITR)** — RTO minutes, RPO seconds; restore to a new account (you cannot overwrite).
- Backup vault + data are isolated from user account keys; delete of the account keeps backups for the retention window.
- Good defense-in-depth against ransomware / accidental DELETE.

## Hardened Bicep skeleton

```bicep
resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: name
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [ { locationName: location, failoverPriority: 0 } ]
    consistencyPolicy: { defaultConsistencyLevel: 'Session' }
    disableLocalAuth: true
    publicNetworkAccess: 'Disabled'
    minimalTlsVersion: 'Tls12'
    capabilities: [ { name: 'EnableNoSQLVectorSearch' } ]
    backupPolicy: {
      type: 'Continuous'
      continuousModeProperties: { tier: 'Continuous30Days' }
    }
    keyVaultKeyUri: cmkUri                // CMK
  }
}
```

## Exam traps

- Data-plane RBAC ≠ ARM RBAC — **different CLI commands**, different assignments.
- `disableLocalAuth: true` immediately breaks anything still using keys — roll out RBAC first.
- Private Endpoint does **not** propagate to every region automatically; create one per region that needs it.
- CMK doesn't re-encrypt existing data; it wraps the DEK, so rotation is fast but revocation needs careful testing.
- Defender alerts sit in **Microsoft Defender for Cloud**, not the Cosmos blade.

## Sources

- [Security overview](https://learn.microsoft.com/en-us/azure/cosmos-db/database-security)
- [Data-plane RBAC](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac)
- [Disable key auth](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac#disable-local-auth)
- [Private endpoints](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-configure-private-endpoints)
- [CMK setup](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-customer-managed-keys)
- [Always Encrypted](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-always-encrypted)
- [Defender for Cosmos DB](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-defender-for-cosmos)
