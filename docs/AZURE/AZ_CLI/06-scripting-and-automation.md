# Azure CLI — Scripting & Automation

> **Production-grade scripting with `az`: idempotency, error handling, async polling, CI/CD integration, secret hygiene, and debugging.**
> **Related:** [00 Overview](00-overview.md) · [01 Auth & context](01-authentication-and-context.md) · [02 Output & JMESPath](02-output-and-jmespath-queries.md) · [05 az rest, extensions, graph](05-az-rest-extensions-and-graph.md)
> **Sources:** [Use Bash with the CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-bash) · [Use PowerShell with the CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-powershell) · [Tips for troubleshooting](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-troubleshooting)

---

## Table of contents

1. [Bash patterns](#bash-patterns)
2. [PowerShell patterns](#powershell-patterns)
3. [Idempotency](#idempotency)
4. [Async operations (`--no-wait` + `wait`)](#async-operations---no-wait--wait)
5. [Error handling & exit codes](#error-handling--exit-codes)
6. [Debugging (`--debug`, `--verbose`)](#debugging---debug---verbose)
7. [Secret hygiene](#secret-hygiene)
8. [CI/CD — GitHub Actions](#cicd--github-actions)
9. [CI/CD — Azure DevOps](#cicd--azure-devops)
10. [Retry patterns](#retry-patterns)

---

## Bash patterns

### Strict mode preamble

Every non-trivial Azure CLI script should start with:

```bash
#!/usr/bin/env bash
set -euo pipefail   # -e: exit on error, -u: undefined vars, -o pipefail: pipe errors propagate

# Optional but valuable:
export AZURE_CORE_OUTPUT=json      # force JSON regardless of user's config
export AZURE_CORE_NO_COLOR=1       # strip ANSI for logging
```

- Without `-o pipefail`, `az ... | jq ...` masks `az` failures — pipeline returns `jq`'s exit code.
- Trap errors for context:

```bash
trap 'echo "FAIL at line $LINENO: $BASH_COMMAND" >&2' ERR
```

### Variable capture

```bash
# Single scalar
VM_IP=$(az vm show \
  --resource-group rg-dev --name vm-web \
  --query publicIps --show-details -o tsv)

# Array of IDs
mapfile -t VM_IDS < <(az vm list --query "[].id" -o tsv)
echo "Found ${#VM_IDS[@]} VMs"
for id in "${VM_IDS[@]}"; do echo "$id"; done

# Pipe-friendly IDs
az vm list --query "[].id" -o tsv \
  | xargs -n1 -I{} az vm deallocate --ids {} --no-wait
```

- `xargs -n1` runs the command once per ID — `-P <N>` parallelizes.
- `--show-details` on `az vm show` denormalizes related resources (NIC, public IP) into the VM JSON — saves extra lookups.

### Temp files for complex payloads

```bash
PAYLOAD=$(mktemp)
trap 'rm -f "$PAYLOAD"' EXIT
cat > "$PAYLOAD" <<'JSON'
{
  "properties": { ... }
}
JSON
az rest --method PUT --url "$URL" --body "@$PAYLOAD"
```

Heredocs keep secrets out of `ps` output (visible vs passing on the command line).

## PowerShell patterns

### Strict mode preamble

```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$env:AZURE_CORE_OUTPUT = 'json'
$env:AZURE_CORE_NO_COLOR = '1'
```

- `$ErrorActionPreference = 'Stop'` turns PowerShell warnings into terminating errors for **cmdlets**. For native `az`, you still need to check `$LASTEXITCODE`.

### Variable capture

```powershell
# JSON → object
$vm = az vm show -g rg-dev -n vm-web --show-details | ConvertFrom-Json
$vm.publicIps

# Collection (array)
$ids = az vm list --query "[].id" -o tsv
$ids -split "`n" | ForEach-Object { az vm deallocate --ids $_ --no-wait }
```

### Error checks on native calls

```powershell
az group create -n rg-dev -l eastus
if ($LASTEXITCODE -ne 0) { throw "az group create failed ($LASTEXITCODE)" }
```

Wrap it once:

```powershell
function Invoke-Az {
  param([Parameter(ValueFromRemainingArguments=$true)] [string[]]$Args)
  $out = az @Args
  if ($LASTEXITCODE -ne 0) { throw "az $($Args -join ' ') -> $LASTEXITCODE" }
  return $out
}

$groups = Invoke-Az group list --query '[].name' -o tsv
```

### Avoid `2>&1` around native `az`

In Windows PowerShell 5.1, `az ... 2>&1` wraps stderr lines in `ErrorRecord` and sets `$?` to false even on exit code 0. Leave stderr alone — capture stdout only:

```powershell
$output = az group list -o json     # stderr flows to console
```

## Idempotency

Most `create` commands are **not** idempotent — they fail if the target exists. Strategies:

### Check-then-create

```bash
if ! az group show -n rg-dev >/dev/null 2>&1; then
  az group create -n rg-dev -l eastus
fi
```

### Upsert via `update` flags

For resources that support it — `az vm run-command invoke`, `az webapp config set`, `az resource update` — the semantics are merge-style PATCH.

### Declarative via Bicep (recommended)

Any mutation needing idempotency belongs in Bicep; `az deployment group create` re-runs are safe by design (Incremental mode).

```bash
az deployment group create \
  --name refresh-network \
  --resource-group rg-dev \
  --template-file network.bicep \
  --parameters env=dev
```

### `--if-none-match` patterns

Some resources expose optimistic-concurrency headers (ETags). Use via `az rest --headers 'If-None-Match=*'` for conditional create.

## Async operations (`--no-wait` + `wait`)

Long-running operations (LRO) — VM create, AKS upgrade, deployment — default to blocking until complete. `--no-wait` returns immediately once ARM accepts the request; pair with a `wait` sub-command:

```bash
# Fire
az aks create -g rg-dev -n aks-prod --node-count 3 --no-wait

# Later — poll until done
az aks wait -g rg-dev -n aks-prod --created --timeout 1800

# Other conditions
az group wait -n rg-dev --deleted
az vm wait -g rg-dev -n vm-web --updated
az vm wait -g rg-dev -n vm-web --custom "instanceView.statuses[?code=='PowerState/running']"
```

`wait` conditions: `--created`, `--deleted`, `--updated`, `--exists`, `--custom "<JMESPath>"` (returns when the expression evaluates truthy).

### Parallelize with `--no-wait`

```bash
for rg in rg-dev rg-qa rg-stage; do
  az deployment group create \
    --resource-group "$rg" \
    --template-file main.bicep \
    --parameters env="${rg##rg-}" \
    --no-wait
done

# Then wait on all three
for rg in rg-dev rg-qa rg-stage; do
  az deployment group wait --resource-group "$rg" --name main --created
done
```

## Error handling & exit codes

Azure CLI exit codes:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Generic CLI error (validation, auth, resource not found, HTTP error) |
| `2` | Parser / argument error |
| `3` | `az … wait` condition not met before `--timeout` |

- Exit code `1` lumps transport errors (401, 403, 404, 409, 429, 5xx) — check stderr or `--debug` for the actual ARM code.
- Empty responses on filtered queries are **not** errors (`az vm list --query "[?name=='nothing']"` returns `[]` with exit 0). Test with JMESPath `length` or `jq empty`.
- Throttling: ARM returns HTTP 429 with a `Retry-After` header; the CLI retries with backoff by default.

### Inspect the ARM error body

```bash
az vm create ... 2> /tmp/err || {
  cat /tmp/err                             # full error text
  grep -oE '"code":"[^"]+' /tmp/err        # just the ARM error code
}
```

### Force warnings-as-errors for CI

```bash
az group create ... --only-show-errors   # suppress info/warnings
# Combine with set -e so any non-zero exit aborts the pipeline.
```

## Debugging (`--debug`, `--verbose`)

| Flag | What it adds |
|---|---|
| `--verbose` | Request IDs, URL, response summary, timing |
| `--debug` | Everything in `--verbose` + full HTTP request/response headers and bodies, token redacted |
| `AZURE_CORE_COLLECT_TELEMETRY=false` | Opt out of anonymized telemetry |

```bash
az vm create --debug 2> debug.log
grep -E "(Method|Url|StatusCode|x-ms-correlation)" debug.log
```

- `x-ms-correlation-request-id` and `x-ms-request-id` are the ARM request identifiers — include them in support tickets.
- `--debug` logs tokens truncated but still sensitive enough to avoid pasting raw. Scrub before sharing.
- `az config set logging.enable_log_file=true` persists `--debug`-level output under `~/.azure/logs/` without re-running.

## Secret hygiene

```bash
# NEVER
az login --service-principal -u $APP_ID -p "supers3cret!" ...   # visible in process list

# BETTER: env var (mind `export` visibility in multi-user shells)
export AZURE_CLIENT_SECRET="..."
az login --service-principal -u "$AZURE_CLIENT_ID" -p "$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID"

# BEST: no secret at all — OIDC federation or managed identity (01-authentication chapter)

# Pull from Key Vault at runtime
DBPW=$(az keyvault secret show --vault-name kv-prod --name db-pw --query value -o tsv)
```

Do not:
- Log `--debug` output with tokens in CI logs (mask via the CI's secret marking feature).
- Commit `.azure/*.json` to git — add `.azure/` to `.gitignore`.
- Echo `$AZURE_CLIENT_SECRET` or the output of `az ad sp credential reset`.

Do:
- Use GitHub Actions' `::add-mask::` directive or Azure DevOps' `issecret=true` to mask values derived at runtime.
- Prefer user-assigned MI for long-lived workloads, SP + federated creds for CI.

## CI/CD — GitHub Actions

```yaml
name: deploy

on:
  push: { branches: [main] }

permissions:
  id-token: write     # required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Bicep what-if
        run: |
          az deployment group what-if \
            --resource-group rg-prod \
            --template-file main.bicep

      - name: Deploy
        run: |
          az deployment group create \
            --resource-group rg-prod \
            --template-file main.bicep \
            --parameters env=prod \
            --only-show-errors
```

- [`azure/login@v2`](https://github.com/Azure/login) handles OIDC and installs `az` on hosted runners (pre-installed; action just authenticates).
- Pin a CLI version on hosted runners via `azure/cli@v2` with the `azcliversion` input if you need reproducibility.
- Use environments + required reviewers for prod gates.

## CI/CD — Azure DevOps

### Classic AzureCLI task

```yaml
- task: AzureCLI@2
  displayName: 'Deploy infra'
  inputs:
    azureSubscription: 'svc-conn-prod'        # ADO service connection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az deployment group create \
        --resource-group rg-prod \
        --template-file main.bicep \
        --parameters env=prod
    useGlobalConfig: true
    addSpnToEnvironment: true                 # exposes $servicePrincipalId, $idToken, $tenantId
```

- `addSpnToEnvironment: true` exposes the federated ID token to your script — handy for re-authenticating tools that need their own `az login`.
- Service connection of type *Workload Identity Federation* = passwordless; manage the federated credential via Entra or let ADO auto-create/refresh it.

## Retry patterns

### Bash

```bash
retry() {
  local max=${1:-5}; shift
  local i=0 delay=2
  until "$@"; do
    ((i++)); (( i > max )) && return 1
    echo "Retry $i/$max after ${delay}s: $*" >&2
    sleep "$delay"
    delay=$(( delay * 2 ))
  done
}

retry 5 az storage account create -g rg-dev -n stgdev1 -l eastus --sku Standard_LRS
```

### PowerShell

```powershell
function Invoke-WithRetry {
  param([ScriptBlock]$Script, [int]$Max = 5)
  $delay = 2
  for ($i = 1; $i -le $Max; $i++) {
    try { return & $Script } catch {
      if ($i -eq $Max) { throw }
      Start-Sleep -Seconds $delay
      $delay *= 2
    }
  }
}

Invoke-WithRetry { az storage account create -g rg-dev -n stgdev1 -l eastus --sku Standard_LRS }
```

- The CLI already retries transient 5xx and 429 responses internally (built-in urllib3 retry). User-level retry loops handle **logical** retry: eventual-consistency gaps (SP just created, role assignment races), post-deletion tombstone windows, etc.
- Avoid retrying on exit code `2` (argument errors) — they are deterministic.

## Gotchas

- **Pipelines mask real failures**: `az ... | jq ... | grep ...` drops any `az` error unless `set -o pipefail`.
- **Running multiple `az` calls concurrently on the same machine** can race on `~/.azure/msal_token_cache.json` — either serialize or give each process its own `AZURE_CONFIG_DIR`.
- **`AZURE_EXTENSION_DIR` and `AZURE_CONFIG_DIR`** are independent — set both in containerized CI to isolate per-job state.
- **`az configure --defaults`** persists in `~/.azure/config` — surprising if your next CI run on the same runner inherits it. Prefer explicit flags in CI.
- **PowerShell `$?` is unreliable for native `az`** — always check `$LASTEXITCODE`.
- **Bash `set -e` does not exit on failures inside `$(...)` substitution within a larger expression** — e.g., `echo "$(az ... || true)"` masks the error. Capture separately and check.
- **`az login --identity` in AKS needs [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/)** (IMDS isn't available to pods by design). Transitional: `az login --identity` from the VM/VMSS node-pool host is cluster-level, not per-pod.

## Sources

- [Use Bash with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-bash)
- [Use PowerShell with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-powershell)
- [Tips for troubleshooting](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-troubleshooting)
- [Async operations and `--no-wait`](https://learn.microsoft.com/en-us/cli/azure/azure-cli-reference-for-long-running-operations)
- [`azure/login` action](https://github.com/Azure/login) · [Azure CLI task for ADO](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-cli-v2)
- [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
