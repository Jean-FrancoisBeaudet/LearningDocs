# RBAC & Authorization

> **Exam mapping:** AZ-204 · AZ-305 *(Design authorization solutions)*
> **One-liner:** Two distinct role systems — **Entra (directory) roles** for identity admin, **Azure RBAC** for resource access — both pluggable with ABAC conditions and PIM.
> **Related:** [managed identities](04-managed-identities.md) · [PIM](07-privileged-identity-management.md) · [conditional access](06-conditional-access-and-mfa.md)

## Two role systems — do not mix them up

| | **Entra (directory) roles** | **Azure RBAC (resource) roles** |
|---|------------------------------|-------------------------------|
| Scope | The Entra **tenant** and its objects (users, groups, apps, devices) | Azure **resources**: management group → subscription → resource group → resource |
| Examples | Global Administrator, User Administrator, Application Administrator, Privileged Role Administrator | Owner, Contributor, Reader, Storage Blob Data Reader, Key Vault Secrets User |
| Defined in | Entra | Azure (`Microsoft.Authorization/roleDefinitions`) |
| Assigned via | `New-MgRoleManagementDirectoryRoleAssignment`, Entra admin center → Roles & admins | `az role assignment create`, portal → IAM blade |
| PIM-eligible | Yes (Entra roles + groups) | Yes (Azure resources + groups) |

A user can be a Subscription **Owner** (Azure RBAC) without being a **Global Admin** (Entra) — and vice versa. Global Admin can elevate to gain root Azure RBAC at the tenant root via *Access management for Azure resources* toggle (audit-logged, then revoke).

## Azure RBAC anatomy

A role assignment = `{ security principal, role definition, scope }`.

- **Security principal:** user, group, SP/MI.
- **Role definition:** built-in or custom.
- **Scope:** management group ⊃ subscription ⊃ resource group ⊃ resource.

Scopes are inheriting (lower scopes inherit assignments from higher), and **deny assignments** (rare; only Azure platform / blueprints set them) override allow.

### Built-in role categories

| Category | Example roles |
|----------|---------------|
| **Privileged admin** | Owner, Contributor, Role Based Access Control Administrator, User Access Administrator |
| **General** | Reader |
| **Data plane** | Storage Blob Data Owner/Contributor/Reader, Cosmos DB Built-in Data Reader/Contributor, Key Vault Secrets/Crypto/Certificates User/Officer, Service Bus Data Sender/Receiver |
| **Compute / AI** | AcrPull, AcrPush, AKS RBAC Cluster Admin, AzureML Data Scientist, Cognitive Services User, **Cognitive Services OpenAI User / Contributor** |

> **Owner = Contributor + role assignment writes.** Contributor cannot grant access to others; Owner can.
> **User Access Administrator** is "Owner without resource control" — only manages assignments.
> **RBAC Administrator** (newer) lets you grant role assignments but limited to a specific allow-list of role definitions — preferred over Owner/UAA for delegated access management.

## Custom roles

Define `Actions` / `NotActions` / `DataActions` / `NotDataActions` / `AssignableScopes`.

```json
{
  "Name": "Storage Blob Restorer",
  "Description": "Restore deleted blobs in the data team subscription",
  "Actions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/read"
  ],
  "DataActions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read",
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write"
  ],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<sub-id>"]
}
```

```bash
az role definition create --role-definition ./role.json
```

Limits: **5 000 custom roles per tenant** (Azure Government: 2 000); a role definition's `AssignableScopes` cannot exceed where you said it could be assigned.

## Azure ABAC (conditions)

Adds **conditions** to RBAC assignments — attribute-based filtering on the resource, request, environment, or principal. GA for Storage; preview/limited for Key Vault.

Example: a user has *Storage Blob Data Reader* on a container, but only blobs tagged `Department=Finance` should be readable.

```text
(
  !(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'})
)
OR
(
  @Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:Department<$key_case_sensitive$>] StringEquals 'Finance'
)
```

Common attribute sources:

| Source | Examples |
|--------|----------|
| `@Request` | `@Request[Microsoft.Storage/.../blobs:snapshot]` |
| `@Resource` | Tags, container name, blob index tags, encryption scope |
| `@Principal` | Custom security attributes on the user/SP (preview) |
| `@Environment` | Private link, network, etc. (limited) |

ABAC reduces role-assignment sprawl — fewer roles needed because conditions narrow each one.

## Scope hierarchy & inheritance

```
/                                        ← root (visible only to elevated GA)
└── /providers/Microsoft.Management/managementGroups/{mg}
    └── /subscriptions/{subId}
        └── /subscriptions/{subId}/resourceGroups/{rg}
            └── /subscriptions/.../providers/Microsoft.Storage/storageAccounts/{name}
                └── ...nested resources (containers, blobs)
```

Assignments inherit downward; you cannot remove an inherited role at a lower scope (only override with a different one or use deny). Use **management groups** to apply org-wide RBAC + Policy.

## Service-specific authz models

| Service | Notes |
|---------|-------|
| **Storage** | Choose RBAC + Entra **per request** vs. SAS vs. account key. Disable shared key (`allowSharedKeyAccess: false`) for hardened deployments. |
| **Key Vault** | RBAC (preferred, default for new vaults from API `2026-02-01`) **or** legacy access policies — one model per vault. |
| **Cosmos DB** | Control plane = Azure RBAC. Data plane = **Cosmos-specific RBAC** (`az cosmosdb sql role assignment`); set `disableLocalAuth: true`. |
| **Service Bus / Event Hubs / Event Grid** | RBAC (Sender/Receiver/Owner data roles) or SAS. RBAC is preferred. |
| **Azure SQL** | Entra-only auth recommended (`Microsoft Entra admin` for the server) + database users mapped to Entra principals. |
| **Azure OpenAI / AI Foundry** | `Cognitive Services OpenAI User` for inference; `Contributor` for deploy; disable key auth where possible. |
| **AKS** | Cluster RBAC via Entra integration; pod-level via Workload Identity + Azure RBAC on resources the pod accesses. |

## CLI quick reference

```bash
# Assign a built-in role at resource scope
az role assignment create \
  --assignee-object-id $PRINCIPAL_OID \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID

# List assignments for a principal
az role assignment list --assignee $UPN --all -o table

# List built-in roles matching a name
az role definition list --query "[?contains(roleName, 'Key Vault')].roleName"

# Create a deny-when-not-tagged condition (Storage)
az role assignment create \
  --assignee $PRINCIPAL_OID \
  --role "Storage Blob Data Reader" \
  --scope $CONTAINER_SCOPE \
  --condition "@Resource[Microsoft.Storage/.../tags:Department<\$key_case_sensitive\$>] StringEquals 'Finance'" \
  --condition-version "2.0"
```

## PIM interaction

PIM applies on top of either role system. Assignments become **eligible** instead of permanent active; users activate just-in-time with optional MFA, approval, ticket number, max duration. See [`07-privileged-identity-management.md`](07-privileged-identity-management.md).

| Concept | Where it lives |
|---------|----------------|
| PIM for **Entra roles** | Free with P2; manages directory roles |
| PIM for **Azure resources** | Free with P2; manages Azure RBAC roles |
| PIM for **Groups** | Free with P2; manages role-assignable groups |

## Limits to remember

| Thing | Limit |
|-------|-------|
| Azure role assignments per subscription | **4 000** |
| Azure role assignments per management group | **500** |
| Custom roles per tenant | **5 000** |
| Eligible PIM assignments per resource scope | 500 |
| Conditions per assignment | 1 (with full DSL) |

## Exam traps

- **Entra roles ≠ Azure roles.** Global Admin doesn't grant Azure resource access until they elevate.
- **Owner can grant access; Contributor cannot.** Use **RBAC Administrator** (with allow-listed roles) when you need to delegate "permission to grant" without giving Owner.
- **`AssignableScopes` of a custom role caps where it can be used.**
- **Role assignment changes can take ~5 minutes** to propagate (token cache + ARM cache).
- **Storage SAS tokens bypass RBAC** — disable shared-key auth to enforce Entra-only.
- **Key Vault default model is changing** — new vaults default to **Azure RBAC** from API version `2026-02-01`. Old vaults stay on access policies until migrated.
- **ABAC conditions are evaluated at the data plane** — check `--condition-version 2.0` on `az role assignment`.

## Sources

- [Azure RBAC overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Azure ABAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview)
- [Entra built-in roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)
- [Key Vault RBAC vs access policies](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-access-policy)
- [Key Vault API 2026-02-01 RBAC default](https://learn.microsoft.com/en-us/azure/key-vault/general/access-control-default)
