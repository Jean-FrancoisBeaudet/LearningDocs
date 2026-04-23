# Azure CLI — Overview

> **The Azure CLI (`az`) is Microsoft's cross-platform command-line tool for managing [Azure resources](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli).**
> Written in Python, it wraps the Azure Resource Manager (ARM) REST API and a handful of data-plane APIs behind a consistent `az <noun> <verb>` grammar.
> **Chapter index:** [01 Auth & context](01-authentication-and-context.md) · [02 Output & JMESPath](02-output-and-jmespath-queries.md) · [03 Resource mgmt & governance](03-resource-management-and-governance.md) · [15 Cheat sheet](15-cli-cheatsheet.md)
> **Sources:** [What is the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli) · [Install](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) · [Reference](https://learn.microsoft.com/en-us/cli/azure/reference-index)

---

## Table of contents

1. [Mental model](#mental-model)
2. [Install](#install)
3. [Versions, updates, extensions](#versions-updates-extensions)
4. [Config & defaults](#config--defaults)
5. [Shells & hosting](#shells--hosting)
6. [Help discovery](#help-discovery)
7. [Sovereign clouds](#sovereign-clouds)
8. [First-session checklist](#first-session-checklist)

---

## Mental model

- Grammar: `az <group> [<subgroup> ...] <command> [arguments]` — e.g., `az network vnet subnet create ...`.
- Almost every command is either a thin wrapper over an ARM REST call (control plane) or a data-plane call (storage, Key Vault, Cosmos DB). Concept map:

| Surface | Examples | Auth | Notes |
|---|---|---|---|
| **ARM (control plane)** | `az group`, `az vm`, `az network`, `az deployment` | Entra ID token (`az login`) | Returns ARM resource JSON; idempotent PUTs where possible |
| **Data plane** | `az storage blob`, `az keyvault secret`, `az cosmosdb sql container` | Entra ID token OR shared key / SAS / connection string — depends on `--auth-mode` or explicit flags | Each service's own API; behavior is service-specific |

- Outputs are always JSON by default — shape what you get with [`--output` and `--query`](02-output-and-jmespath-queries.md).
- Every command accepts global flags: `--subscription`, `--output`, `--query`, `--debug`, `--verbose`, `--only-show-errors`, `--help`.

## Install

Official install pages: [Windows](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows), [macOS](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-macos), [Linux](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux).

### Windows

```powershell
# MSI (recommended)
winget install --exact --id Microsoft.AzureCLI

# or direct MSI download
Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile AzureCLI.msi
Start-Process msiexec.exe -ArgumentList "/I AzureCLI.msi /quiet" -Wait
```

### macOS

```bash
brew update && brew install azure-cli
```

### Linux

```bash
# Debian/Ubuntu
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# RHEL/Fedora/CentOS
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo dnf install azure-cli

# SUSE / openSUSE
sudo zypper install azure-cli
```

### Docker (ephemeral)

```bash
docker run --rm -it -v ${HOME}/.azure:/root/.azure mcr.microsoft.com/azure-cli
```

Mounting `~/.azure` preserves the MSAL token cache between invocations — otherwise you log in every run.

### Cloud Shell

- Pre-installed at [shell.azure.com](https://shell.azure.com/) and the portal shell icon. Both bash and PowerShell variants ship `az`.
- Backed by a free 5 GB file share in a linked storage account.
- Always reflects the latest GA `az` build — no updates to manage.

## Versions, updates, extensions

```bash
az version           # CLI + extension versions
az upgrade           # self-update (v2.11+); respects your install method
az upgrade --all     # also bumps installed extensions
```

- Release cadence: **roughly every 3 weeks**. Breaking changes are called out in [release notes](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli).
- Pin a specific version for CI by using the `mcr.microsoft.com/azure-cli:<version>` Docker tag.

### Extensions

[Extensions](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview) add commands outside the core CLI (previews, newer services, community tools).

```bash
az extension list                             # installed
az extension list-available --output table    # all published in the index
az extension add --name aks-preview
az extension update --name aks-preview
az extension remove --name aks-preview
```

- Store: `~/.azure/cliextensions/<name>/`.
- Preview vs experimental: extensions show a `[Preview]` or `[Experimental]` badge in help. Preview = API shape may change; Experimental = may be removed entirely.
- Auto-install prompt: some core commands trigger an auto-install prompt the first time you run them (e.g., `az aks command invoke`). Disable with `az config set extension.use_dynamic_install=no`.
- Deep-dive in [05 az rest, extensions, Resource Graph](05-az-rest-extensions-and-graph.md).

## Config & defaults

Config lives in `~/.azure/config` (Linux/macOS) or `%USERPROFILE%\.azure\config` (Windows). Manage via [`az config`](https://learn.microsoft.com/en-us/cli/azure/config) and [`az configure`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-configure).

```bash
az config set core.output=table                       # default output format
az config set core.collect_telemetry=false            # opt out of telemetry
az config set core.no_color=true                      # for log-friendly output
az config set logging.enable_log_file=true            # writes to ~/.azure/logs/
az config set extension.use_dynamic_install=yes_without_prompt

# Per-command defaults — spares you --resource-group on every command
az configure --defaults group=rg-dev location=eastus
az configure --defaults group='' location=''         # clear
```

- Precedence (highest wins): **command-line flag** > **env var (`AZURE_*`)** > **`az configure --defaults`** > **built-in default**.
- Store secrets in env vars only, never in config. `~/.azure/config` is plain text.

## Shells & hosting

| Shell | Notes |
|---|---|
| **bash** | First-class; JMESPath quoting is simplest here |
| **zsh** | Same as bash for `az`; pair with `azure-cli` completion script |
| **PowerShell (pwsh/Windows PowerShell)** | Works fine — mind the [quoting gotchas](02-output-and-jmespath-queries.md#quoting-gotchas) around `--query` |
| **cmd.exe** | Works but painful for `--query`; avoid for non-trivial scripts |
| **[`az interactive`](https://learn.microsoft.com/en-us/cli/azure/interactive-azure-cli)** | REPL with live tab-completion, command preview, param hints, `#[cmd]` shell escape. Install: `az extension add --name interactive`. Great for exploration, not scripting |
| **Azure Cloud Shell** | Browser-hosted bash or pwsh; always latest `az`, MI-backed auth for the logged-in user |

### Tab completion

```bash
# bash / zsh
source /etc/bash_completion.d/azure-cli       # Linux package install
# or
source $(brew --prefix)/etc/bash_completion.d/az  # macOS brew

# PowerShell
Register-ArgumentCompleter -Native -CommandName az -ScriptBlock { ... }   # see link below
```

Completion setup scripts: [bash](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux#enable-tab-completion-on-linux), [PowerShell](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows#enable-tab-completion-on-powershell).

## Help discovery

```bash
az --help                              # top-level groups
az vm --help                           # group help
az vm create --help                    # command help (flags + examples)
az find "create vm with availability zone"   # AI-assisted search across docs
az find vm                             # common commands for a keyword
```

- `az find` queries the [Azure CLI recommender service](https://learn.microsoft.com/en-us/cli/azure/reference-docs-index) — it often surfaces examples faster than Learn search.
- Every `--help` page ends with an **Examples** section copy-pasted from the official reference docs.
- For the full command tree in machine-readable form: [`az --help --output json`](https://learn.microsoft.com/en-us/cli/azure/reference-index).

## Sovereign clouds

[`az cloud`](https://learn.microsoft.com/en-us/cli/azure/cloud) switches the ARM/AD endpoints the CLI targets.

```bash
az cloud list --output table
az cloud set --name AzureUSGovernment       # AzureCloud | AzureChinaCloud | AzureUSGovernment | AzureGermanCloud (retired)
az cloud show --query endpoints
```

- Default: `AzureCloud` (public). Switching clouds does **not** carry your login across — `az login` is per-cloud.
- Each cloud has its own ARM, Graph, and AAD authority endpoints — use `az cloud show` to inspect when scripting against sovereign regions.

## First-session checklist

```bash
az version                              # confirm install
az upgrade                              # latest GA
az login                                # interactive (or SP / MI — see chapter 01)
az account show                         # confirm tenant + subscription
az configure --defaults group=rg-dev    # reduce typing
az config set core.output=jsonc         # pretty-colored JSON for interactive work
```

From here the path is:
- Authentication details: [01 Authentication and context](01-authentication-and-context.md).
- Shaping output: [02 Output and JMESPath queries](02-output-and-jmespath-queries.md).
- Scripting patterns: [06 Scripting and automation](06-scripting-and-automation.md).

## Gotchas

- **MSI installers lag** occasionally behind the latest release on pip. Use the MSI anyway — it handles PATH and Python isolation cleanly.
- **Do not `pip install azure-cli`** into a shared Python env — dependency conflicts with other Azure SDKs are routine. Use the official installers.
- **`az upgrade` requires `core.output=json`** internally; overriding default output to `table` globally can confuse the upgrader — clear it first if upgrade misbehaves.
- **Python runtime baked in**: on Windows/macOS the installer ships its own Python. Don't point `PATH` at a different Python and expect `az` to follow.

## Sources

- [What is the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli)
- [Install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azure CLI reference index](https://learn.microsoft.com/en-us/cli/azure/reference-index)
- [Release notes](https://learn.microsoft.com/en-us/cli/azure/release-notes-azure-cli)
- [Extensions overview](https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview)
- [`az config`](https://learn.microsoft.com/en-us/cli/azure/config) · [`az configure`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-configure) · [`az cloud`](https://learn.microsoft.com/en-us/cli/azure/cloud) · [`az interactive`](https://learn.microsoft.com/en-us/cli/azure/interactive-azure-cli)
