# Azure CLI — Security & Key Vault

> **`az keyvault` secrets/keys/certificates + soft-delete & purge, Defender for Cloud, Sentinel, privileged directory roles.**
> **Related:** [04 Identity & RBAC commands](04-identity-and-rbac-commands.md) · [ENTRAID / 05 RBAC & authorization](../ENTRAID/05-rbac-and-authorization.md) · [ENTRAID / 07 PIM](../ENTRAID/07-privileged-identity-management.md)
> **Sources:** [`az keyvault`](https://learn.microsoft.com/en-us/cli/azure/keyvault) · [`az security`](https://learn.microsoft.com/en-us/cli/azure/security) · [`az sentinel`](https://learn.microsoft.com/en-us/cli/azure/sentinel)

---

## Table of contents

1. [Key Vault — create](#key-vault--create)
2. [Access model: RBAC vs access policies](#access-model-rbac-vs-access-policies)
3. [Secrets](#secrets)
4. [Keys](#keys)
5. [Certificates](#certificates)
6. [Soft-delete, recover, purge](#soft-delete-recover-purge)
7. [Networking](#networking)
8. [Managed HSM](#managed-hsm)
9. [Defender for Cloud](#defender-for-cloud)
10. [Microsoft Sentinel](#microsoft-sentinel)
11. [Entra directory roles (`az ad directory-role`)](#entra-directory-roles-az-ad-directory-role)

---

## Key Vault — create

```bash
az keyvault create \
  -g rg-sec -n kv-prod \
  --location eastus \
  --sku standard \
  --enable-rbac-authorization true \
  --enable-purge-protection true \
  --retention-days 90 \
  --default-action Deny \
  --public-network-access Disabled \
  --enabled-for-deployment false \
  --enabled-for-disk-encryption true \
  --enabled-for-template-deployment true

az keyvault list -o table
az keyvault show -g rg-sec -n kv-prod
```

- `--enable-purge-protection true` is **irreversible** — once enabled, deleted vaults/secrets must serve their soft-delete retention fully before re-use of the name. Required for compliance workloads (HIPAA, PCI).
- `--retention-days` range: **7-90**. Default 90.
- Premium SKU adds HSM-backed keys.

## Access model: RBAC vs access policies

Two mutually exclusive models on a given vault:

| Model | How you grant | Pros | Cons |
|---|---|---|---|
| **RBAC** (`--enable-rbac-authorization true`) | `az role assignment` with data-plane roles like *Key Vault Secrets User*, *Key Vault Crypto Officer* | Unified with Azure RBAC, MG-scope inheritance, auditable in Activity Log | Slight propagation delay (≤5 min) |
| **Access policies** (legacy) | `az keyvault set-policy` with explicit list/get/set/delete perms | No propagation delay | Vault-local, not auditable at RBAC level |

New vaults should use RBAC. Migration: flip `--enable-rbac-authorization true`, **add RBAC assignments first**, then access policies can be removed.

### RBAC — data-plane roles on a Key Vault

```bash
KV_ID=$(az keyvault show -g rg-sec -n kv-prod --query id -o tsv)
MI_PID=$(az identity show -g rg-mi -n mi-app --query principalId -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id "$MI_PID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$KV_ID"
```

Common roles:

| Role | Grants |
|---|---|
| Key Vault Secrets User | `get` on secrets |
| Key Vault Secrets Officer | list/get/set/delete secrets |
| Key Vault Certificates Officer | full certificate CRUD |
| Key Vault Crypto User | get/wrap/unwrap/sign/verify on keys |
| Key Vault Crypto Officer | full key CRUD |
| Key Vault Crypto Service Encryption User | `wrap`/`unwrap` only (for CMK pattern) |
| Key Vault Administrator | everything (secrets + keys + certs) |
| Key Vault Reader | metadata read only |
| Key Vault Purge Role (built-in) | purge deleted resources |

### Access policies (legacy)

```bash
az keyvault set-policy -g rg-sec -n kv-legacy \
  --object-id <spObjectId> \
  --secret-permissions get list \
  --key-permissions get unwrapKey wrapKey \
  --certificate-permissions get list

az keyvault delete-policy -g rg-sec -n kv-legacy --object-id <spObjectId>
```

## Secrets

```bash
# Set (auto-creates new version)
az keyvault secret set \
  --vault-name kv-prod --name db-password \
  --value 'Sup3rS3cret!' \
  --tags env=prod rotated-at=2026-04-21

# Get latest, get specific version
az keyvault secret show --vault-name kv-prod --name db-password
az keyvault secret show --vault-name kv-prod --name db-password --version <versionId>

# List (metadata only — no values)
az keyvault secret list --vault-name kv-prod -o table
az keyvault secret list-versions --vault-name kv-prod --name db-password -o table

# Read the value into a shell var (use --query value -o tsv)
PW=$(az keyvault secret show --vault-name kv-prod --name db-password --query value -o tsv)

# Set expiration / activation
az keyvault secret set-attributes \
  --vault-name kv-prod --name db-password \
  --not-before $(date -u -d '+1 day' +'%Y-%m-%dT%H:%M:%SZ') \
  --expires    $(date -u -d '+90 days' +'%Y-%m-%dT%H:%M:%SZ') \
  --enabled true

# From file
az keyvault secret set --vault-name kv-prod --name cert-pfx --file ./cert.pfx --encoding base64

# Delete (soft-delete)
az keyvault secret delete --vault-name kv-prod --name db-password
```

- Secret value max size: **25 KiB**. Certs / large blobs should go into Key Vault as certificates or into Storage + signed URL.
- **Versioning is automatic** — `set` never overwrites, always creates a new version; apps should reference a specific version in production.

## Keys

```bash
# Create (software or HSM)
az keyvault key create \
  --vault-name kv-prod --name app-signing \
  --kty RSA --size 2048 --protection software

az keyvault key create \
  --vault-name kv-prod --name tde-key \
  --kty RSA-HSM --size 3072 --protection hsm   # requires Premium SKU

# Show / list
az keyvault key show --vault-name kv-prod --name app-signing
az keyvault key list --vault-name kv-prod -o table

# Rotation policy
az keyvault key rotation-policy update \
  --vault-name kv-prod --name app-signing \
  --value @rotation-policy.json
# Where rotation-policy.json contains lifetimeActions like rotate/notify

az keyvault key rotate --vault-name kv-prod --name app-signing

# Backup & restore
az keyvault key backup --vault-name kv-prod --name app-signing --file app-signing.backup
az keyvault key restore --vault-name kv-prod --file app-signing.backup

# Encrypt / decrypt / sign / verify (data-plane ops)
az keyvault key encrypt --vault-name kv-prod --name app-signing \
  --algorithm RSA-OAEP --value "$(echo -n 'hello' | base64)"
```

- Rotation: preferred pattern is auto-rotate (every 90-365 days) + consumers reference the key *name* (not a specific version) so they pick up the new version on next fetch.
- HSM-backed keys can't be exported — only wrap/unwrap against them.

## Certificates

```bash
# Create a self-signed cert with policy defaults
az keyvault certificate create \
  --vault-name kv-prod --name tls-example \
  --policy "$(az keyvault certificate get-default-policy)"

# Import an existing PFX
az keyvault certificate import \
  --vault-name kv-prod --name tls-imported \
  --file ./mycert.pfx --password 'PfxPass'

# CA-issued (integrated issuer — DigiCert, GlobalSign, GeoTrust)
az keyvault certificate issuer create \
  --vault-name kv-prod --issuer-name digicert \
  --provider DigiCert --account-id <acctId> --password <apiKey>

az keyvault certificate create \
  --vault-name kv-prod --name tls-prod \
  --policy "$(cat policy.json)"
# policy.json selects issuerName=digicert, subject=CN=www.example.com, etc.

# List / show / download
az keyvault certificate list --vault-name kv-prod -o table
az keyvault certificate show --vault-name kv-prod --name tls-prod
az keyvault certificate download --vault-name kv-prod --name tls-prod --file ./tls-prod.cer --encoding DER
az keyvault certificate download --vault-name kv-prod --name tls-prod --file ./tls-prod.pem --encoding PEM
```

- `az keyvault certificate get-default-policy` → JSON template. Customize for RSA vs ECC, key usage, subject, SAN, renewal before expiry, etc.
- Cert renewal is automatic when `lifetime_actions.action.action_type=AutoRenew` is set — vault requests a new cert from the issuer, downloads it, creates a new version, and the consuming services (App Service with Key Vault cert binding) pick it up.

## Soft-delete, recover, purge

```bash
# Deleted vaults (still in soft-delete)
az keyvault list-deleted -o table
az keyvault show-deleted --name kv-old --location eastus

# Recover
az keyvault recover --name kv-old --location eastus

# Purge (permanent — requires purge permission)
az keyvault purge --name kv-old --location eastus

# Secrets / keys / certs in deleted state
az keyvault secret list-deleted --vault-name kv-prod -o table
az keyvault secret recover --vault-name kv-prod --name api-token
az keyvault secret purge --vault-name kv-prod --name api-token
```

- **Purge protection** prevents `az keyvault purge` — you must wait out the full retention window (7-90 days). Required for compliance workloads.
- Purge RBAC: `Microsoft.KeyVault/locations/deletedVaults/purge/action` (in built-in roles *Contributor*, *Owner*, or the specific *Key Vault Purge Role*).

## Networking

```bash
# Public → deny by default, allow specific IPs and vnet subnets
az keyvault update -g rg-sec -n kv-prod --default-action Deny --bypass AzureServices

az keyvault network-rule add -g rg-sec -n kv-prod --ip-address 203.0.113.10
az keyvault network-rule add -g rg-sec -n kv-prod --vnet-name vnet-svc --subnet subnet-app
az keyvault network-rule remove -g rg-sec -n kv-prod --ip-address 203.0.113.10

# Private endpoint
KV_ID=$(az keyvault show -g rg-sec -n kv-prod --query id -o tsv)
az network private-endpoint create \
  -g rg-sec -n pe-kv-prod \
  --vnet-name vnet-svc --subnet subnet-pe \
  --private-connection-resource-id "$KV_ID" \
  --group-id vault --connection-name pe-kv-prod

# Disable public network access entirely
az keyvault update -g rg-sec -n kv-prod --public-network-access Disabled
```

- `--bypass AzureServices` lets managed Azure services (App Service, Functions, ARM templates) reach the vault even with firewall Deny — trade-off between convenience and zero-trust posture.

## Managed HSM

[Azure Key Vault Managed HSM](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/overview) = dedicated FIPS 140-2 Level 3 HSM pool.

```bash
# Provision (ceremony — requires initial admin object IDs)
az keyvault create \
  -g rg-sec -n mhsm-prod \
  --hsm-name mhsm-prod --location eastus \
  --administrators <adminObjectId1> <adminObjectId2> --retention-days 28

# Activate (first-time security-domain download — keep the file safe!)
az keyvault security-domain download \
  --hsm-name mhsm-prod \
  --sd-wrapping-keys certs/key1.cer certs/key2.cer certs/key3.cer \
  --sd-quorum 2 \
  --security-domain-file mhsm-sd.json

# Keys
az keyvault key create --hsm-name mhsm-prod --name cmk --kty RSA-HSM --size 3072

# Local RBAC (distinct from Azure RBAC — uses role definitions specific to mHSM)
az keyvault role assignment create \
  --hsm-name mhsm-prod --scope "/" \
  --role "Managed HSM Crypto User" \
  --assignee-object-id <miObjectId>
```

## Defender for Cloud

```bash
# Enable per plan
az security pricing list --query "[].{plan:name, tier:pricingTier}" -o table
az security pricing create --name VirtualMachines --tier Standard
az security pricing create --name KeyVaults --tier Standard
az security pricing create --name StorageAccounts --tier Standard

# Auto-provisioning (legacy; AMA/Arc monitoring)
az security auto-provisioning-setting update --name default --auto-provision On

# Alerts
az security alert list -o table
az security alert update --name <alertName> --location eastus --status Dismiss

# Recommendations
az security assessment list -o table
az security assessment show --name <assessmentKey>

# Secure score
az security secure-score list -o table
```

- `az security` coverage is partial; many Defender features (attack paths, workflow automation) are portal-only or via `az rest` against the [Defender for Cloud REST API](https://learn.microsoft.com/en-us/rest/api/defenderforcloud/).
- Enabling a plan bills per-resource-hour — match to compliance scope, not every subscription.

## Microsoft Sentinel

Sentinel is Azure's SIEM/SOAR, layered on a Log Analytics workspace. Requires the `sentinel` extension.

```bash
az extension add --name sentinel

# Onboard a workspace (idempotent)
az sentinel workspace-manager create \
  -g rg-sec --workspace-name laws-sec -n laws-sec

# Data connectors
az sentinel data-connector list -g rg-sec --workspace-name laws-sec -o table

# Analytics rules (Scheduled Alert rule example)
az sentinel alert-rule create \
  -g rg-sec --workspace-name laws-sec \
  --name rule-brute-force --kind Scheduled \
  --display-name "Brute-force sign-ins" \
  --query "SigninLogs | where ResultType == 50126 | summarize count() by UserPrincipalName, IPAddress" \
  --query-frequency PT5M --query-period PT1H \
  --severity High --trigger-threshold 5 \
  --enabled true

# Incidents
az sentinel incident list -g rg-sec --workspace-name laws-sec -o table
```

- The extension lags the full Sentinel feature set — expect to reach for `az rest` against `microsoft.securityinsights` provider for newer capabilities.

## Entra directory roles (`az ad directory-role`)

These are **Entra ID tenant roles** (Global Admin, User Admin, Privileged Role Administrator), distinct from Azure RBAC.

```bash
# List role templates + activated roles
az ad directory-role list -o table

# Activate a role template (first use in tenant)
az ad directory-role activate --role-template-id <templateId>

# Members
az ad directory-role member list --directory-role <displayName>
az ad directory-role member add --directory-role "User Administrator" --member-id <objectId>
az ad directory-role member remove --directory-role "User Administrator" --member-id <objectId>
```

- Complex assignments (scoped admin roles, [administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)) live under `az ad` subgroups and [PIM](../ENTRAID/07-privileged-identity-management.md); CLI coverage is thinner than portal. For PIM eligible-vs-active toggling, use `az rest` against the [`rolemanagement`](https://learn.microsoft.com/en-us/graph/api/resources/unifiedroleeligibilityschedule) Graph endpoints.

## Gotchas

- **Access-policy vs RBAC**: enabling RBAC does not auto-migrate existing policies — old policies keep working alongside new RBAC. Clean up after migration to prevent drift.
- **Key Vault Secrets Officer** has `delete` — reserve for break-glass. Day-to-day automation should use *Secrets User* only.
- **Purge protection enablement is one-way**. Test in dev first.
- **Vault name uniqueness is global** within azure.net DNS — reusing a purged vault name may hit DNS cache for a few minutes.
- **Soft-deleted secrets don't list by default** — add `--include-versions` (for versions) / `list-deleted` (for tombstones).
- **Managed HSM security-domain file** is the disaster-recovery key material. Losing it = all keys unrecoverable. Store encrypted, in multiple locations, with quorum-based access.
- **Defender `az security pricing create --tier Free`** doesn't fully uninstall the plan — it only stops billing. Some agents/extensions persist; clean up manually on deprovision.
- **Sentinel and Defender overlap** — Sentinel imports Defender alerts via connector. Avoid double-alerting by disabling Sentinel's analytics rules that duplicate Defender's.
- **`az keyvault certificate download --encoding DER`** returns just the public cert; the private key never leaves the vault without explicit export and `certificate` permission — treat that permission as privileged.

## Sources

- [`az keyvault`](https://learn.microsoft.com/en-us/cli/azure/keyvault)
- [Key Vault RBAC guide](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)
- [Key Vault soft-delete](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview)
- [Key Vault private endpoints](https://learn.microsoft.com/en-us/azure/key-vault/general/private-link-service)
- [Managed HSM overview](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/overview)
- [`az security`](https://learn.microsoft.com/en-us/cli/azure/security) · [Defender for Cloud docs](https://learn.microsoft.com/en-us/azure/defender-for-cloud/)
- [`az sentinel`](https://learn.microsoft.com/en-us/cli/azure/sentinel) · [Microsoft Sentinel overview](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [Entra directory roles reference](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)
