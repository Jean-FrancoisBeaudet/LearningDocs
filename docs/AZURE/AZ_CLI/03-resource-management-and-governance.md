# Azure CLI — Resource Management & Governance

> **Resource groups, generic `az resource` ops, tags, locks, ARM/Bicep deployments, Azure Policy, and management groups — the control-plane basics.**
> **Related:** [00 Overview](00-overview.md) · [04 Identity & RBAC](04-identity-and-rbac-commands.md) · [AZ-104/03 subscriptions & governance](../AZ-104/03-subscriptions-and-governance.md) · [AZ-104/07 ARM & Bicep](../AZ-104/07-arm-and-bicep.md)
> **Sources:** [`az group`](https://learn.microsoft.com/en-us/cli/azure/group) · [`az resource`](https://learn.microsoft.com/en-us/cli/azure/resource) · [`az tag`](https://learn.microsoft.com/en-us/cli/azure/tag) · [`az lock`](https://learn.microsoft.com/en-us/cli/azure/lock) · [`az deployment`](https://learn.microsoft.com/en-us/cli/azure/deployment) · [`az policy`](https://learn.microsoft.com/en-us/cli/azure/policy) · [`az account management-group`](https://learn.microsoft.com/en-us/cli/azure/account/management-group)

---

## Table of contents

1. [Resource groups](#resource-groups)
2. [Generic resource commands](#generic-resource-commands)
3. [Tags](#tags)
4. [Resource locks](#resource-locks)
5. [ARM & Bicep deployments](#arm--bicep-deployments)
6. [`az bicep` toolchain](#az-bicep-toolchain)
7. [Azure Policy](#azure-policy)
8. [Management groups](#management-groups)
9. [Purge operations](#purge-operations)

---

## Resource groups

[`az group`](https://learn.microsoft.com/en-us/cli/azure/group) manages [resource groups](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-cli) — the deployment and lifecycle boundary for everything else.

```bash
az group list -o table
az group create --name rg-dev --location eastus --tags env=dev owner=team-a

az group show --name rg-dev
az group update --name rg-dev --set tags.costcenter=1234

# List contents
az resource list --resource-group rg-dev -o table

# Move resources between RGs (same subscription)
az resource move \
  --destination-group rg-dev-new \
  --ids /subscriptions/.../resourceGroups/rg-dev/providers/Microsoft.Storage/storageAccounts/stgdev1

# Delete — watch --no-wait for CI
az group delete --name rg-dev --yes --no-wait
az group wait --name rg-dev --deleted
```

- `--no-wait` returns immediately; pair with [`az group wait`](https://learn.microsoft.com/en-us/cli/azure/group#az-group-wait) in scripts.
- Cross-subscription moves: [`az resource move`](https://learn.microsoft.com/en-us/cli/azure/resource#az-resource-move) with `--destination-subscription-id`.
- Not every resource type supports move — check the [move support matrix](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-support-resources).

## Generic resource commands

[`az resource`](https://learn.microsoft.com/en-us/cli/azure/resource) is the fallback for anything without a dedicated command group — or for bulk operations.

```bash
# List everything in the subscription
az resource list -o table

# Filter by type + tag
az resource list \
  --resource-type Microsoft.Storage/storageAccounts \
  --tag env=prod \
  --query "[].{name:name, rg:resourceGroup, loc:location}" -o table

# Show a resource by ID (the universal pointer)
az resource show --ids /subscriptions/.../storageAccounts/stgdev1

# Patch arbitrary properties (when no dedicated command exists)
az resource update \
  --ids <resourceId> \
  --set properties.minimumTlsVersion=TLS1_2
```

- [`az resource list --query "[].id" -o tsv | xargs -n1 az resource show --ids`] is the standard "inspect everything" pattern.
- `--set`, `--add`, `--remove` on `az resource update` take JSON-path-ish expressions; they issue a PATCH to ARM.

## Tags

[Tags](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources) are name/value pairs on resources, groups, and subscriptions. Limit: **50 per resource**, values ≤ 256 chars.

### On a resource

```bash
# Set/replace all tags (dedicated command group: az tag)
az tag create --resource-id <rid> --tags env=dev owner=team-a costcenter=1234

# Merge into existing tags (no replace) — use az resource tag
az resource tag --ids <rid> --tags newtag=value --is-incremental

# Remove a single tag key
az tag update --resource-id <rid> --operation Delete --tags owner=team-a
```

### On a resource group

```bash
az group update --name rg-dev --set tags.env=dev tags.owner=team-a
```

### Bulk: add a tag to every VM in a sub

```bash
for id in $(az vm list --query "[].id" -o tsv); do
  az resource tag --ids "$id" --tags env=prod --is-incremental
done
```

Or apply via Policy's [**Inherit a tag from the resource group**](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#tags) built-in — see [Azure Policy](#azure-policy) below for the CLI command.

### Tag inheritance

Tags **do not** cascade by default. Rely on Policy (`Modify` effect) to enforce RG → resource inheritance.

## Resource locks

[Locks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources) prevent accidental delete or change at sub / RG / resource scope.

```bash
# Create (two levels: CanNotDelete, ReadOnly)
az lock create \
  --name lock-rg-prod \
  --resource-group rg-prod \
  --lock-type CanNotDelete \
  --notes "Prod RG — no deletes"

az lock list -o table
az lock list --resource-group rg-prod -o table
az lock delete --name lock-rg-prod --resource-group rg-prod
```

- `ReadOnly` blocks both delete and modify — including data-plane actions that secretly go through ARM (e.g., rotating a storage key).
- Locks **inherit downward**: an RG lock locks every resource inside.
- Only `Owner` and `User Access Administrator` can create/delete locks by default — a `Contributor` can create every other resource but not locks.

## ARM & Bicep deployments

[Deployments](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) target one of four scopes:

| Scope | CLI group | Common uses |
|---|---|---|
| Resource group | `az deployment group` | Most workloads |
| Subscription | `az deployment sub` | RG creation, subscription-level RBAC, Policy |
| Management group | `az deployment mg` | Landing zones, MG-scoped Policy |
| Tenant | `az deployment tenant` | Cross-MG or whole-tenant policy |

```bash
# Deploy a Bicep file to an RG (Bicep is compiled on the fly)
az deployment group create \
  --name deploy-webapp \
  --resource-group rg-dev \
  --template-file main.bicep \
  --parameters env=dev sku=P1v3 \
  --parameters @secrets.bicepparam

# Preview the change set
az deployment group what-if \
  --resource-group rg-dev \
  --template-file main.bicep \
  --parameters env=dev

# Require confirmation after showing what-if
az deployment group create ... --confirm-with-what-if

# Async pattern
az deployment group create ... --no-wait
az deployment group wait --resource-group rg-dev --name deploy-webapp --created
```

- `--template-uri https://raw.githubusercontent.com/...` deploys straight from git/raw URL for ARM JSON (Bicep still needs local compile).
- `--mode Complete` deletes anything in the RG not in the template — use with care. Default is `Incremental`.
- [`az deployment group validate`](https://learn.microsoft.com/en-us/cli/azure/deployment/group#az-deployment-group-validate) runs pre-flight validation without side effects.
- For **stacks** (deployment stacks = deployment + lifecycle-managed resource bag): [`az stack group`](https://learn.microsoft.com/en-us/cli/azure/stack/group) (see [deployment stacks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks)).

### What-if output interpretation

```
Resource changes: 3 to create, 1 to modify.

+ Microsoft.Storage/storageAccounts/stgnew       [create]
~ Microsoft.Web/sites/app-dev                    [modify]
   ~ properties.siteConfig.minTlsVersion:        "1.0" => "1.2"
```

Symbol legend: `+` create, `-` delete, `~` modify, `*` no-change, `!` deploy-complete-mode delete.

## `az bicep` toolchain

[`az bicep`](https://learn.microsoft.com/en-us/cli/azure/bicep) wraps the [Bicep CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install) — used under the hood for every `--template-file *.bicep` invocation.

```bash
az bicep install                     # first time
az bicep upgrade                     # latest
az bicep version

az bicep build --file main.bicep          # compile to ARM JSON
az bicep decompile --file main.json       # ARM → Bicep (best-effort)
az bicep format --file main.bicep         # formatter
az bicep lint --file main.bicep           # analyzer
az bicep publish --file main.bicep --target br:<registry>.azurecr.io/bicep/modules/foo:v1
az bicep restore --file main.bicep        # fetch referenced registry modules into cache
```

- Modules can be **local**, from a **Bicep registry** (ACR-backed), or from the [public registry](https://github.com/Azure/bicep-registry-modules) (`br/public:`).
- `bicepconfig.json` controls linter rules, module aliases, and experimental features.

## Azure Policy

[Azure Policy](https://learn.microsoft.com/en-us/azure/governance/policy/overview) evaluates resources against definitions (rules) grouped into initiatives (sets), assigned at a scope.

```bash
# Built-in definitions
az policy definition list --query "[?policyType=='BuiltIn'].{name:displayName, id:name}" -o table

# Assign a built-in policy at an RG
az policy assignment create \
  --name require-tls12-storage \
  --display-name "Require TLS 1.2 on storage accounts" \
  --policy "17k78e20-9358-41c9-923c-fb736d382a4d" \
  --scope /subscriptions/<subId>/resourceGroups/rg-prod

# Custom definition from a JSON file
az policy definition create \
  --name require-env-tag \
  --rules policy-rules.json \
  --mode Indexed \
  --display-name "Require env tag"

# Initiative (set)
az policy set-definition create \
  --name sec-baseline \
  --definitions definitions.json

# Compliance query (what's non-compliant?)
az policy state list \
  --filter "complianceState eq 'NonCompliant'" \
  --query "[].{resource:resourceId, policy:policyDefinitionName}" \
  -o table

# Remediation for DeployIfNotExists
az policy remediation create \
  --name remediate-tags \
  --policy-assignment <assignmentId> \
  --resource-group rg-prod
```

- Effects: `Audit`, `Deny`, `Modify`, `Append`, `DeployIfNotExists`, `AuditIfNotExists`, `Disabled`. `DeployIfNotExists` and `Modify` need a **system-assigned or user-assigned managed identity** on the assignment (pass `--mi-system-assigned` or `--mi-user-assigned <id>`).
- Built-in policies are referenced by GUID, not name — the `--policy` flag takes either full ID or GUID suffix.
- Exemptions: [`az policy exemption create`](https://learn.microsoft.com/en-us/cli/azure/policy/exemption#az-policy-exemption-create) with `--exemption-category Waiver` or `Mitigated`.

## Management groups

[Management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview) organize subscriptions hierarchically — up to **6 levels deep** under the tenant root.

```bash
az account management-group list -o table

az account management-group create \
  --name mg-landing-zones \
  --display-name "Landing Zones" \
  --parent <rootMgName>

# Move a subscription under an MG
az account management-group subscription add \
  --name mg-landing-zones \
  --subscription <subId>

# Inspect the hierarchy
az account management-group show \
  --name <rootMgName> --expand --recurse \
  --query "children[].{name:displayName, children:children[].displayName}" \
  -o yaml
```

- Tenant root (`Tenant Root Group`) — created automatically; you need Global Administrator + **root scope role assignment** (one-time "elevate" via the portal or `az role assignment create --scope /providers/Microsoft.Management/managementGroups/<tenantId>`) to manage it with the CLI.
- MG-scoped RBAC and Policy assignments inherit to every sub/RG/resource below.

## Purge operations

Several Azure services have **soft-delete + purge-protection** semantics. The CLI surface for hard-deleting the tombstone:

| Service | Purge command |
|---|---|
| [Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview) | `az keyvault purge --name <kv>` |
| Key Vault secret/key/cert | `az keyvault secret purge`, `az keyvault key purge`, `az keyvault certificate purge` |
| [Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/delete-workspace) | `az monitor log-analytics workspace delete --force` + re-create within soft-delete window to recover; no explicit purge |
| [Cognitive Services / Azure OpenAI account](https://learn.microsoft.com/en-us/azure/ai-services/recover-purge-resources) | `az cognitiveservices account purge --name ... --location ...` |
| Deleted resource group | No purge — deletion is permanent after ~14 days of tombstone |

- **Purge protection** cannot be reverted once enabled on Key Vault — plan for the 7-90 day soft-delete retention.
- Purge requires a specific role: for Key Vault it's [Key Vault Contributor with `Microsoft.KeyVault/locations/deletedVaults/purge/action`](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide) or the built-in `Key Vault Purge Role`.

## Gotchas

- **`--mode Complete` on RG deployment wipes anything not in the template**. Always run `what-if` first.
- **`az resource update --set` issues a PATCH**, not a PUT — it merges. For a full replace use `--resource` with the full JSON payload or re-deploy via Bicep.
- **Tags propagation**: Policy `Modify` effect applies on-change and on-remediation, not retroactively without a remediation task.
- **Subscription deployments create resource groups** but **cannot delete them on `Complete` mode** — `Complete` only applies to the resources inside the deployment's own scope.
- **Resource move limitations** include almost all networking resources (vnets can move, but only with their contents, and gateways block the move).
- **Bicep registry auth**: `az bicep publish` to ACR requires [`AcrPush` role](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles); pulling modules only needs `AcrPull`.

## Sources

- [Manage resource groups — CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-cli)
- [Tag resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)
- [Lock resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)
- [Bicep deployments — CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-cli)
- [`az deployment group what-if`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-what-if)
- [Azure Policy CLI](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/programmatically-create)
- [Management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Move resources support matrix](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-support-resources)
