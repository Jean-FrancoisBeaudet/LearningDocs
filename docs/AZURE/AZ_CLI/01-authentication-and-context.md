# Azure CLI — Authentication & Context

> **How `az` acquires Entra ID tokens, and how to point it at the right tenant and subscription.**
> Covers interactive login, service principals, federated OIDC, managed identity, device code, `az account` context switching, and token inspection.
> **Related:** [00 Overview](00-overview.md) · [04 Identity & RBAC commands](04-identity-and-rbac-commands.md) · [06 Scripting & automation](06-scripting-and-automation.md) · [ENTRAID — service principals](../ENTRAID/02-app-registrations-and-service-principals.md) · [ENTRAID — managed identities](../ENTRAID/04-managed-identities.md)
> **Sources:** [Sign in with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli) · [`az login`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-login) · [`az account`](https://learn.microsoft.com/en-us/cli/azure/account)

---

## Table of contents

1. [How auth works end-to-end](#how-auth-works-end-to-end)
2. [Interactive login](#interactive-login)
3. [Device code flow](#device-code-flow)
4. [Service principal auth](#service-principal-auth)
5. [Federated OIDC (GitHub Actions / Azure DevOps)](#federated-oidc-github-actions--azure-devops)
6. [Managed identity](#managed-identity)
7. [Tenants & subscriptions (`az account`)](#tenants--subscriptions-az-account)
8. [Tokens: inspecting & using directly](#tokens-inspecting--using-directly)
9. [Env vars & credential precedence](#env-vars--credential-precedence)
10. [Troubleshooting](#troubleshooting)

---

## How auth works end-to-end

1. `az login` (or equivalent) talks to the [Microsoft identity platform](https://learn.microsoft.com/en-us/entra/identity-platform/v2-overview) and receives an access token + refresh token via [MSAL for Python](https://learn.microsoft.com/en-us/entra/msal/python/).
2. Tokens are cached in `~/.azure/msal_token_cache.json` (encrypted on Windows via DPAPI; keychain on macOS; plaintext on Linux unless `--encrypt-token-cache` is set).
3. Every subsequent `az ...` call fetches a scoped token from cache — refreshing via MSAL when expired — and sends it as `Authorization: Bearer <jwt>` to ARM (or the target data-plane service).
4. Context — **which tenant and subscription you're acting in** — is separate from the token and stored in `~/.azure/azureProfile.json`; you toggle it with [`az account set`](https://learn.microsoft.com/en-us/cli/azure/account#az-account-set).

Token cache operations:

```bash
az account clear                        # wipe all subscriptions + token cache
az logout                               # sign out the current account only
az account get-access-token --help      # see token commands
```

## Interactive login

Default for laptops/workstations. Opens a browser tab against `https://login.microsoftonline.com`.

```bash
az login
az login --tenant contoso.onmicrosoft.com    # specify tenant explicitly
az login --tenant 11111111-...-1111          # or by GUID
az login --scope https://graph.microsoft.com/.default  # pre-consent a data-plane scope
```

- If you're a guest in multiple tenants, `az login --tenant ...` is often required — else you land in your home tenant and see no subscriptions.
- Interactive login respects Conditional Access (MFA, compliant-device, etc.) — the browser experience handles prompts.
- Returns a summary table of all subscriptions visible to the signed-in account. The first isDefault one becomes the active subscription.

## Device code flow

For headless machines, SSH sessions, and remote containers where the CLI can't open a browser.

```bash
az login --use-device-code
# prints: "To sign in, use a web browser to open https://microsoft.com/devicelogin and enter the code ABC-XYZ"
```

- User code expires in 15 minutes.
- Often required for **remote dev boxes, WSL if no browser bridge, CI debugging, Cloud Shell → external tenant**.
- Same CA policies apply — a compliant device policy may still block the flow from an unmanaged host.

## Service principal auth

For scripts, CI/CD without OIDC, and long-running daemons. Backed by an [application registration and service principal](../ENTRAID/02-app-registrations-and-service-principals.md). See also [`az ad sp create-for-rbac`](https://learn.microsoft.com/en-us/cli/azure/ad/sp#az-ad-sp-create-for-rbac).

### Password (client secret)

```bash
az login \
  --service-principal \
  --username <appId> \
  --password <clientSecret> \
  --tenant <tenantId>
```

- **Rotate secrets**. Max secret lifetime is 2 years per Entra defaults, configurable via [app management policy](https://learn.microsoft.com/en-us/entra/identity-platform/reference-app-management-policy).
- Never commit the secret. Pull from Key Vault, env, or CI secret store.

### Certificate

```bash
az login \
  --service-principal \
  --username <appId> \
  --tenant <tenantId> \
  --password /path/to/cert.pem         # PEM with private key + cert concatenated
```

- Preferred over passwords — can be stored in Key Vault, rotated via `az ad sp credential reset --cert`.
- `.pfx` works too but must be unlocked (`openssl pkcs12 -in file.pfx -out cert.pem -nodes`).

### Create & grant in one shot

```bash
az ad sp create-for-rbac \
  --name sp-deploy-dev \
  --role Contributor \
  --scopes /subscriptions/<subId>/resourceGroups/rg-dev \
  --years 1
```

- Output includes `appId`, `password`, `tenant` — treat the `password` like a one-time reveal.
- Prefer a narrower role than `Contributor` where possible (e.g., `Storage Blob Data Contributor` on the specific account).

## Federated OIDC (GitHub Actions / Azure DevOps)

The modern, passwordless CI pattern — exchanges a workload's identity token for an Entra ID access token via [workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation).

### GitHub Actions

```yaml
permissions:
  id-token: write         # required — GH issues the OIDC JWT
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          # no client-secret — OIDC handles it
      - run: az group list -o table
```

Register the federated credential on the SP:

```bash
az ad app federated-credential create \
  --id <appObjectId> \
  --parameters '{
    "name": "gh-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Azure DevOps

Use the [Azure Resource Manager service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints) with **Workload identity federation (automatic)** — no secret to rotate, ADO creates the federated credential for you.

## Managed identity

[Managed identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) lets an Azure-hosted workload (VM, VMSS, App Service, AKS, Functions, Container Apps) authenticate to ARM without storing any secret.

```bash
# System-assigned MI on the current Azure host
az login --identity

# User-assigned MI (by clientId, objectId, or resource ID)
az login --identity --username <userAssignedMIClientId>
```

- Works anywhere the [Instance Metadata Service (IMDS)](https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service) is reachable (`169.254.169.254`).
- Fails fast if you run it on a laptop — there is no IMDS locally.
- Token audience is ARM by default; for a data-plane call (e.g., Key Vault) pass `--scope` to `az account get-access-token`.

## Tenants & subscriptions (`az account`)

```bash
az account list --output table           # all subs the signed-in identity can see
az account show                          # current active sub/tenant
az account set --subscription "Dev/Test - East US"   # switch by name or ID
az account set --subscription 11111111-2222-3333-4444-555555555555
az account list-locations --output table # regions available for current sub
az account tenant list                   # tenants the identity is a member/guest of
```

- **`--subscription` as a one-shot**: `az vm list --subscription "Prod"` overrides the active sub for one command without calling `az account set`.
- **Multi-tenant workflow**: `az login --tenant <tenantA>`, then `az login --tenant <tenantB>` layers sessions. `az account list` shows both; `az account set` moves between them.

### Management groups

Roll up subscriptions under [management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview):

```bash
az account management-group list
az account management-group show --name mg-landing-zones --expand --recurse
```

Deployments/Policy assignments at the MG scope: see [03 Resource management & governance](03-resource-management-and-governance.md).

## Tokens: inspecting & using directly

```bash
az account get-access-token                                           # ARM token (default)
az account get-access-token --resource https://vault.azure.net        # Key Vault token
az account get-access-token --resource https://graph.microsoft.com    # MS Graph token
az account get-access-token --query accessToken -o tsv                # just the JWT string

# Decode the JWT payload with jq (no signature check)
TOKEN=$(az account get-access-token --query accessToken -o tsv)
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

Common claims:

| Claim | Meaning |
|---|---|
| `aud` | Audience — must match target API (`https://management.azure.com/`) |
| `iss` | Issuer — `https://sts.windows.net/<tenantId>/` |
| `appid` / `oid` | App + object IDs for SP/MI tokens; `upn` + `oid` for users |
| `roles` | App roles granted to the SP |
| `scp` | Delegated scopes (user tokens only) |
| `exp` | Epoch expiry — typically 60-90 minutes |

Use cases for raw tokens: feeding [`az rest`](05-az-rest-extensions-and-graph.md#az-rest-the-escape-hatch) against undocumented APIs, authenticating `curl` during debugging, passing to non-`az` tooling (Kubernetes `kubelogin`, `kubectl` exec plugins, Terraform `ARM_ACCESS_TOKEN`).

## Env vars & credential precedence

The CLI honors these env vars — most of them are also what `DefaultAzureCredential` in the Azure SDKs uses, keeping parity between scripts and app code.

| Env var | Purpose |
|---|---|
| `AZURE_CLIENT_ID` | SP / user-assigned MI client ID |
| `AZURE_CLIENT_SECRET` | SP password |
| `AZURE_CLIENT_CERTIFICATE_PATH` | Path to PEM cert (SP cert auth) |
| `AZURE_TENANT_ID` | Entra tenant |
| `AZURE_SUBSCRIPTION_ID` | Default subscription for `az` |
| `AZURE_FEDERATED_TOKEN_FILE` | OIDC token file (used by `azure/login@v2` etc.) |
| `AZURE_HTTP_USER_AGENT` | Append custom UA suffix for audit trails |
| `AZURE_CONFIG_DIR` | Override `~/.azure` location (useful in CI) |
| `AZURE_CORE_OUTPUT` | Default `--output` format |
| `AZURE_CORE_ONLY_SHOW_ERRORS` | `true` silences warnings & info messages |
| `REQUESTS_CA_BUNDLE` | Point at corp CA bundle behind SSL inspection |

Effective auth identity precedence inside the CLI:

1. `--subscription` / explicit flags on a specific command
2. `az account set` current selection
3. Env vars (`AZURE_SUBSCRIPTION_ID` etc.)
4. The first subscription in `~/.azure/azureProfile.json`

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `AADSTS50076: MFA required` on SP login | SP isn't a user — MFA shouldn't trigger. Check Conditional Access — a "require MFA for all users" policy without an SP exclusion will block. |
| `No subscriptions found` after `az login` | You landed in the wrong tenant. Re-run `az login --tenant <tenantId>`. |
| `AADSTS700016: Application ... not found in directory` | Wrong tenant for the SP, or the app was deleted. Verify with `az ad sp show --id <appId>`. |
| `az login --identity` fails with `MSI: Failed to retrieve token` | Not on an Azure host, or MI not assigned to this resource. On AKS use [workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) instead of IMDS. |
| `SSL: CERTIFICATE_VERIFY_FAILED` | Corporate TLS inspection. Point `REQUESTS_CA_BUNDLE` at the corp CA chain; for Python urllib3, `export AZURE_CLI_DISABLE_CONNECTION_VERIFICATION=1` is a last-resort debug hack — never in CI. |
| Tokens expire mid-script | Expected; MSAL refreshes on next `az` call. For long `--no-wait` loops, refresh proactively with `az account get-access-token`. |
| `az login` hangs on WSL | No browser bridge. Use `az login --use-device-code`. |

### Verbose auth diagnostics

```bash
az login --debug 2>login.log
grep -iE "msal|token|tenant|wwwauthenticate" login.log
```

- `--debug` dumps full HTTP traces including redirected auth endpoints — invaluable for CA / federated-creds debugging.
- Never paste `--debug` logs into tickets without redacting — they contain tokens and tenant IDs.

## Gotchas

- **`az login` with `--allow-no-subscriptions`** is needed if the identity has Entra roles but zero Azure RBAC assignments — otherwise the CLI refuses to complete login.
- **Token cache lock on Windows**: running multiple `az` processes in parallel can race on `msal_token_cache.json`. Serialize or set `AZURE_CONFIG_DIR` per-worker in CI.
- **Tenant boundary != subscription boundary**: you can have subs in tenant A and be invited as guest to tenant B. `az account list --all` shows both; `az account set` moves between.
- **Cloud Shell auth is not portable**: the token cache inside Cloud Shell is bound to your user's `clouddrive`; `scp`-ing config files out doesn't migrate login state to a laptop.

## Sources

- [Sign in with the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
- [`az login`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-login) · [`az logout`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-logout) · [`az account`](https://learn.microsoft.com/en-us/cli/azure/account)
- [Create an SP with the CLI](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1)
- [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Managed identity overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)
- [MSAL Python](https://learn.microsoft.com/en-us/entra/msal/python/)
