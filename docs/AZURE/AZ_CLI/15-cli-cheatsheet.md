# Azure CLI Cheat Sheet

> **Daily-usage reference. Scan top-to-bottom, jump to a section by link.**
> Orientation: practical everyday commands, not exam-style trivia. For in-depth context, each section links to the full chapter.
> **Full docs:** [Azure CLI reference](https://learn.microsoft.com/en-us/cli/azure/reference-index)

---

## Table of contents

1. [Auth & context one-liners](#1-auth--context-one-liners)
2. [Output & JMESPath recipes](#2-output--jmespath-recipes)
3. [Resource management](#3-resource-management)
4. [Identity & RBAC](#4-identity--rbac)
5. [Compute (VMs, VMSS, App Service)](#5-compute-vms-vmss-app-service)
6. [Storage](#6-storage)
7. [Networking](#7-networking)
8. [AKS, ACR, Container Apps](#8-aks-acr-container-apps)
9. [Databases](#9-databases)
10. [Monitor & diagnostics](#10-monitor--diagnostics)
11. [Key Vault](#11-key-vault)
12. [`az rest` template](#12-az-rest-template)
13. [Resource Graph queries](#13-resource-graph-queries)
14. [Exit codes & global flags](#14-exit-codes--global-flags)
15. [Common gotchas](#15-common-gotchas)

---

## 1. Auth & context one-liners

*Full chapter: [01 Authentication & context](01-authentication-and-context.md).*

```bash
az login                                               # interactive browser
az login --use-device-code                             # headless / SSH
az login --service-principal -u <appId> -p <secret> --tenant <tid>
az login --identity                                    # system-assigned MI on Azure host
az login --identity --username <userAssignedClientId>  # user-assigned MI

az account show
az account list -o table
az account set --subscription "Dev/Test - East US"
az account list-locations -o table

az account get-access-token                            # ARM token
az account get-access-token --resource https://vault.azure.net
az account clear                                       # wipe token cache + subs
```

Env vars:

```bash
export AZURE_SUBSCRIPTION_ID=...
export AZURE_CLIENT_ID=...   AZURE_TENANT_ID=...   AZURE_CLIENT_SECRET=...
export AZURE_CORE_OUTPUT=jsonc
export AZURE_CORE_ONLY_SHOW_ERRORS=true
```

## 2. Output & JMESPath recipes

*Full chapter: [02 Output & JMESPath queries](02-output-and-jmespath-queries.md).*

| Format | Best for |
|---|---|
| `-o json` | Default, pipe to `jq` / scripts |
| `-o jsonc` | Interactive eyeballing (color) — **don't pipe** |
| `-o table` | Curated columns for human review |
| `-o tsv` | Shell vars, `xargs`, `awk` |
| `-o yaml` | Paste into templates |

Top-10 JMESPath patterns:

```bash
# 1. Flat list of names
--query "[].name" -o tsv

# 2. Pick + rename for a table
--query "[].{name:name, rg:resourceGroup, loc:location}" -o table

# 3. Filter by exact field
--query "[?location=='eastus']"

# 4. Multiple conditions
--query "[?location=='eastus' && powerState=='VM running']"

# 5. Substring match
--query "[?contains(name, 'prod')]"

# 6. Starts-with / ends-with
--query "[?starts_with(name, 'rg-')]"

# 7. Boolean / null literals (backticks)
--query "[?enabled==\`true\`]"
--query "[?tags==\`null\`]"

# 8. Count
--query "length(@)"

# 9. Sort then top N
--query "sort_by([], &createdAt)[:5]"

# 10. Flatten nested arrays
--query "[].securityRules[].{nsg:id, rule:name, prio:priority}"
```

Shell idioms:

```bash
ID=$(az group show -n rg-dev --query id -o tsv)        # scalar
az vm list --query "[].id" -o tsv | xargs -n1 az vm deallocate --ids
az vm list -o json | jq '.[] | .name'                  # CLI → jq
```

## 3. Resource management

*Full chapter: [03 Resource management & governance](03-resource-management-and-governance.md).*

```bash
# Groups
az group create -n rg-dev -l eastus --tags env=dev
az group list -o table
az group delete -n rg-dev --yes --no-wait

# Generic
az resource list --resource-group rg-dev -o table
az resource show --ids <resourceId>
az resource move --destination-group rg-new --ids <id1> <id2>

# Tags
az tag create --resource-id <rid> --tags env=prod owner=team
az resource tag --ids <rid> --tags costcenter=1234 --is-incremental

# Locks
az lock create -n lock-prod -g rg-prod --lock-type CanNotDelete
az lock list -g rg-prod -o table

# Deployments
az deployment group what-if -g rg-dev -f main.bicep --parameters env=dev
az deployment group create  -g rg-dev -f main.bicep --parameters env=dev
az deployment group list -g rg-dev -o table

# Bicep
az bicep upgrade
az bicep build --file main.bicep
az bicep lint  --file main.bicep

# Policy
az policy assignment list -o table
az policy state list --filter "complianceState eq 'NonCompliant'" -o table
```

## 4. Identity & RBAC

*Full chapter: [04 Identity & RBAC commands](04-identity-and-rbac-commands.md).*

```bash
# Directory objects
az ad user show --id alice@contoso.com
az ad group create --display-name devs --mail-nickname devs
az ad group member add --group devs --member-id <oid>

# SP — passwordless OIDC pattern
az ad sp create-for-rbac -n sp-deploy --role Contributor --scopes <scope>
az ad app federated-credential create --id <appObjId> --parameters @fed.json

# RBAC — preferred form (object ID + type)
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id <spOid> --assignee-principal-type ServicePrincipal \
  --scope <storageAccountId>

az role assignment list --assignee alice@contoso.com --all -o table
az role definition list --query "[?contains(roleName,'Data')]" -o table

# Managed identity
az identity create -g rg-mi -n mi-app -l eastus
az vm identity assign     -g rg-dev -n vm-web --identities <miId>
az webapp identity assign -g rg-dev -n app-web --identities <miId>
```

## 5. Compute (VMs, VMSS, App Service)

*Full chapter: [08 Compute commands](08-compute-commands.md).*

```bash
# VMs
az vm create -g rg-dev -n vm-web --image Ubuntu2204 --size Standard_B2ms --generate-ssh-keys
az vm list --show-details --query "[].{name:name,ip:publicIps,state:powerState}" -o table
az vm deallocate -g rg-dev -n vm-web          # stops billing
az vm run-command invoke -g rg-dev -n vm-web --command-id RunShellScript --scripts "uname -a"
az vm list-sizes -l eastus -o table

# VMSS (Flexible)
az vmss create -g rg-dev -n vmss-web --orchestration-mode Flexible \
  --image Ubuntu2204 --instance-count 3 --zones 1 2 3
az vmss scale -g rg-dev -n vmss-web --new-capacity 10

# App Service
az appservice plan create -g rg-dev -n plan-web --sku P1v3 --is-linux
az webapp create -g rg-dev -n app-web --plan plan-web --runtime "DOTNETCORE:8.0"
az webapp deploy -g rg-dev -n app-web --src-path ./build.zip --type zip
az webapp log tail -g rg-dev -n app-web
az webapp config appsettings set -g rg-dev -n app-web --settings K=V
az webapp deployment slot swap -g rg-dev -n app-web --slot staging --target-slot production
```

## 6. Storage

*Full chapter: [09 Storage commands](09-storage-commands.md).*

```bash
# Account
az storage account create -g rg-dev -n stgdev1 --sku Standard_RAGRS --kind StorageV2 \
  --min-tls-version TLS1_2 --allow-shared-key-access false --default-action Deny
az storage account show-connection-string -g rg-dev -n stgdev1 --query connectionString -o tsv

# Auth — prefer Entra
export AZURE_STORAGE_AUTH_MODE=login

# Containers + blobs
az storage container create --account-name stgdev1 -n logs --auth-mode login
az storage blob upload      --account-name stgdev1 -c logs -n app.log -f ./app.log --auth-mode login --overwrite
az storage blob upload-batch --account-name stgdev1 -d logs -s ./logs --auth-mode login
az storage blob list        --account-name stgdev1 -c logs --auth-mode login -o table
az storage blob set-tier    --account-name stgdev1 -c logs -n app.log --tier Cool --auth-mode login

# File share
az storage share-rm create -g rg-dev --storage-account stgdev1 -n share-data --quota 100

# Lifecycle
az storage account management-policy create -g rg-dev --account-name stgdev1 --policy @lifecycle.json
```

## 7. Networking

*Full chapter: [10 Networking commands](10-networking-commands.md).*

```bash
# Vnet + subnets
az network vnet create -g rg-dev -n vnet-dev --address-prefixes 10.0.0.0/16 \
  --subnet-name subnet-app --subnet-prefixes 10.0.1.0/24
az network vnet subnet create -g rg-dev --vnet-name vnet-dev -n subnet-db --address-prefixes 10.0.2.0/24

# Peering
az network vnet peering create -g rg-a -n a-to-b --vnet-name vnet-a --remote-vnet <vnetB-id> \
  --allow-vnet-access

# NSG
az network nsg create -g rg-dev -n nsg-app
az network nsg rule create -g rg-dev --nsg-name nsg-app -n allow-https --priority 100 \
  --access Allow --direction Inbound --protocol Tcp --destination-port-ranges 443

# Public IP / LB
az network public-ip create -g rg-dev -n pip-app --sku Standard --zone 1 2 3 --allocation-method Static
az network lb create -g rg-dev -n lb-web --sku Standard --public-ip-address pip-app \
  --frontend-ip-name fe --backend-pool-name bp

# Private endpoint (template)
az network private-endpoint create -g rg-dev -n pe-svc \
  --vnet-name vnet-dev --subnet subnet-pe \
  --private-connection-resource-id <resId> --group-id <service-group> \
  --connection-name <name>

# Private DNS
az network private-dns zone create -g rg-dev -n privatelink.blob.core.windows.net
az network private-dns link vnet create -g rg-dev -n link --zone-name privatelink.blob.core.windows.net \
  --virtual-network vnet-dev --registration-enabled false

# Troubleshoot
az network watcher test-connectivity --source-resource vm-web -g NetworkWatcherRG \
  --dest-address 10.0.2.10 --dest-port 443
```

## 8. AKS, ACR, Container Apps

*Full chapter: [11 AKS & ACR](11-aks-and-container-registry-commands.md).*

```bash
# ACR
az acr create -g rg-dev -n mycr --sku Standard
az acr login -n mycr
az acr repository show-tags -n mycr --repository myapp -o table
az acr build -r mycr -t myapp:v1 ./src
az acr import -n mycr --source docker.io/library/nginx:latest --image nginx:latest

# AKS
az aks create -g rg-dev -n aks-prod \
  --enable-managed-identity --attach-acr mycr \
  --network-plugin azure --network-plugin-mode overlay \
  --enable-aad --enable-azure-rbac --enable-workload-identity --enable-oidc-issuer \
  --node-count 3 --zones 1 2 3

az aks get-credentials -g rg-dev -n aks-prod
az aks nodepool add -g rg-dev --cluster-name aks-prod -n userpool --node-count 3 --mode User
az aks upgrade -g rg-dev -n aks-prod --kubernetes-version 1.30.5
az aks command invoke -g rg-dev -n aks-prod --command "kubectl get nodes"

# Container Apps
az containerapp env create -g rg-dev -n cae-prod --location eastus --logs-workspace-id <lawsId> --logs-workspace-key <key>
az containerapp create -g rg-dev -n app-web --environment cae-prod \
  --image mycr.azurecr.io/myapp:v1 --registry-server mycr.azurecr.io --registry-identity system \
  --ingress external --target-port 8080 --min-replicas 0 --max-replicas 10
az containerapp logs show -g rg-dev -n app-web --follow --tail 100
```

## 9. Databases

*Full chapter: [12 Databases commands](12-databases-commands.md).*

```bash
# Azure SQL
az sql server create -g rg-dev -n sqlprod01 --admin-user sa --admin-password 'P@ss!'
az sql db create -g rg-dev -s sqlprod01 -n appdb --edition GeneralPurpose --family Gen5 --capacity 2
az sql server firewall-rule create -g rg-dev -s sqlprod01 -n office --start-ip 203.0.113.10 --end-ip 203.0.113.10
az sql server ad-only-auth enable -g rg-dev -n sqlprod01
az sql db restore -g rg-dev -s sqlprod01 -n appdb-r --source-database appdb --time '2026-04-20T12:00:00'

# Cosmos DB
az cosmosdb create -g rg-dev -n cosmosprod --locations regionName=eastus failoverPriority=0 isZoneRedundant=true
az cosmosdb sql database  create -g rg-dev -a cosmosprod -n appdb
az cosmosdb sql container create -g rg-dev -a cosmosprod -d appdb -n orders --partition-key-path "/customerId" --throughput 400

# Postgres Flexible
az postgres flexible-server create -g rg-dev -n pg-prod --tier GeneralPurpose --sku-name Standard_D4s_v3 \
  --admin-user pga --admin-password 'Pg!' --version 16 --public-access Disabled --vnet vnet-dev --subnet subnet-pg \
  --private-dns-zone privatelink.postgres.database.azure.com --yes

# Redis
az redis create -g rg-dev -n redis-prod --sku Standard --vm-size C1 --minimum-tls-version 1.2
az redis list-keys -g rg-dev -n redis-prod
```

## 10. Monitor & diagnostics

*Full chapter: [13 Monitor & diagnostics](13-monitor-and-diagnostics-commands.md).*

```bash
# Log Analytics
WSID=$(az monitor log-analytics workspace show -g rg-ops -n laws-prod --query customerId -o tsv)
az monitor log-analytics query -w "$WSID" \
  --analytics-query "AppRequests | where Success == false | take 10" \
  -o table

# Metrics
az monitor metrics list --resource <vmId> --metric "Percentage CPU" \
  --aggregation Average --interval PT5M --start-time 2026-04-21T00:00:00Z

# Activity log
az monitor activity-log list -g rg-dev --start-time $(date -u -d '24h ago' +'%Y-%m-%dT%H:%M:%SZ') \
  --query "[].{time:eventTimestamp, op:operationName.localizedValue, caller:caller}" -o table

# Diagnostic setting
az monitor diagnostic-settings create \
  --name diag-to-laws --resource <resId> \
  --workspace <lawsResId> \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Alerts
az monitor action-group create -g rg-ops -n ag-oncall --short-name oncall \
  --email-receiver name=team email=oncall@contoso.com useCommonAlertSchema=true

az monitor metrics alert create -g rg-ops -n alert-vm-cpu \
  --scopes <vmId> --condition "avg Percentage CPU > 80" \
  --window-size 5m --evaluation-frequency 1m --severity 2 --action ag-oncall

# Autoscale
az monitor autoscale create -g rg-dev --resource <vmssId> \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-web --min-count 2 --max-count 10 --count 3
```

## 11. Key Vault

*Full chapter: [14 Security & Key Vault](14-security-and-keyvault-commands.md).*

```bash
# Create (RBAC, purge protection, private)
az keyvault create -g rg-sec -n kv-prod --sku standard \
  --enable-rbac-authorization true --enable-purge-protection true \
  --retention-days 90 --default-action Deny --public-network-access Disabled

# Grant data-plane role
az role assignment create --role "Key Vault Secrets User" \
  --assignee-object-id <miOid> --assignee-principal-type ServicePrincipal \
  --scope $(az keyvault show -g rg-sec -n kv-prod --query id -o tsv)

# Secrets
az keyvault secret set  --vault-name kv-prod -n db-pw --value 'S3cr3t!'
az keyvault secret show --vault-name kv-prod -n db-pw --query value -o tsv
az keyvault secret list --vault-name kv-prod -o table

# Keys
az keyvault key create --vault-name kv-prod -n signing --kty RSA --size 2048
az keyvault key rotation-policy update --vault-name kv-prod -n signing --value @rotation.json

# Certificates
az keyvault certificate create --vault-name kv-prod -n tls \
  --policy "$(az keyvault certificate get-default-policy)"

# Soft-delete
az keyvault list-deleted -o table
az keyvault recover --name kv-old --location eastus
az keyvault purge --name kv-old --location eastus          # blocked if purge protection on
```

## 12. `az rest` template

*Full chapter: [05 `az rest`, extensions, graph](05-az-rest-extensions-and-graph.md).*

```bash
# GET
az rest --method GET \
  --url "https://management.azure.com/subscriptions?api-version=2022-12-01"

# POST / PUT with body (inline)
az rest --method POST --url "<url>" --body '{"properties": {...}}'

# Body from file + headers
az rest --method PUT --url "<url>" \
  --body @payload.json \
  --headers "If-Match=*" "x-ms-client-request-id=$(uuidgen)"

# Non-ARM audience
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/me" \
  --resource https://graph.microsoft.com/
```

Pagination loop:

```bash
URL="<initialUrl>"
while [ -n "$URL" ]; do
  R=$(az rest --method GET --url "$URL")
  echo "$R" | jq -c '.value[]'
  URL=$(echo "$R" | jq -r '.nextLink // empty')
done
```

## 13. Resource Graph queries

*Full chapter: [05 `az rest`, extensions, graph](05-az-rest-extensions-and-graph.md#azure-resource-graph-az-graph-query).*

```bash
# Count resources by type
az graph query -q "Resources | summarize count() by type | order by count_ desc"

# All resources missing a tag
az graph query -q "
  Resources
  | where tags.costcenter == '' or isnull(tags.costcenter)
  | project name, type, resourceGroup, subscriptionId
" -o table

# Orphan public IPs
az graph query -q "
  Resources
  | where type =~ 'microsoft.network/publicipaddresses'
  | where isnull(properties.ipConfiguration)
  | project name, resourceGroup, ipAddress = tostring(properties.ipAddress)
" -o table

# Role assignments granting Owner
az graph query -q "
  AuthorizationResources
  | where type == 'microsoft.authorization/roleassignments'
  | where tostring(properties.roleDefinitionId) endswith '8e3af657-a8ff-443c-a75c-2fe8c4bcb635'
  | project principalId = tostring(properties.principalId), scope = tostring(properties.scope)
" -o table
```

## 14. Exit codes & global flags

*Full chapter: [06 Scripting & automation](06-scripting-and-automation.md).*

| Exit | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic (validation / HTTP / auth) |
| 2 | Argument parse error |
| 3 | `az … wait` timeout |

Global flags on every command:

```
--subscription <id|name>        override active sub for this call
--output {json,jsonc,yaml,yamlc,table,tsv,none}
--query <JMESPath>
--debug                         full HTTP trace (sensitive)
--verbose                       URLs + correlation IDs
--only-show-errors              mute info/warnings
--help
```

Patterns:

```bash
set -euo pipefail
trap 'echo "FAIL at line $LINENO: $BASH_COMMAND" >&2' ERR

# Async
az aks create ... --no-wait
az aks wait -g rg -n aks --created --timeout 1800

# Retry
retry() { local m=${1:-5}; shift; local i=0; until "$@"; do ((i++)); (( i > m )) && return 1; sleep $((2**i)); done; }
retry 5 az storage account create -g rg -n stg1 --sku Standard_LRS
```

## 15. Common gotchas

- **Data-plane vs control-plane RBAC**: `Contributor` on a storage account doesn't let you read blobs. You need `Storage Blob Data Contributor` (and `--auth-mode login`).
- **`--overwrite` defaults to false** on blob upload — surprising when re-running scripts.
- **Private endpoint DNS** needs three commands: `private-endpoint create` + `private-dns zone create` + `private-dns link vnet create` + `private-endpoint dns-zone-group create`. Skip one → hits public IP instead of private.
- **JMESPath booleans need backticks**: `[?enabled==\`true\`]`, never `=='true'` (that's a string).
- **`az login`** does **not** imply `Connect-AzAccount` — two separate token caches.
- **`az ad sp create-for-rbac`** prints the password once — store it immediately, or rotate with `az ad sp credential reset`.
- **`--subscription "Friendly Name"`** uses a fuzzy match on display names — typo = silent wrong-sub deploy. Use GUIDs in CI.
- **`az upgrade`** needs `core.output=json` internally — clear your default if the upgrader misbehaves.
- **MSAL token cache race** on parallel `az` calls: give each CI worker its own `AZURE_CONFIG_DIR`.
- **Azure Firewall + `AzureFirewallSubnet`**, VPN + `GatewaySubnet` — subnet names are load-bearing. Misspell = deploy rejects or succeeds but misbehaves.
- **Purge protection is one-way** on Key Vault and Managed HSM — you can't turn it off.
- **Resource Graph is eventually consistent** — freshly-created resources may not appear for 30-60 s.

## Quick links back to chapters

| # | Chapter |
|---|---|
| 00 | [Overview](00-overview.md) |
| 01 | [Authentication & context](01-authentication-and-context.md) |
| 02 | [Output & JMESPath queries](02-output-and-jmespath-queries.md) |
| 03 | [Resource management & governance](03-resource-management-and-governance.md) |
| 04 | [Identity & RBAC commands](04-identity-and-rbac-commands.md) |
| 05 | [`az rest`, extensions, graph](05-az-rest-extensions-and-graph.md) |
| 06 | [Scripting & automation](06-scripting-and-automation.md) |
| 07 | [CLI vs PowerShell vs portal](07-cli-vs-powershell-vs-portal.md) |
| 08 | [Compute commands](08-compute-commands.md) |
| 09 | [Storage commands](09-storage-commands.md) |
| 10 | [Networking commands](10-networking-commands.md) |
| 11 | [AKS & container registry](11-aks-and-container-registry-commands.md) |
| 12 | [Databases](12-databases-commands.md) |
| 13 | [Monitor & diagnostics](13-monitor-and-diagnostics-commands.md) |
| 14 | [Security & Key Vault](14-security-and-keyvault-commands.md) |
