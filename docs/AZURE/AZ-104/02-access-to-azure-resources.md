# Access to Azure Resources (RBAC)

> **Exam mapping:** AZ-104 → "Manage Azure identities and governance" (20–25%)
> **One-liner:** A **role assignment** binds a **security principal** to a **role definition** at a **scope** — master that formula and you master Azure access control.
> **Related:** [01-entra-id](01-entra-id.md) · [03-subscriptions-and-governance](03-subscriptions-and-governance.md) · [ENTRAID/05-rbac-and-authorization](../ENTRAID/05-rbac-and-authorization.md)

---

## 1. Azure RBAC Fundamentals

Azure role-based access control ([Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)) is the authorization system for Azure Resource Manager. Every access decision resolves to:

```
Role Assignment = Security Principal + Role Definition + Scope
```

### Security principal

The identity requesting access. Can be:

| Principal type | Example |
|---|---|
| **User** | `alice@contoso.com` |
| **Group** | "VM Admins" security group |
| **Service principal** | An app registration's identity |
| **Managed identity** | System-assigned MI on a VM or Function App |

> **Exam tip:** Always prefer assigning roles to **groups** rather than individual users — easier to manage and audit.

### Role definition

A collection of permissions expressed as `Actions`, `NotActions`, `DataActions`, and `NotDataActions`. Built-in roles are provided by Microsoft; custom roles can be created.

### Scope

The set of resources the access applies to (see Section 3).

### Deny assignments

[Deny assignments](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) block principals from performing specific actions **even if a role assignment grants them access**. Key facts:

- You **cannot** create deny assignments directly — they are created and managed by Azure itself (e.g., Azure Blueprints, deployment stacks).
- Deny assignments are evaluated **before** allow assignments.
- A deny assignment can include `DoNotApplyToChildScopes` to prevent inheritance to child scopes.
- Deny overrides allow, but an `ExcludePrincipals` list on the deny assignment can exempt specific principals.

---

## 2. Built-in Roles Deep-Dive

Azure RBAC has **100+** [built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles). The AZ-104 focuses on the following.

### Privileged administrator roles

| Role | What it can do | Key limitation |
|---|---|---|
| **Owner** | Full access to all resources **+ can assign roles** | Cannot manage Blueprints or share image galleries |
| **Contributor** | Full access to all resources | **Cannot assign roles** (no `Microsoft.Authorization/*` write) |
| **Reader** | View all resources | Cannot modify anything |
| **User Access Administrator** | Manage user access to resources (assign/revoke roles) | **Cannot** manage the resources themselves |
| **Role Based Access Control Administrator** | Assign roles via Azure RBAC only | Cannot manage access via Policy or other means |

> **Exam trap:** Owner = Contributor + User Access Administrator. Owner can assign roles; Contributor cannot. User Access Administrator can assign roles but cannot create/modify resources.

### Service-specific roles tested on AZ-104

| Role | Scope / Service | Key permissions |
|---|---|---|
| **Virtual Machine Contributor** | Compute | Manage VMs but not the VNet or storage they connect to |
| **Storage Blob Data Owner** | Storage (data plane) | Full access to blob containers and data, including assigning access |
| **Storage Blob Data Contributor** | Storage (data plane) | Read, write, delete blobs — **cannot** assign access or set ACLs |
| **Storage Blob Data Reader** | Storage (data plane) | Read-only access to blob data |
| **Network Contributor** | Networking | Manage networks, subnets, NSGs, route tables, etc. |
| **Backup Operator** | Recovery Services | Manage backups (except removing backup, creating vaults, giving access) |
| **Monitoring Contributor** | Monitor | Read monitoring data + edit monitoring settings, action groups, alerts |
| **Monitoring Reader** | Monitor | Read all monitoring data |
| **Key Vault Administrator** | Key Vault (data plane) | Full access to keys, secrets, and certificates in a vault |
| **Key Vault Secrets Officer** | Key Vault (data plane) | Manage secrets only (not keys or certificates) |

> **Important distinction:** "Storage Blob Data *" roles control the **data plane** (reading/writing blobs). The plain "Contributor" role on a storage account controls the **management plane** (creating/deleting the account, configuring networking) but does **not** grant access to the data inside.

---

## 3. Scope Hierarchy

```
Management Group          ← broadest
  └── Subscription
        └── Resource Group
              └── Resource      ← narrowest
```

Format of a scope string:

```
/providers/Microsoft.Management/managementGroups/{mg-id}
/subscriptions/{sub-id}
/subscriptions/{sub-id}/resourceGroups/{rg-name}
/subscriptions/{sub-id}/resourceGroups/{rg-name}/providers/{provider}/{type}/{name}
```

### Inheritance rules

| Rule | Detail |
|---|---|
| **Downward inheritance** | A role assigned at a parent scope is inherited by **all** child scopes. Reader on a subscription = Reader on every RG and resource in that subscription. |
| **No upward inheritance** | Assigning a role at a resource group does **not** grant access to the subscription. |
| **Additive model** | Effective permissions = union of all role assignments at the current scope + all inherited assignments. |
| **Deny overrides allow** | If a deny assignment exists, it blocks the action even if an allow role assignment grants it. |
| **Cannot block inheritance** | You cannot "break" RBAC inheritance. The only way to restrict an inherited permission is via a deny assignment (which you cannot create manually). |

> **Exam tip:** If a question says "User has Contributor at subscription level, but should not modify resources in RG-X" — you cannot simply remove inheritance. The intended answer is usually to restructure scope assignments, use a different subscription, or (in advanced scenarios) use Azure Policy to deny specific operations.

See [Understand scope for Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview).

---

## 4. Assigning Roles

### Prerequisites

You need `Microsoft.Authorization/roleAssignments/write` permission — granted by **Owner**, **User Access Administrator**, or **Role Based Access Control Administrator**.

### Azure CLI

```bash
# Assign "Virtual Machine Contributor" to a user at resource-group scope
az role assignment create \
  --assignee "alice@contoso.com" \
  --role "Virtual Machine Contributor" \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG"

# Assign to a managed identity using its object (principal) ID
az role assignment create \
  --assignee-object-id "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageacct"

# Assign to a group at subscription scope
az role assignment create \
  --assignee-object-id "{group-object-id}" \
  --assignee-principal-type Group \
  --role "Reader" \
  --subscription "{sub-id}"
```

See [Assign Azure roles using Azure CLI](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-cli).

### Azure PowerShell

```powershell
# Assign "Contributor" to a user at resource-group scope
New-AzRoleAssignment `
  -SignInName "alice@contoso.com" `
  -RoleDefinitionName "Contributor" `
  -ResourceGroupName "myRG"

# Assign to a service principal by object ID at subscription scope
New-AzRoleAssignment `
  -ObjectId "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" `
  -RoleDefinitionName "Reader" `
  -Scope "/subscriptions/{sub-id}"

# Assign to a managed identity
New-AzRoleAssignment `
  -ObjectId $mi.PrincipalId `
  -RoleDefinitionName "Storage Blob Data Reader" `
  -Scope "/subscriptions/{sub-id}/resourceGroups/myRG"
```

See [Assign Azure roles using Azure PowerShell](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-powershell).

### Azure Portal

1. Navigate to the resource, resource group, subscription, or management group.
2. Click **Access control (IAM)**.
3. Click **+ Add** → **Add role assignment**.
4. Select a role → select members (user, group, SP, MI) → **Review + assign**.

See [Assign Azure roles using the Azure portal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal).

### What you can assign to

| Principal type | Identifier used |
|---|---|
| User | UPN (`alice@contoso.com`) or Object ID |
| Security group | Object ID |
| Service principal | Application ID or Object ID |
| Managed identity | Object ID (Principal ID) |

---

## 5. Interpreting Access

### CLI / PowerShell

```bash
# List all role assignments for a user
az role assignment list --assignee "alice@contoso.com" --output table

# List all role assignments at a specific scope
az role assignment list --scope "/subscriptions/{sub-id}/resourceGroups/myRG" --output table

# List role assignments including inherited ones
az role assignment list --scope "/subscriptions/{sub-id}/resourceGroups/myRG" --include-inherited --output table
```

```powershell
# PowerShell equivalent
Get-AzRoleAssignment -SignInName "alice@contoso.com"
Get-AzRoleAssignment -Scope "/subscriptions/{sub-id}/resourceGroups/myRG"
```

### Portal — "Check access" blade

1. Go to the resource / RG / subscription → **Access control (IAM)**.
2. Click the **Check access** tab.
3. Search for a user, group, or service principal.
4. View **effective** role assignments — shows both **direct** and **inherited** assignments with their source scope.

### Understanding the output

| Column | Meaning |
|---|---|
| **Role** | The role definition name (e.g., Contributor) |
| **Principal** | Who has the assignment |
| **Scope** | Where the assignment was made |
| **Inherited** | Whether the assignment was made at a parent scope (Yes/No) |
| **Condition** | Whether ABAC conditions are attached |

> **Key concept:** A role assignment listed as "Inherited" on a resource group was actually assigned at the subscription (or management group) level. You must go to that parent scope to remove it.

---

## 6. Azure RBAC vs. Microsoft Entra Roles

This is one of the **most commonly tested distinctions** on AZ-104.

| | **Azure RBAC** | **Microsoft Entra roles** |
|---|---|---|
| **What it manages** | Azure **resources** (VMs, storage, networking, etc.) | The **directory** (users, groups, app registrations, domains) |
| **Where defined** | `Microsoft.Authorization/roleDefinitions` | Microsoft Entra / Microsoft Graph |
| **Assigned via** | Portal IAM blade, `az role assignment create` | Entra admin center → Roles & admins, `New-MgRoleManagementDirectoryRoleAssignment` |
| **Scope model** | Management Group → Subscription → RG → Resource | Tenant-wide (or [administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-manage) for scoping) |
| **Example roles** | Owner, Contributor, Reader, VM Contributor | Global Administrator, User Administrator, Application Administrator |
| **Custom roles** | Yes — JSON with Actions/DataActions | Yes — via Entra |
| **PIM support** | Yes | Yes |

### Common confusion points

1. **Global Administrator ≠ Owner.** Global Admin is an Entra role with full directory control. It does **not** automatically have access to Azure subscriptions. However, a Global Admin can **elevate** themselves using the "Access management for Azure resources" toggle, which grants User Access Administrator at the tenant root management group scope.

2. **Subscription Owner ≠ Global Admin.** A subscription Owner has full Azure RBAC on that subscription but zero Entra directory permissions.

3. **The two systems do not overlap.** Azure RBAC permissions cannot be used in Entra custom roles, and vice versa.

See [Azure roles, Microsoft Entra roles, and classic subscription administrator roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles).

---

## 7. Custom Roles

When no built-in role provides the exact permissions needed, create a [custom role](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles).

### When to create one

- A team needs to start/stop VMs but not delete them.
- An app needs to read from specific resource types only.
- An operator needs a subset of Contributor permissions.

### JSON definition structure

```json
{
  "Name": "VM Operator",
  "IsCustom": true,
  "Description": "Can start, restart, and stop VMs but not create or delete them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/{sub-id}"
  ]
}
```

### Key properties

| Property | Purpose |
|---|---|
| `Actions` | Management-plane operations allowed (e.g., `Microsoft.Compute/*/read`) |
| `NotActions` | Operations excluded from `Actions` (subtracted, not explicit deny) |
| `DataActions` | Data-plane operations allowed (e.g., `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read`) |
| `NotDataActions` | Data-plane operations excluded |
| `AssignableScopes` | Where the role can be assigned — must be management groups, subscriptions, or resource groups |

### Creating a custom role

```bash
# CLI
az role definition create --role-definition vm-operator.json

# PowerShell
New-AzRoleDefinition -InputFile "vm-operator.json"
```

### Limits

- Up to **5,000** custom roles per tenant.
- `AssignableScopes` must include at least one management group, subscription, or resource group — cannot be a single resource.

> **AZ-104 scope:** The exam tests **awareness** of custom roles — know when they are needed, the JSON structure, and `AssignableScopes`. Deep creation details are more heavily tested on AZ-305.

---

## 8. Conditions (ABAC)

[Azure attribute-based access control (ABAC)](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) adds **conditions** to role assignments, enabling finer-grained control based on resource attributes.

### How it works

A role assignment condition is an additional check evaluated during authorization:

```
IF  principal has "Storage Blob Data Reader"
AND blob container name == "public-data"
THEN  allow read
```

### Current support

- **Azure Blob Storage** — conditions on container name, blob path, tags, encryption scope, etc.
- **Azure Queue Storage** — conditions on queue name.
- **Azure Container Registry** — conditions on repository permissions.

### Example use case

Grant a user **Storage Blob Data Contributor** but only for blobs in a specific container or with a specific blob index tag, without creating separate storage accounts.

### AZ-104 relevance

The exam may test **basic awareness**:

- ABAC exists and extends Azure RBAC with attribute conditions.
- It currently applies primarily to storage services.
- Conditions are attached to role assignments, not to role definitions.
- Conditions **cannot** be used to create explicit deny access (you still need deny assignments for that).

See [Authorize access to Blob Storage using Azure role assignment conditions](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-auth-abac).

---

## Exam Traps

| Trap | What to remember |
|---|---|
| **Owner vs. User Access Administrator** | Owner = full resource access + role assignment. User Access Administrator = role assignment only, no resource management. Contributor = full resource access but **no** role assignment. |
| **RBAC vs. Entra roles** | Azure RBAC → resources. Entra roles → directory. They are **separate systems**. Global Admin does not automatically have Azure RBAC access (must elevate). |
| **Scope inheritance cannot be broken** | You cannot remove an inherited role assignment at a child scope. You must go to the scope where it was assigned. |
| **Deny overrides allow** | Deny assignments always win. But you cannot create them yourself — only Azure (Blueprints, deployment stacks) can. |
| **NotActions ≠ Deny** | `NotActions` simply subtracts from `Actions` — it is not a deny. If another role assignment grants the action, the user still has it. |
| **Data plane vs. management plane** | Contributor on a storage account does **not** grant access to read blobs. You need Storage Blob Data Reader/Contributor/Owner for data-plane access. |
| **"Check access" vs. role assignment list** | "Check access" shows **effective** permissions (all inherited + direct). The IAM role assignments list shows only assignments at that specific scope unless you filter for inherited. |
| **Custom role AssignableScopes** | This controls where the role **can be assigned**, not where it is actually assigned. A role with `AssignableScopes` of Sub-A cannot be assigned in Sub-B. |
| **RBAC assignment limit** | Azure supports up to **4,000** role assignments per subscription (recently increased from 2,000). |

---

## Sources

- [What is Azure RBAC?](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Understand scope for Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview)
- [Steps to assign an Azure role](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-steps)
- [Assign Azure roles using Azure CLI](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-cli)
- [Assign Azure roles using Azure PowerShell](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-powershell)
- [Assign Azure roles using the Azure portal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal)
- [Azure roles, Entra roles, and classic subscription administrator roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles)
- [Azure custom roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles)
- [Azure ABAC overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview)
- [Azure deny assignments](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments)
- [Authorize access to Blob Storage using conditions](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-auth-abac)
- [AZ-104 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [Understand Azure role definitions](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-definitions)
