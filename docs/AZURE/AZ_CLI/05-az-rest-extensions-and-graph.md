# Azure CLI — `az rest`, Extensions, and Resource Graph

> **When a command doesn't exist yet — `az rest` talks to any Azure REST API with your CLI token. Extensions add whole new command groups. `az graph query` runs KQL across every subscription.**
> **Related:** [00 Overview](00-overview.md) · [01 Authentication](01-authentication-and-context.md) · [06 Scripting](06-scripting-and-automation.md)
> **Sources:** [`az rest`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-rest) · [Extensions overview](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview) · [`az graph`](https://learn.microsoft.com/en-us/cli/azure/graph) · [Azure Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)

---

## Table of contents

1. [`az rest` — the escape hatch](#az-rest--the-escape-hatch)
2. [Calling ARM, Graph, and data planes](#calling-arm-graph-and-data-planes)
3. [Pagination patterns](#pagination-patterns)
4. [Body payloads & headers](#body-payloads--headers)
5. [Extensions lifecycle](#extensions-lifecycle)
6. [Publishing a private extension](#publishing-a-private-extension)
7. [Azure Resource Graph (`az graph query`)](#azure-resource-graph-az-graph-query)
8. [Resource Graph recipes](#resource-graph-recipes)

---

## `az rest` — the escape hatch

[`az rest`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-rest) is a thin HTTP client that uses your current `az login` token. Use it when:

- A preview API is not yet wrapped by a dedicated `az ...` command.
- You need a feature exposed only on the ARM REST spec.
- You're reverse-engineering what the portal does (watch F12 network calls, copy the URL, paste into `az rest`).

```bash
az rest --method GET \
  --url "https://management.azure.com/subscriptions?api-version=2022-12-01"
```

By default `az rest` detects the audience from the URL and requests the matching scope:

| URL host | Audience (`resource`) | Token fetched |
|---|---|---|
| `management.azure.com` | `https://management.azure.com/` | ARM token |
| `graph.microsoft.com` | `https://graph.microsoft.com/` | Graph token |
| `vault.azure.net` (data plane) | `https://vault.azure.net` | Key Vault token |
| `<acct>.blob.core.windows.net` | must be specified explicitly | Storage token |

Override with `--resource`:

```bash
az rest \
  --method GET \
  --url "https://stgdev1.blob.core.windows.net/?comp=list" \
  --resource https://storage.azure.com/
```

## Calling ARM, Graph, and data planes

### ARM — preview API

```bash
# List deployment stacks (ARM preview at time of writing)
az rest \
  --method GET \
  --url "https://management.azure.com/subscriptions/<subId>/providers/Microsoft.Resources/deploymentStacks?api-version=2024-03-01" \
  --query "value[].{name:name, state:properties.provisioningState}" -o table
```

### Microsoft Graph — beyond `az ad`

```bash
# Get the signed-in user's manager (not exposed via az ad)
az rest \
  --method GET \
  --url "https://graph.microsoft.com/v1.0/me/manager"

# Add a claim mapping policy to an SP
az rest \
  --method POST \
  --url "https://graph.microsoft.com/v1.0/servicePrincipals/<spId>/claimsMappingPolicies/\$ref" \
  --body '{ "@odata.id": "https://graph.microsoft.com/v1.0/policies/claimsMappingPolicies/<policyId>" }'
```

### Data plane — Key Vault example

```bash
az rest \
  --method GET \
  --url "https://kv-prod.vault.azure.net/secrets/api-token?api-version=7.4" \
  --resource https://vault.azure.net
```

- Prefer the dedicated `az keyvault secret show` for this case — `az rest` is for things without a wrapper.
- Data-plane audiences are service-specific (see each service's REST docs).

## Pagination patterns

ARM list APIs return `{value: [...], nextLink: "..."}`. `az rest` does not auto-paginate.

```bash
# Manual pagination
URL="https://management.azure.com/subscriptions/<subId>/resources?api-version=2021-04-01"
while [ -n "$URL" ]; do
  RESP=$(az rest --method GET --url "$URL")
  echo "$RESP" | jq -c '.value[]'
  URL=$(echo "$RESP" | jq -r '.nextLink // empty')
done
```

Pagination helpers inside wrapped commands (`az resource list`, `az vm list`) handle this for you — you only hit it with `az rest`.

## Body payloads & headers

```bash
# Inline body
az rest \
  --method POST \
  --url "https://management.azure.com/subscriptions/<subId>/providers/Microsoft.Authorization/policyAssignments/<name>?api-version=2022-06-01" \
  --body '{
    "properties": {
      "displayName": "Require tag env",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/<guid>",
      "scope": "/subscriptions/<subId>"
    }
  }'

# Body from file
az rest --method PUT --url "..." --body @payload.json

# Custom headers (useful for odata preferences, conditional requests)
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/groups" \
  --headers "ConsistencyLevel=eventual" "Prefer=odata.include-annotations=\"*\""
```

- `--body` accepts a raw JSON string, a `@file.json` reference, or a stdin heredoc.
- `--headers` takes `key=value` pairs, space-separated. Multi-value headers need comma-joined values inside the single `key=value` form.
- `az rest` sets `Content-Type: application/json` automatically on methods with a body.

## Extensions lifecycle

[Extensions](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview) ship commands that aren't in the core CLI. Common reasons: preview features, separate release cadence, optional dependencies, third-party publishers.

```bash
# Discover
az extension list-available --output table
az extension list-available --query "[?contains(name,'aks')]"

# Install
az extension add --name aks-preview
az extension add --name aks-preview --version 7.0.0b6    # pin
az extension add --source ./myext-0.1.0-py3-none-any.whl # local wheel

# List installed
az extension list -o table

# Update / remove
az extension update --name aks-preview
az extension remove --name aks-preview
```

- Extensions install into `~/.azure/cliextensions/<name>/` — a Python wheel unpack. `az` picks them up on next invocation.
- **Preview vs Experimental**: preview = API shape may change; experimental = may vanish entirely. Both show a warning on every invocation.
- **Auto-install**: set `az config set extension.use_dynamic_install=yes_without_prompt` to install on first use without prompting (handy in CI; dangerous on dev boxes if you expect reproducibility).
- **Index override**: enterprises can host their own extension index — set `az config set extension.index_url=https://my-extensions.internal/index.json`.

### Common extensions worth knowing

| Extension | What it adds |
|---|---|
| `aks-preview` | Newer AKS features before they land in core |
| `interactive` | `az interactive` REPL |
| `resource-graph` | `az graph query` (auto-loaded on first use on recent CLI) |
| `containerapp` | Azure Container Apps commands (now in core, but some previews still here) |
| `datafactory` / `synapse` | ADF + Synapse pipelines |
| `storage-preview` | Storage preview features (object replication, inventory) |
| `application-insights` | `az monitor app-insights` fully |
| `ml` | Azure Machine Learning |
| `fleet` | AKS Fleet Manager |

## Publishing a private extension

Quick outline — full guide: [Publish a private extension](https://learn.microsoft.com/en-us/cli/azure/service-page/azure%20cli).

```bash
# Scaffold
az extension add --name azdev
azdev setup
azdev extension create myext --local-sdk <module>

# Build the wheel
cd src/myext
python setup.py bdist_wheel

# Host the .whl somewhere reachable (blob storage + SAS, artifact feed)
# Generate/refresh the private index JSON pointing at the wheels
azdev extension update-index https://mycdn.example.com/myext-0.1.0-py3-none-any.whl
```

Consumers then either install directly from the wheel URL (`az extension add --source <wheelUrl>`) or point at your index via `extension.index_url`.

## Azure Resource Graph (`az graph query`)

[Azure Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview) is a KQL-queryable index of every ARM resource across every subscription and MG your identity can see. `az graph query` wraps it. Requires the `resource-graph` extension (auto-loads on newer CLI).

```bash
# Count resources by type
az graph query -q "Resources | summarize count() by type | order by count_ desc"

# All storage accounts with public-network-access enabled
az graph query -q "
  Resources
  | where type == 'microsoft.storage/storageaccounts'
  | where properties.publicNetworkAccess == 'Enabled'
  | project name, resourceGroup, subscriptionId, location
" -o table

# Scope to a management group
az graph query --management-groups mg-landing-zones -q "Resources | summarize count() by subscriptionId"

# Scope to specific subs
az graph query --subscriptions <subId1> <subId2> -q "Resources | where tags.env == 'prod' | project name,type"
```

KQL subset supported: `project`, `where`, `summarize`, `join` (inner), `extend`, `order by`, `top`, `count`, `parse`. Full list: [Resource Graph query language](https://learn.microsoft.com/en-us/azure/governance/resource-graph/concepts/query-language).

### Tables

| Table | Contents |
|---|---|
| `Resources` | All ARM resources |
| `ResourceContainers` | Subscriptions, RGs, MGs |
| `PolicyResources` | Policy assignments, definitions, exemptions, states |
| `AuthorizationResources` | Role assignments, definitions, eligibility |
| `HealthResources` | Resource health (preview) |
| `AdvisorResources` | Advisor recommendations |
| `ServiceHealthResources` | Service health events |
| `SecurityResources` | Defender for Cloud assessments (if licensed) |
| `ExtendedProperties` | Extended resource props (select few types) |

### Paging and limits

- Default page size: 1000 rows. Pass `--first 5000` to widen (max 1000 per page internally — the CLI auto-paginates up to `--first`).
- Hard cap on `--first`: **5000**. Beyond that, use `summarize` / aggregation in the query itself.
- `--skip` for classic pagination; prefer `order by` + `take` for deterministic results.

## Resource Graph recipes

### All VMs with their primary NIC private IP

```bash
az graph query -q "
  Resources
  | where type =~ 'microsoft.compute/virtualmachines'
  | extend nicId = tostring(properties.networkProfile.networkInterfaces[0].id)
  | join kind=leftouter (
      Resources
      | where type =~ 'microsoft.network/networkinterfaces'
      | extend privateIp = tostring(properties.ipConfigurations[0].properties.privateIPAddress)
      | project nicId = id, privateIp
    ) on nicId
  | project name, resourceGroup, privateIp, size = tostring(properties.hardwareProfile.vmSize)
" -o table
```

### Every resource missing a required tag

```bash
az graph query -q "
  Resources
  | where tags.costcenter == '' or isnull(tags.costcenter)
  | project name, type, resourceGroup, subscriptionId
" -o table
```

### Cost-model feeder — resource counts per subscription per type

```bash
az graph query -q "
  Resources
  | summarize total=count() by subscriptionId, type
  | order by total desc
  | take 50
" -o table
```

### Orphan public IPs (no NIC / LB attached)

```bash
az graph query -q "
  Resources
  | where type =~ 'microsoft.network/publicipaddresses'
  | where isnull(properties.ipConfiguration)
  | project name, resourceGroup, subscriptionId, ipAddress = tostring(properties.ipAddress)
" -o table
```

### Role-assignment sprawl — who has Owner where?

```bash
az graph query -q "
  AuthorizationResources
  | where type == 'microsoft.authorization/roleassignments'
  | extend roleDefId = tostring(properties.roleDefinitionId)
  | where roleDefId endswith '8e3af657-a8ff-443c-a75c-2fe8c4bcb635'  // Owner
  | project principalId = tostring(properties.principalId),
            scope = tostring(properties.scope)
" -o table
```

## Gotchas

- **`az rest` never writes to stderr on 4xx** — always inspect `--debug` output when a call silently returns empty. HTTP errors become CLI non-zero exits with a terse message.
- **`api-version` matters** — the same endpoint in different API versions has different shapes. Pin an explicit `?api-version=...`; do not rely on "latest."
- **Don't rewrap `az rest` into a macro** for something the native CLI already supports. Native commands handle pagination, retries, and return-shape normalization.
- **Extension Python deps leak**: extensions install into a shared `~/.azure/cliextensions/` — two extensions with conflicting pins can break both. Use `az extension remove` + reinstall, or a fresh `AZURE_CONFIG_DIR` per tool.
- **Resource Graph is eventually consistent** — resources created milliseconds ago may not appear for 30-60 seconds. Do not use `az graph query` as a consistency barrier after a deploy.
- **Resource Graph doesn't expose deep child props** for every type — some nested arrays are flattened, others are stubbed. When in doubt, fall back to `az <service> show`.

## Sources

- [`az rest`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-rest)
- [Extensions overview](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview)
- [Available extensions list](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-list)
- [Azure Resource Graph overview](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)
- [Resource Graph query language](https://learn.microsoft.com/en-us/azure/governance/resource-graph/concepts/query-language)
- [Resource Graph sample queries](https://learn.microsoft.com/en-us/azure/governance/resource-graph/samples/samples-by-category)
