# Azure CLI vs Azure PowerShell vs Portal vs IaC

> **When to pick `az`, `Az` PowerShell, the portal, or an IaC tool — and how to translate between them.**
> **Related:** [00 Overview](00-overview.md) · [03 Resource management (deployments)](03-resource-management-and-governance.md) · [06 Scripting & automation](06-scripting-and-automation.md)
> **Sources:** [Choose the right CLI](https://learn.microsoft.com/en-us/cli/azure/choose-the-right-azure-command-line-tool) · [Azure PowerShell overview](https://learn.microsoft.com/en-us/powershell/azure/what-is-azure-powershell)

---

## Table of contents

1. [The four surfaces](#the-four-surfaces)
2. [Decision matrix](#decision-matrix)
3. [Parity & preview gaps](#parity--preview-gaps)
4. [`az` ↔ `Az` command mapping](#az--az-command-mapping)
5. [Output shapes & filtering](#output-shapes--filtering)
6. [IaC: when Bicep / Terraform beats both](#iac-when-bicep--terraform-beats-both)
7. [Interop patterns](#interop-patterns)

---

## The four surfaces

| Surface | Language | State | Primary strength |
|---|---|---|---|
| **Azure CLI (`az`)** | Python-backed CLI, shell-agnostic | Imperative, no state file | Cross-platform scripting, single-purpose automation, data-plane helpers |
| **Azure PowerShell (`Az.*` modules)** | PowerShell cmdlets | Imperative, no state file | Object pipeline, tight integration with Windows infra + M365 modules |
| **Portal** | Web UI at [portal.azure.com](https://portal.azure.com) | Stateful (applied live) | Discovery, one-off exploratory ops, no-install changes |
| **IaC (Bicep / ARM / Terraform / Pulumi)** | Declarative DSLs | Stateful (template/state) | Repeatable multi-resource deployments, drift detection, reviews in PR |

`az` and `Az` have roughly **>95% ARM parity** — it's rare for only one to support a service at GA. Both are thin wrappers over the ARM REST API plus service data planes.

## Decision matrix

Pick per task, not per project. It's normal to mix them.

| Task | Best fit | Why |
|---|---|---|
| One-off "what does this resource look like?" | Portal | No install, nested blades, JSON view |
| Reproducible multi-resource deploy | Bicep / Terraform | Declarative, review-able, drift-checkable |
| Scripting a 3-step flow in CI | `az` | Same script runs on Linux runners, Windows runners, dev laptops |
| Cross-tool pipeline (`az` output → `kubectl`, `terraform`, `gh`) | `az` | Flat-text + JMESPath is shell-friendly |
| Object-oriented filtering / chaining | `Az` PowerShell | Pipeline passes typed objects, not strings |
| Bulk operations where typed errors matter | `Az` | `try/catch` with typed Azure exceptions, better loops |
| On-call "quick check" on a phone | Cloud Shell + `az` | Both CLI tools ship in Cloud Shell |
| Entra admin tasks (M365 side) | Graph PowerShell | `Az` covers only the Azure-facing slice; `Az` is **not** the replacement for AzureAD/MsOnline modules |
| Windows workload management (DSC, registry, on-prem hybrid) | `Az` | Natural fit with local PowerShell tooling |
| Deep data-plane work (blob copy, queue peek) | `az` (then language SDK if heavier) | `az storage` command surface is richer than `Az.Storage` data-plane cmdlets |
| Policy-as-code / landing zones | Bicep or Terraform | Declarative diff wins every time |
| Ad-hoc Resource Graph query | `az graph query` | Single command; works everywhere |

## Parity & preview gaps

- New services usually land in `az` **first as an extension** (`az extension add --name <preview>`), then core GA.
- `Az` often follows within a release or two. Check the [PS release notes](https://learn.microsoft.com/en-us/powershell/azure/release-notes-azureps) and [CLI release notes](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli).
- Very new / experimental features may only exist via **`az rest`** or the **ARM REST API** directly — neither tool has wrapped them yet.
- Some legacy features persist only in PowerShell (certain Backup / ASR cmdlets, classic deployment model remnants). Very few matter in new work.

## `az` ↔ `Az` command mapping

Rough translation cheat table. `Az` cmdlets return typed PowerShell objects; `az` returns JSON strings.

| Area | Azure CLI | Azure PowerShell |
|---|---|---|
| Login | `az login` | `Connect-AzAccount` |
| Subscription switch | `az account set --subscription <id>` | `Set-AzContext -Subscription <id>` |
| Resource group list | `az group list` | `Get-AzResourceGroup` |
| Resource group create | `az group create -n <n> -l <loc>` | `New-AzResourceGroup -Name <n> -Location <loc>` |
| Generic resource list | `az resource list` | `Get-AzResource` |
| Role assignment | `az role assignment create` | `New-AzRoleAssignment` |
| VM create | `az vm create` | `New-AzVM` |
| VM run command | `az vm run-command invoke` | `Invoke-AzVMRunCommand` |
| Storage key list | `az storage account keys list` | `Get-AzStorageAccountKey` |
| Blob upload | `az storage blob upload` | `Set-AzStorageBlobContent` |
| Key Vault secret | `az keyvault secret show` | `Get-AzKeyVaultSecret` |
| Deployment (RG) | `az deployment group create` | `New-AzResourceGroupDeployment` |
| What-if | `az deployment group what-if` | `Get-AzResourceGroupDeploymentWhatIfResult` |
| Policy assignment | `az policy assignment create` | `New-AzPolicyAssignment` |
| Log Analytics query | `az monitor log-analytics query` | `Invoke-AzOperationalInsightsQuery` |
| Graph (Entra) — user | `az ad user show` | `Get-MgUser` (Microsoft Graph PowerShell, not `Az`) |

Cmdlet verb/noun pattern: PowerShell = **`Verb-AzNoun`** (New/Get/Set/Remove + Az + singular noun). CLI = **`az <noun-group> [sub ...] <verb>`**.

## Output shapes & filtering

### CLI (JSON + JMESPath)

```bash
az vm list \
  --query "[?powerState=='VM running'].{name:name, size:hardwareProfile.vmSize}" \
  -o table
```

### PowerShell (objects + pipeline)

```powershell
Get-AzVM -Status `
  | Where-Object { $_.PowerState -eq 'VM running' } `
  | Select-Object Name, @{n='Size';e={$_.HardwareProfile.VmSize}} `
  | Format-Table
```

Which is better? Context:

- CLI + JMESPath shines when **one command returns exactly what you need** and you want a single shell pipeline.
- PowerShell shines when **you're joining two commands' outputs** or doing imperative loops with typed branching — `Where-Object` on objects beats stitching JSON in shell.

### Interop: parse `az` output in PowerShell

```powershell
$vms = az vm list -o json | ConvertFrom-Json
$vms | Where-Object { $_.location -eq 'eastus' } | Select-Object name
```

`ConvertFrom-Json` returns `[PSCustomObject]` — dot-access and `Where-Object` just work. In Windows PowerShell 5.1, very large payloads (>1-2 MB) are slow — prefer `Az` cmdlets there.

## IaC: when Bicep / Terraform beats both

Any of these signals → reach for IaC, not a CLI script:

- Resource count ≥ 5 per deployment.
- Lifetime ≥ a few weeks.
- More than one environment (dev/stage/prod).
- Review / PR workflow required by governance.
- Drift detection is needed.

Within Azure:

- **[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)** — native to Azure, uses ARM under the hood, no state file, ships with the CLI (`az bicep ...`), best-in-class `what-if`, deployment stacks.
- **[ARM JSON templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)** — Bicep's compile target. Write new work in Bicep; maintain existing ARM JSON if you inherited it.
- **[Terraform](https://learn.microsoft.com/en-us/azure/developer/terraform/overview)** — multi-cloud, huge community modules, state file (local or [Azure blob backend](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)), `terraform plan` diff.
- **[Pulumi](https://www.pulumi.com/docs/clouds/azure/)** — real programming languages (TS/Python/Go/.NET) over the Azure provider; state via Pulumi Cloud or self-hosted.

CLI/PowerShell still play a role inside an IaC project: bootstrap (creating the state backend, the CI SP, the federated cred), post-deploy hooks (`az rest` for features not yet in the provider), and ad-hoc ops on resources the IaC doesn't own.

## Interop patterns

### Share context across `az` and `Az`

Both tools authenticate separately — logging in with one does **not** log in the other. Scripts that use both:

```powershell
Connect-AzAccount -Identity                  # PS
az login --identity                          # CLI
```

Or export a token from one and feed the other:

```powershell
$token = (Get-AzAccessToken -ResourceUrl 'https://management.azure.com/').Token
$env:ARM_ACCESS_TOKEN = $token               # used by Terraform's azurerm provider
```

### Use the CLI inside an `Az` script

```powershell
# CLI has a command PowerShell doesn't
$rest = az rest --method GET --url "https://management.azure.com/..." | ConvertFrom-Json
```

Perfectly fine. `az` is globally available on hosted runners (GitHub/ADO) whether the job is PS or bash.

### Bicep from either tool

```bash
az deployment group create -g rg-dev -f main.bicep
```

```powershell
New-AzResourceGroupDeployment -ResourceGroupName rg-dev -TemplateFile main.bicep
```

Same ARM call under the hood — same `what-if` output format — pick per preference.

### Extensibility gaps

| Need | Reach for |
|---|---|
| Entra admin beyond `az ad` | [Microsoft Graph PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview) (`Get-MgUser`, etc.) |
| Multi-cloud infra | Terraform / Pulumi |
| Kubernetes inside the cluster | `kubectl` (`az aks get-credentials` then handoff) |
| Fine-grained cost querying | [Cost Management CLI](https://learn.microsoft.com/en-us/azure/cost-management-billing/automate/automate-workflows-cli) (`az costmanagement`) or REST |
| Azure DevOps ops | `az devops` (extension) or ADO REST |

## Gotchas

- **Two auth caches**: `~/.azure/` (CLI) vs `~/.Azure/` capital-A (PowerShell). Both can be present on a dev box; keep them mentally separate.
- **PowerShell 5.1 vs 7.x**: `Az` works on both but is faster on 7.x; a few newer `Az` features drop 5.1 support. CLI has no equivalent split.
- **`az` → JSON + `ConvertFrom-Json`** is a common pattern in pwsh scripts; safe, but large datasets are faster via `Az` cmdlets directly.
- **"Can I just write everything in `az`?"** — yes, and many teams do; it's the lowest-common-denominator across Linux, macOS, Windows, and CI. Just don't ignore the option of `Az` when you genuinely need typed objects.
- **Portal-only features exist** for a few weeks at service launch — expect to `az rest` or wait for wrappers.

## Sources

- [Choose the right Azure command-line tool](https://learn.microsoft.com/en-us/cli/azure/choose-the-right-azure-command-line-tool)
- [Azure PowerShell overview](https://learn.microsoft.com/en-us/powershell/azure/what-is-azure-powershell)
- [Azure CLI vs Azure PowerShell vs Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/choose-deployment-tool)
- [Bicep vs ARM](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
- [Terraform on Azure](https://learn.microsoft.com/en-us/azure/developer/terraform/overview)
- [Microsoft Graph PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview)
