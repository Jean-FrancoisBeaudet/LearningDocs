# Azure CLI — Identity & RBAC Commands

> **`az ad` for Entra directory objects, `az role` for Azure RBAC, `az identity` for user-assigned managed identities.**
> Covers users, groups, app registrations, service principals, federated credentials, role definitions and assignments (including ABAC conditions), and MI assignment to workloads.
> **Related:** [01 Authentication & context](01-authentication-and-context.md) · [ENTRAID — app regs & SPs](../ENTRAID/02-app-registrations-and-service-principals.md) · [ENTRAID — managed identities](../ENTRAID/04-managed-identities.md) · [ENTRAID — RBAC](../ENTRAID/05-rbac-and-authorization.md)
> **Sources:** [`az ad`](https://learn.microsoft.com/en-us/cli/azure/ad) · [`az role`](https://learn.microsoft.com/en-us/cli/azure/role) · [`az identity`](https://learn.microsoft.com/en-us/cli/azure/identity)

---

## Table of contents

1. [`az ad` surface](#az-ad-surface)
2. [Users](#users)
3. [Groups](#groups)
4. [App registrations](#app-registrations)
5. [Service principals](#service-principals)
6. [Federated credentials](#federated-credentials)
7. [Azure RBAC — role definitions](#azure-rbac--role-definitions)
8. [Azure RBAC — role assignments](#azure-rbac--role-assignments)
9. [ABAC conditions](#abac-conditions)
10. [Managed identities (`az identity`)](#managed-identities-az-identity)

---

## `az ad` surface

[`az ad`](https://learn.microsoft.com/en-us/cli/azure/ad) calls [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) under the hood (previously AD Graph — migrated in CLI 2.37+). Requires an Entra directory role or Graph application permission depending on the call.

| Subgroup | Scope |
|---|---|
| `az ad user` | Directory users |
| `az ad group` | Security / M365 groups |
| `az ad app` | App registrations (the app definition) |
| `az ad sp` | Service principals (the app's instance in this tenant) |
| `az ad signed-in-user` | Current identity helpers |
| `az ad directory-role` | Built-in Entra roles (Global Admin, User Admin, etc.) — activation/listing |

Permission primer: most `list` / `show` calls need `User.Read.All`, `Group.Read.All`, or `Application.Read.All`; mutations need the `...ReadWrite.All` counterparts. Signed-in users with Global Admin / Privileged Role Administrator can do everything; SPs need explicit Graph app roles granted to them.

## Users

```bash
# List (paged under the hood)
az ad user list --query "[].{upn:userPrincipalName, id:id, type:userType}" -o table
az ad user list --filter "startswith(displayName,'Jean')" --output table

# Show
az ad user show --id alice@contoso.com
az ad user show --id <objectId>

# Create (cloud-only)
az ad user create \
  --display-name "Alice Example" \
  --user-principal-name alice@contoso.onmicrosoft.com \
  --password 'Tr0ub4dor&3' \
  --force-change-password-next-sign-in true

# Update
az ad user update --id alice@contoso.com --set department='Eng'

# Delete
az ad user delete --id alice@contoso.com

# Member of which groups?
az ad user get-member-groups --id alice@contoso.com -o table
```

- **Guests** (B2B) are created via [`az ad invitation create`](https://learn.microsoft.com/en-us/cli/azure/ad/invitation) — see ENTRAID/09-external-identities.md.
- Password complexity must satisfy tenant password policy; the CLI surfaces the Graph error message on failure.

## Groups

```bash
az ad group create --display-name devs --mail-nickname devs
az ad group show --group devs
az ad group list --filter "startswith(displayName,'devs')" -o table

# Members
az ad group member add --group devs --member-id <userObjectId>
az ad group member list --group devs -o table
az ad group member check --group devs --member-id <userObjectId>
az ad group member remove --group devs --member-id <userObjectId>

# Owners
az ad group owner add --group devs --owner-object-id <userObjectId>
az ad group owner list --group devs

# Dynamic-membership groups (require Entra ID P1)
az ad group create \
  --display-name devs-dynamic \
  --mail-nickname devsDyn \
  --group-types DynamicMembership Unified \
  --membership-rule 'user.department -eq "Eng"' \
  --membership-rule-processing-state On

az ad group delete --group devs
```

- `--group` accepts display name, object ID, or mail nickname — the CLI resolves in that order.
- Nested groups: add a group as a member via `--member-id <groupObjectId>` (assignment-type groups only; dynamic groups can't be members of dynamic groups).

## App registrations

```bash
# Create
az ad app create \
  --display-name my-api \
  --sign-in-audience AzureADMyOrg \
  --identifier-uris api://my-api-7f3e \
  --web-redirect-uris https://myapi.example.com/.auth/login/aad/callback

# Show / update
az ad app show --id <appId>                       # by application (client) ID
az ad app update --id <appId> --set tags='["web"]'

# List (filters use Graph $filter syntax)
az ad app list --display-name my-api -o table

# Exposed scopes & app roles (delegated vs application permissions)
az ad app permission list --id <appId>
az ad app permission add \
  --id <appId> \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope   # User.Read

# Admin-consent required for app permissions
az ad app permission admin-consent --id <appId>

# Delete
az ad app delete --id <appId>
```

- Every `az ad app create` also auto-creates a matching **service principal** in your home tenant unless you pass `--sign-in-audience AzureADMultipleOrgs` and defer SP creation.
- Graph API IDs: `00000003-0000-0000-c000-000000000000` is Microsoft Graph; see the [full list](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/governance/verify-first-party-apps-sign-in).

## Service principals

An [SP](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals) is the tenant-local identity that an app (managed identity, multi-tenant app, or your own registration) uses to sign in.

```bash
# One-shot: create app + SP + role assignment
az ad sp create-for-rbac \
  --name sp-deploy-dev \
  --role Contributor \
  --scopes /subscriptions/<subId>/resourceGroups/rg-dev \
  --years 1 \
  --create-cert                      # or --cert @mycert.pem or omit for password

# List
az ad sp list --display-name sp-deploy-dev -o table
az ad sp show --id <appId>

# Rotate credentials
az ad sp credential list --id <appId>
az ad sp credential reset --id <appId> --append --years 1         # new password/secret
az ad sp credential reset --id <appId> --append --cert @cert.pem  # replace with cert
az ad sp credential delete --id <appId> --key-id <credId>

# Associate only (SP already exists for a first-party app in another tenant)
az ad sp create --id <appId>

# Delete
az ad sp delete --id <appId>
```

- `--skip-assignment` on `create-for-rbac` creates the SP without granting anything — you wire RBAC later with `az role assignment`.
- `--role Contributor` is almost always too broad. Scope narrower by data-plane role (e.g., `Storage Blob Data Contributor`) on a specific resource.

## Federated credentials

Passwordless SP auth for CI/CD via [workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust).

```bash
# Create (example: GitHub Actions on main branch)
az ad app federated-credential create \
  --id <appObjectId> \
  --parameters '{
    "name": "gh-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "description": "GitHub main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Other subject shapes:
#   GitHub pull requests:         repo:myorg/myrepo:pull_request
#   GitHub environment:           repo:myorg/myrepo:environment:prod
#   Azure DevOps:                 sc://myorg/myproject/my-svc-conn
#   Kubernetes workload identity: system:serviceaccount:<ns>:<sa>

az ad app federated-credential list --id <appObjectId>
az ad app federated-credential delete --id <appObjectId> --federated-credential-id <fcId>
```

- **Max 20 federated credentials per app.**
- Federated-cred changes propagate within seconds — no need to rotate secrets.
- Same pattern works for user-assigned managed identities via [`az identity federated-credential create`](https://learn.microsoft.com/en-us/cli/azure/identity/federated-credential) — useful for AKS workload identity.

## Azure RBAC — role definitions

Azure RBAC lives at ARM (separate from Entra directory roles). [Built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) cover most cases.

```bash
# List built-ins
az role definition list --output table --query "[?roleType=='BuiltInRole'].{name:roleName, desc:description}" | head

# Find a role by name substring
az role definition list --query "[?contains(roleName,'Blob Data')].roleName" -o tsv

# Inspect what a role grants
az role definition list --name "Storage Blob Data Contributor" \
  --query "[0].{actions:permissions[0].actions, dataActions:permissions[0].dataActions, notDataActions:permissions[0].notDataActions}"

# Create a custom role
cat <<'JSON' > role-ops-reader.json
{
  "Name": "Ops Reader",
  "Description": "Read + monitor ops data",
  "Actions": [
    "Microsoft.Storage/storageAccounts/read",
    "Microsoft.Insights/*/read",
    "Microsoft.OperationalInsights/workspaces/*/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<subId>"]
}
JSON

az role definition create --role-definition @role-ops-reader.json
az role definition update --role-definition @role-ops-reader.json
az role definition delete --name "Ops Reader"
```

- **Actions** vs **DataActions**: control-plane vs data-plane. `*/read` on a storage account does **not** include reading blob contents — that's a `DataAction` on `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read`.
- `AssignableScopes` must be set before the role can be assigned; widen it later to reuse across subs/MGs.
- Limit: **5000 custom roles per tenant**.

## Azure RBAC — role assignments

```bash
# Grant — scopes cascade
az role assignment create \
  --role "Contributor" \
  --assignee alice@contoso.com \
  --scope /subscriptions/<subId>/resourceGroups/rg-dev

# By object ID (safer for SPs — display name can be ambiguous)
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id <spObjectId> \
  --assignee-principal-type ServicePrincipal \
  --scope <storageAccountResourceId>

# List on a scope
az role assignment list \
  --scope /subscriptions/<subId>/resourceGroups/rg-dev \
  --query "[].{who:principalName, what:roleDefinitionName, where:scope}" -o table

# List for a principal
az role assignment list --assignee alice@contoso.com --all -o table

# Revoke
az role assignment delete --assignee alice@contoso.com --role Contributor \
  --scope /subscriptions/<subId>/resourceGroups/rg-dev
```

- **`--assignee` vs `--assignee-object-id`**: `--assignee` accepts UPN / SP display name / object ID and resolves via Graph. In CI, prefer `--assignee-object-id` + `--assignee-principal-type` — avoids a Graph roundtrip and race conditions right after creating an SP.
- Limit: **4000 role assignments per subscription**, **500 per MG**. Consolidate via groups.
- Propagation: RBAC changes typically visible within **<1 minute**, can take up to **5 minutes**.

## ABAC conditions

[Azure ABAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) attaches boolean conditions to a role assignment — e.g., "only blobs tagged `classification=public`."

```bash
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id <spId> --assignee-principal-type ServicePrincipal \
  --scope <storageAccountId> \
  --description "Only public-classified blobs" \
  --condition "((!(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'})) OR (@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:classification<\$key_case_sensitive\$>StringEquals 'public'))" \
  --condition-version "2.0"
```

- Conditions are supported on a growing list of built-ins; use the portal's condition builder to generate the string, then paste into CLI.
- Debugging: `az role assignment list --query "[?condition!=null]"` shows every conditional assignment in scope.

## Managed identities (`az identity`)

User-assigned managed identities are standalone Azure resources you attach to one or more workloads. [System-assigned MI](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) is enabled per-resource via its own command group (`az vm identity assign`, `az webapp identity assign`, …).

```bash
# Create user-assigned MI
az identity create \
  --resource-group rg-mi \
  --name mi-shared \
  --location eastus
# ← the output contains: id, principalId (= object ID), clientId (= app ID), tenantId

az identity show --resource-group rg-mi --name mi-shared
az identity list --resource-group rg-mi -o table
az identity delete --resource-group rg-mi --name mi-shared

# Federated cred on a user-assigned MI (for AKS workload identity)
az identity federated-credential create \
  --resource-group rg-mi \
  --identity-name mi-shared \
  --name aks-sa \
  --issuer <oidcIssuerUrl> \
  --subject system:serviceaccount:default:app-sa \
  --audiences api://AzureADTokenExchange
```

### Assign MI to a workload

```bash
# VM — system-assigned
az vm identity assign --resource-group rg-dev --name vm-web
az vm identity remove --resource-group rg-dev --name vm-web

# VM — user-assigned
az vm identity assign --resource-group rg-dev --name vm-web --identities <miResourceId>

# App Service
az webapp identity assign --resource-group rg-dev --name app-web
az webapp identity assign --resource-group rg-dev --name app-web --identities <miResourceId>

# Function App
az functionapp identity assign --resource-group rg-dev --name func-etl

# AKS (kubelet identity for image pull)
az aks update --resource-group rg-dev --name aks-prod \
  --assign-kubelet-identity <miResourceId>

# Container Apps (both system- and user-assigned)
az containerapp identity assign --resource-group rg-dev --name app --system-assigned
az containerapp identity assign --resource-group rg-dev --name app --user-assigned <miResourceId>
```

Once assigned, grant Azure RBAC on the target resource using the MI's `principalId`:

```bash
MI_PID=$(az identity show -g rg-mi -n mi-shared --query principalId -o tsv)

az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id "$MI_PID" \
  --assignee-principal-type ServicePrincipal \
  --scope <storageAccountId>
```

## Gotchas

- **Graph permissions != Azure RBAC.** `az ad` needs Graph perms (directory roles, app permissions) — adding an Entra user through `Contributor` on a sub does nothing. Keep the two models separate mentally.
- **Race condition after creating an SP**: `az ad sp create` returns before Graph propagation completes. Adding a role assignment immediately sometimes fails with *principal does not exist*. Retry with backoff, or use `--assignee-object-id` (skips the Graph resolve).
- **`az role assignment list --all` is paginated** — set `--query "length(@)"` first to check volume on large tenants; full enumerations can take minutes.
- **Deleting an app registration deletes its SP** but **not** the role assignments — you get orphaned `assignments pointing to vanished principal IDs`. Clean with `az role assignment list --all --query "[?principalName==null].id"`.
- **Entra directory roles are separate** — activate/inspect via `az ad directory-role` but most admin work happens in the portal or [Privileged Identity Management](../ENTRAID/07-privileged-identity-management.md); CLI coverage is partial.
- **MI tokens are audience-specific** — `az login --identity` defaults to ARM audience; explicit `--scope` is required for data-plane tokens.

## Sources

- [`az ad`](https://learn.microsoft.com/en-us/cli/azure/ad) · [`az role`](https://learn.microsoft.com/en-us/cli/azure/role) · [`az identity`](https://learn.microsoft.com/en-us/cli/azure/identity)
- [App objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Azure RBAC built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Create a custom role](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles-cli)
- [Azure ABAC conditions](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview)
- [Workload identity federation setup](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust)
- [Managed identities overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
