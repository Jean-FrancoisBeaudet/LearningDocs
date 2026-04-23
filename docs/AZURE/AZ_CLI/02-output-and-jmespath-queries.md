# Azure CLI — Output Formats & JMESPath Queries

> **Shape every `az` output with `--output` (format) and `--query` (filter).**
> The pair turns raw ARM JSON into exactly what your eyes, scripts, or pipelines need. Mastering `--query` is the single highest-leverage skill with the CLI.
> **Related:** [00 Overview](00-overview.md) · [06 Scripting & automation](06-scripting-and-automation.md) · [JMESPath spec](https://jmespath.org/specification.html) · [JMESPath tutorial](https://jmespath.org/tutorial.html)
> **Sources:** [Format output](https://learn.microsoft.com/en-us/cli/azure/format-output-azure-cli) · [Query output with JMESPath](https://learn.microsoft.com/en-us/cli/azure/query-azure-cli)

---

## Table of contents

1. [Output formats (`--output`)](#output-formats---output)
2. [JMESPath primer for `--query`](#jmespath-primer-for---query)
3. [Projection & multi-select](#projection--multi-select)
4. [Filtering & flattening](#filtering--flattening)
5. [Functions](#functions)
6. [Recipes](#recipes)
7. [Quoting gotchas](#quoting-gotchas)
8. [Piping to `jq` and PowerShell](#piping-to-jq-and-powershell)
9. [`az`-specific JMESPath deltas](#az-specific-jmespath-deltas)

---

## Output formats (`--output`)

Global flag: `-o` or `--output`. Default can be set via `az config set core.output=<format>` or env `AZURE_CORE_OUTPUT`.

| Format | Use when |
|---|---|
| `json` | Default. Machine-parseable, stable; feed to `jq` or scripts |
| `jsonc` | Same JSON but ANSI-colored — for interactive eyeballing; **do not pipe into parsers** (color codes break JSON) |
| `yaml` / `yamlc` | Shorter than JSON; same shape; good for copy into pipeline templates |
| `table` | Flattens to an ASCII table. Respects `--query` projection — curate columns first, then `table` |
| `tsv` | Tab-separated, no header, **no quoting** — pipes cleanly into bash/awk/xargs |
| `none` | Suppress stdout. Useful for fire-and-forget when you only care about exit code |

Rules of thumb:

```bash
az vm list -o table                                           # eyeballing — noisy default columns
az vm list --query "[].{name:name,rg:resourceGroup}" -o table # curated table
az vm list --query "[].id" -o tsv | xargs -n1 az vm delete --ids   # feed IDs to another command
az group show -n rg-dev -o jsonc                              # pretty colored JSON, full detail
az keyvault secret show ... --query value -o tsv              # single scalar into a bash var
```

## JMESPath primer for `--query`

[JMESPath](https://jmespath.org/specification.html) is a path + filter + transform language for JSON. The CLI passes your JSON output through the JMESPath engine before handing the result to the formatter.

| Syntax | Meaning | Example |
|---|---|---|
| `foo.bar` | Object member | `location` |
| `[0]` | Array index | `[0]` |
| `[-1]` | Last element | `[-1]` |
| `[]` | Flatten one level | `[].name` |
| `[*]` | Project over every element | `[*].name` (same as `[].name` for top arrays) |
| `[?expr]` | Filter array by boolean expr | `[?provisioningState=='Succeeded']` |
| `\|` | Pipe — forces left-side into scalar/list context | `[*].tags \| [0]` |
| `{k1:v1,k2:v2}` | Multi-select hash (rename/reshape) | `{name:name, rg:resourceGroup}` |
| `[v1,v2]` | Multi-select list | `[name, location]` |

All of these can be combined. `--query` takes exactly one expression, always quoted.

## Projection & multi-select

```bash
# Just names, flat list
az vm list --query "[].name" -o tsv

# Rename two fields for a table
az vm list --query "[].{Name:name, RG:resourceGroup, Size:hardwareProfile.vmSize}" -o table

# Pull nested values, keeping only the ones you need
az storage account list --query "[].{acct:name, tls:minimumTlsVersion, pub:publicNetworkAccess}" -o table
```

- Keys on the left side of `:` become table headers exactly as typed — use `PascalCase` or `Title Case` strings (quoted) for presentation:

```bash
az vm list --query "[].{'VM Name':name, 'Resource Group':resourceGroup}" -o table
```

## Filtering & flattening

```bash
# All VMs in a specific RG — filter at JMESPath, not shell
az vm list --query "[?resourceGroup=='RG-DEV']" -o table

# Succeeded VMs only
az vm list --query "[?provisioningState=='Succeeded'].name" -o tsv

# Multiple conditions: && and ||
az vm list --query "[?location=='eastus' && powerState=='VM running'].name" -o tsv

# Substring match via contains()
az resource list --query "[?contains(type, 'Microsoft.Storage')].name" -o tsv

# Starts-with / ends-with
az group list --query "[?starts_with(name, 'rg-prod')].name" -o tsv
```

- `[?...]` returns a filtered **list** — empty `[]` when nothing matches (exit code still 0).
- JMESPath is strongly typed — `==`, `!=`, `<`, `<=`, `>`, `>=`, `&&`, `||`, `!` (prefix not).
- String literals inside `--query` use **single quotes** in JMESPath; the outer `--query "..."` uses **double quotes** (bash/pwsh permitting).

### Flattening

```bash
# NSG list is per-RG → flatten into a single list of rules
az network nsg list --query "[].{nsg:name, rg:resourceGroup}" -o table

# Sub-rules: flatten via [] after the rule array
az network nsg list \
  --query "[].securityRules[].{nsg:id, rule:name, prio:priority, access:access}" \
  -o table
```

## Functions

[Built-in JMESPath functions](https://jmespath.org/specification.html#built-in-functions) plus a few CLI-specific helpers:

| Function | Purpose | Example |
|---|---|---|
| `length(x)` | Count | `length(@)` = total items |
| `sort(arr)` / `sort_by(arr, &key)` | Sort | `sort_by([], &name)` |
| `reverse(arr)` | Reverse | `reverse(sort(@))` |
| `min_by` / `max_by` | Pick extreme by key | `max_by([], &createdAt).name` |
| `keys(obj)` / `values(obj)` | Introspect object | `keys(tags)` |
| `contains(container, x)` | Substring / membership | `contains(name, 'prod')` |
| `starts_with` / `ends_with` | String prefix/suffix | `starts_with(name, 'rg-')` |
| `to_string(x)` / `to_number(x)` | Coerce | `to_string(tags.costcenter)` |
| `join(sep, list)` | Stringify arrays | `join(',', [].name)` |
| `map(&expr, arr)` | Transform each | `map(&tags.env, [])` |
| `not_null(a, b, ...)` | First non-null | `not_null(tags.env, 'untagged')` |

```bash
# Count VMs per location
az vm list --query "[].location" -o tsv | sort | uniq -c

# Total storage accounts in the sub
az storage account list --query "length(@)"

# Sort VMs by size then take 5 smallest
az vm list --query "sort_by([], &hardwareProfile.vmSize)[:5].{name:name, size:hardwareProfile.vmSize}" -o table
```

## Recipes

### Find all untagged resources

```bash
az resource list --query "[?tags==null || tags.environment==null].{name:name,type:type,rg:resourceGroup}" -o table
```

### Top 10 most expensive VM SKUs used

```bash
az vm list --query "[].hardwareProfile.vmSize" -o tsv | sort | uniq -c | sort -rn | head
```

### Export all SPs and their owners

```bash
az ad sp list --all \
  --query "[].{appId:appId, display:displayName, signInAudience:signInAudience}" \
  -o json > sps.json
```

### IPs from all public load balancers

```bash
az network public-ip list \
  --query "[?publicIPAllocationMethod=='Static'].{name:name, ip:ipAddress, rg:resourceGroup}" \
  -o table
```

### Key Vaults missing purge protection

```bash
az keyvault list --query "[?properties.enablePurgeProtection==null].name" -o tsv
```

### Storage accounts with shared-key access enabled (bad)

```bash
az storage account list --query "[?allowSharedKeyAccess==\`true\`].{name:name,rg:resourceGroup}" -o table
```

Note the **backticks** `\`true\`` — literal JSON booleans in JMESPath must be backticked. Inside a bash double-quoted string escape them with `\`.

### Get a secret value straight into a bash variable

```bash
TOKEN=$(az keyvault secret show \
  --vault-name kv-prod --name api-token \
  --query value -o tsv)
```

## Quoting gotchas

JMESPath string literals use **single quotes**. The shell uses either — so pick the outer-quote convention that avoids escaping.

### bash / zsh

```bash
# Preferred: double-quote outer, single-quote JMESPath strings
az vm list --query "[?resourceGroup=='RG-DEV'].name" -o tsv
```

### PowerShell (pwsh / Windows PowerShell)

PowerShell eats single quotes differently — safest is the **single-outer / double-inner** swap, or the `'...'` here-string:

```powershell
# Single quotes preserve the string as-is
az vm list --query '[?resourceGroup==''RG-DEV''].name' -o tsv

# Or using the stop-parsing token --% for the entire tail
az --% vm list --query "[?resourceGroup=='RG-DEV'].name" -o tsv
```

- The `--%` stop-parsing token (PowerShell-only) hands the rest of the line to the native exe verbatim — the cleanest option for complex queries.
- Avoid backticks as line continuations mid-query in pwsh; variable expansion rules bite.

### cmd.exe

Double-quote outer, escape inner double quotes with `\"`, and avoid `!` anywhere (delayed expansion). Life is too short — switch to pwsh.

### Backticks for literals

JMESPath backticks denote **literal JSON**:

```bash
--query "[?enabled==\`true\`]"           # boolean true, not the string 'true'
--query "[?sku.capacity==\`3\`]"         # integer 3
--query "[?tags.env==\`null\`]"          # equals null literal
```

Escape them per your shell (`\\\`` in double-quoted bash strings when nested inside `$(...)`, etc.). When in doubt, write the query in a file and pass `--query @file.jmespath`:

```bash
az vm list --query @vm-query.jmespath -o table
```

## Piping to `jq` and PowerShell

JMESPath handles ~90% of shaping but `jq` has more firepower for deep transformations. Keep `--query` for the **first filter** (reduce payload size) then hand off:

```bash
az vm list -o json | jq '.[] | {name, nic: .networkProfile.networkInterfaces[0].id}'
```

PowerShell equivalent:

```powershell
az vm list -o json `
  | ConvertFrom-Json `
  | Select-Object name, @{n='RG';e={$_.resourceGroup}}, @{n='NIC';e={$_.networkProfile.networkInterfaces[0].id}}
```

- `ConvertFrom-Json` returns `[PSCustomObject]`; property access uses dot notation, filtering uses `Where-Object`.
- For very large outputs (>1 MB JSON), `-o tsv` + `awk` is faster than `ConvertFrom-Json`.

## `az`-specific JMESPath deltas

The CLI uses a slightly extended JMESPath engine ([`jmespath-ext`](https://pypi.org/project/jmespath-ext/)). Notable extensions:

- `not_null(a, b, ...)` — first non-null arg (standard JMESPath `||` short-circuit-returns falsy values, not strictly non-null).
- `@` is the current node, same as standard JMESPath.
- Multi-select hashes support **string keys with spaces** when quoted: `{'Vm Name':name}`.
- Array slicing works like Python: `[1:5]`, `[:5]`, `[::-1]`.

Things **not** supported:

- `let` bindings.
- Custom functions (you can't register your own).
- Regex — use `contains`, `starts_with`, `ends_with`, or pipe to `jq`/`grep`.

## Gotchas

- **`--query` runs before `--output`**. If `-o table` yields an empty table, check whether the query already filtered everything out — `-o json` will reveal an empty `[]`.
- **Tables flatten only one level.** If your query leaves nested objects, `table` prints `null` or Python-repr strings. Multi-select down to scalars first.
- **`--query` cannot add synthetic fields beyond what's in the response.** For cross-resource joins (e.g., VM + its NIC IP), issue two commands and join in jq / pwsh.
- **`-o tsv` with multi-value fields**: tabs separate columns; newlines inside string values (rare but possible in tags) break the format. Sanitize with `tr -d '\n\t'` if piping.
- **Commands that return a single object vs array**: `az group show` returns an object; `az group list` returns an array. Your query must match (`name` vs `[].name`).

## Sources

- [Format output](https://learn.microsoft.com/en-us/cli/azure/format-output-azure-cli)
- [Query output with JMESPath](https://learn.microsoft.com/en-us/cli/azure/query-azure-cli)
- [JMESPath specification](https://jmespath.org/specification.html) · [JMESPath tutorial](https://jmespath.org/tutorial.html)
- [Use Bash with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-bash)
- [Use PowerShell with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-powershell)
