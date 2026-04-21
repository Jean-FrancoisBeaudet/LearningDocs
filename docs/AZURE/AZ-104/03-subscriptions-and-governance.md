# Azure Subscriptions and Governance

> **Exam mapping:** AZ-104 -- "Manage Azure identities and governance" (20-25%)
> **One-liner:** Govern your Azure environment through policies, locks, tags, cost controls, and a management-group hierarchy that cascades rules down to every subscription and resource.
> **Related:** [AZURE/ENTRAID/05-rbac-and-authorization.md](../ENTRAID/05-rbac-and-authorization.md) | [AZURE/ENTRAID/08-identity-governance.md](../ENTRAID/08-identity-governance.md) | [AZURE/AZ-305/01-identity-governance-monitoring.md](../AZ-305/01-identity-governance-monitoring.md)

---

## Table of Contents

1. [Azure Policy](#1-azure-policy)
2. [Resource Locks](#2-resource-locks)
3. [Tags](#3-tags)
4. [Resource Groups](#4-resource-groups)
5. [Subscriptions](#5-subscriptions)
6. [Cost Management](#6-cost-management)
7. [Management Groups](#7-management-groups)
8. [Exam Traps](#exam-traps)
9. [Sources](#sources)

---

## 1. Azure Policy

[Microsoft Learn -- Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)

Azure Policy evaluates resources and marks them as compliant or non-compliant according to rules you define. It can also **prevent** non-compliant resources from being created and **remediate** existing resources automatically.

### Key Concepts

| Term | Description |
|---|---|
| **Policy definition** | A single rule written in JSON (condition + effect). |
| **Initiative (policy set)** | A collection of related policy definitions grouped together for assignment as a unit. |
| **Assignment** | Binding a definition or initiative to a **scope** (management group, subscription, resource group, or individual resource). |
| **Exemption** | Temporarily or permanently excluding a resource from an assignment. |
| **Compliance** | Evaluation result -- resources are **Compliant**, **Non-compliant**, **Exempt**, or **Conflicting**. |

### Built-in vs Custom Policies

- **Built-in**: Microsoft-maintained; examples include *Allowed locations*, *Allowed virtual machine SKUs*, *Inherit a tag from the resource group*, *Require a tag on resources*.
- **Custom**: Author your own JSON definition when built-in policies don't cover your requirement. Stored in the selected scope (subscription or management group).

### Policy Definition JSON Structure (Simplified)

```json
{
  "properties": {
    "displayName": "Require a tag on resources",
    "policyType": "Custom",
    "mode": "Indexed",          // "All" or "Indexed"
    "parameters": {
      "tagName": {
        "type": "String",
        "metadata": { "description": "Name of the required tag" }
      }
    },
    "policyRule": {
      "if": {
        "field": "[concat('tags[', parameters('tagName'), ']')]",
        "exists": "false"
      },
      "then": {
        "effect": "deny"
      }
    }
  }
}
```

- **mode `All`** -- evaluates all resource types including those that don't support tags/location.
- **mode `Indexed`** -- only evaluates resource types that support tags and location (most common).

### Policy Effects

Effects define what happens when a policy rule matches. They are evaluated in a specific order.

| Effect | When it fires | Typical use |
|---|---|---|
| **Disabled** | Checked **first**; skips evaluation entirely. | Temporarily turning off a policy. |
| **Append** | Adds fields to the request before it is sent to the RP. | Add a tag or IP rule to a resource. |
| **Modify** | Adds, updates, or removes tags/properties on a resource. | Enforce tag inheritance, add managed identity. |
| **Deny** | Blocks the request. Evaluated **before** Audit to avoid double logging. | Prevent creation of non-compliant resources. |
| **Audit** | Creates a warning event in the activity log; resource is marked non-compliant. | Soft enforcement / visibility. |
| **AuditIfNotExists** | Audits when a **related** resource is missing (e.g., no diagnostic setting). | Post-deployment compliance check. |
| **DeployIfNotExists** | Triggers a template deployment to create/configure a related resource. | Auto-deploy a Log Analytics agent. |
| **DenyAction** | Blocks a specific action (currently only `DELETE`). | Protect critical resources from deletion at scale. |
| **Manual** | Compliance is tracked through attestation rather than auto-evaluation. | Processes requiring human sign-off. |

#### Evaluation Order (Resource Manager mode)

```
1. Disabled          -- skip rule if disabled
2. Append / Modify   -- mutate the request
3. Deny              -- block if non-compliant
4. Audit             -- log non-compliance
   ---- request sent to Resource Provider ----
5. AuditIfNotExists / DeployIfNotExists  -- post-provisioning check
```

> **DenyAction** is evaluated independently on delete operations.

### Remediation Tasks

For **DeployIfNotExists** and **Modify** effects, you can create a **remediation task** to bring existing non-compliant resources into compliance. The remediation uses a [managed identity](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources) assigned to the policy assignment.

### CLI Quick Reference

```bash
# Create a custom policy definition
az policy definition create \
  --name "require-env-tag" \
  --display-name "Require Environment tag" \
  --rules @rules.json \
  --params @params.json \
  --mode Indexed

# Create an initiative (policy set)
az policy set-definition create \
  --name "tagging-initiative" \
  --definitions @initiative.json

# Assign a policy to a resource group
az policy assignment create \
  --name "require-env-tag-rg" \
  --policy "require-env-tag" \
  --scope "/subscriptions/<sub-id>/resourceGroups/myRG" \
  --params '{"tagName": {"value": "Environment"}}'

# Assign an initiative to a subscription
az policy assignment create \
  --name "tagging-initiative-sub" \
  --policy-set-definition "tagging-initiative" \
  --scope "/subscriptions/<sub-id>"

# Check compliance state
az policy state summarize \
  --resource-group myRG
```

---

## 2. Resource Locks

[Microsoft Learn -- Lock resources to protect infrastructure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)

Locks prevent accidental modification or deletion of Azure resources, regardless of RBAC permissions.

### Lock Types

| Lock level | Effect |
|---|---|
| **CanNotDelete** (`Delete`) | Authorized users can read and modify, but **cannot delete** the resource. |
| **ReadOnly** (`ReadOnly`) | Authorized users can read, but **cannot update or delete**. Equivalent to granting all users the Reader role for that resource. |

### Inheritance

- A lock on a **resource group** applies to **all resources** within it (including resources added later).
- A lock on a **subscription** applies to all resource groups and resources under it.
- The **most restrictive** lock in the inheritance chain wins.

### Who Can Manage Locks?

You need the `Microsoft.Authorization/locks/*` action, which is included in only two built-in roles:

- **Owner**
- **User Access Administrator**

> Even a Contributor cannot create or delete locks.

### ReadOnly Lock Surprises

A ReadOnly lock blocks **any operation that uses POST** against the Azure Resource Manager API, which is more than just "writes":

| Resource | Blocked operation |
|---|---|
| Storage account | **List keys** (POST) -- cannot retrieve access keys. |
| App Service | **Start/Stop/Restart** the web app. |
| Application Gateway | Get backend health (POST). |
| Azure Advisor | Cannot store query results. |
| Log Analytics workspace | Cannot enable UEBA. |
| AKS | Cannot browse Kubernetes resources in the portal. |
| Subscription | Advisor cannot store results of its queries. |

### CanNotDelete Lock on Resource Groups -- Deployment History Trap

When a resource group has a **CanNotDelete** lock, Azure cannot auto-purge old deployment history entries. Once the 800-deployment limit is reached, **new deployments will fail**. Delete stale entries manually or use a different cleanup strategy.

### CLI Quick Reference

```bash
# Create a CanNotDelete lock on a resource group
az lock create \
  --name "no-delete-rg" \
  --lock-type CanNotDelete \
  --resource-group myRG

# Create a ReadOnly lock on a specific resource
az lock create \
  --name "readonly-vm" \
  --lock-type ReadOnly \
  --resource-group myRG \
  --resource-name myVM \
  --resource-type Microsoft.Compute/virtualMachines

# List locks
az lock list --resource-group myRG --output table

# Delete a lock (must have Owner or User Access Administrator)
az lock delete --name "no-delete-rg" --resource-group myRG
```

---

## 3. Tags

[Microsoft Learn -- Use tags to organize Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)

Tags are **name-value pairs** applied to resources, resource groups, and subscriptions for metadata (cost tracking, governance, automation, environment classification).

### Tag Limits

| Limit | Value |
|---|---|
| Tags per resource / RG / subscription | **50** |
| Tag name length | 512 characters (128 for storage accounts) |
| Tag value length | 256 characters |

> Not all resource types support tags. See the [tag support reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-support).

### Tags Do NOT Inherit

This is one of the most tested concepts on AZ-104:

- Tags applied to a **resource group** are **NOT** automatically inherited by its child resources.
- Tags applied to a **subscription** are **NOT** automatically inherited by resource groups or resources.

To enforce tag inheritance, use **Azure Policy** with the **Modify** effect:

- Built-in policy: *Inherit a tag from the resource group* -- copies a specified tag from the parent RG to new/updated child resources.
- Built-in policy: *Inherit a tag from the subscription* -- copies a tag to child resources.

> Cost Management has its own **tag inheritance** feature that applies subscription/RG tags to child resource **usage records** (not the resources themselves) for cost reporting.

### Common Tagging Strategy

| Tag name | Purpose | Example values |
|---|---|---|
| `Environment` | Dev/Test/Staging/Prod classification | `dev`, `prod` |
| `CostCenter` | Chargeback | `CC-1234` |
| `Owner` | Contact for the resource | `team-platform` |
| `Project` | Project or workload association | `ecommerce-v2` |
| `Automation` | Trigger automation (e.g., auto-shutdown) | `shutdown-18:00` |

### CLI Quick Reference

```bash
# Add/update tags on a resource (merge mode -- keeps existing tags)
az resource tag \
  --tags Environment=prod CostCenter=CC-1234 \
  --ids "/subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM"

# Replace ALL tags on a resource group
az tag create \
  --resource-id "/subscriptions/<sub-id>/resourceGroups/myRG" \
  --tags Environment=prod CostCenter=CC-1234

# Merge tags (additive, keeps existing)
az tag update \
  --resource-id "/subscriptions/<sub-id>/resourceGroups/myRG" \
  --operation merge \
  --tags Project=ecommerce-v2

# Remove all tags
az tag delete \
  --resource-id "/subscriptions/<sub-id>/resourceGroups/myRG" \
  --yes

# List resources by tag
az resource list --tag Environment=prod --output table
```

---

## 4. Resource Groups

[Microsoft Learn -- Manage resource groups](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)

A resource group (RG) is a **logical container** that holds related Azure resources for a solution.

### Key Rules

| Rule | Detail |
|---|---|
| **Every resource must belong to exactly one RG.** | A resource cannot exist without a resource group. |
| **RGs cannot be nested.** | You cannot place one resource group inside another. |
| **RG location is metadata only.** | The RG stores metadata (for compliance). Resources inside it can be in **different regions**. |
| **Deleting an RG deletes all resources inside it.** | This is irreversible -- use locks to protect critical RGs. |
| **RG name is immutable after creation.** | You cannot rename a resource group. |
| **RBAC and Policy apply at the RG scope.** | Assignments at the RG level are inherited by all resources within it. |

### Moving Resources Between Resource Groups

You can move resources between RGs (and even between subscriptions), but there are constraints:

- Both source and destination RGs are **locked during the move** (no create/delete, but existing resources remain accessible).
- Not all resource types support moves. Check the [move operation support matrix](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-support-resources).
- The resource's region does **not** change -- only the RG association.
- You need `Microsoft.Resources/subscriptions/resourceGroups/moveResources/action` permission on **both** source and destination.

```bash
# Move resources to a different resource group
az resource move \
  --destination-group newRG \
  --ids "/subscriptions/<sub-id>/resourceGroups/oldRG/providers/Microsoft.Compute/virtualMachines/myVM"

# Delete a resource group (all contents deleted!)
az group delete --name myRG --yes --no-wait
```

---

## 5. Subscriptions

[Microsoft Learn -- Azure subscription overview](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription)

A subscription is both a **billing boundary** and an **access-control boundary**.

### Why Multiple Subscriptions?

| Reason | Explanation |
|---|---|
| **Billing separation** | Different departments, projects, or environments get their own invoice. |
| **Access boundary** | Subscription-level RBAC provides coarse-grained access control. |
| **Quota/limit isolation** | Each subscription has its own [service limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits) (e.g., max 980 RGs per subscription). |
| **Policy isolation** | Apply different governance policies to dev vs prod subscriptions. |
| **Regulatory compliance** | Isolate workloads subject to different compliance requirements. |

### Subscription Limits (Selected)

| Limit | Default |
|---|---|
| Resource groups per subscription | 980 |
| Deployments per resource group | 800 |
| Regions per subscription | All available |
| Tags per subscription | 50 |

### Transferring Subscriptions

- You can transfer billing ownership of a subscription to another account.
- You can move a subscription to a different Entra ID tenant (changes the trust relationship, RBAC assignments are **removed**).
- You can move a subscription between management groups.

```bash
# List subscriptions
az account list --output table

# Set active subscription
az account set --subscription "<sub-id-or-name>"

# Show current subscription details
az account show
```

---

## 6. Cost Management

[Microsoft Learn -- Cost Management overview](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management)

**Microsoft Cost Management** (formerly Azure Cost Management + Billing) provides tools to monitor, allocate, and optimize cloud spending.

### Core Components

| Component | Purpose |
|---|---|
| **Cost analysis** | Visualize and explore accumulated costs with filters (resource group, tag, service, region). |
| **Budgets** | Set spending limits and trigger alerts/actions when thresholds are crossed. |
| **Cost alerts** | Three types: budget alerts, credit alerts, department spending quota alerts. |
| **Azure Advisor** | Provides cost optimization recommendations (right-size VMs, delete unused resources, buy reservations). |
| **Exports** | Schedule automatic export of cost data to a storage account (CSV). |

### Budgets

Budgets can be scoped to subscriptions, resource groups, or management groups.

| Setting | Options |
|---|---|
| **Amount** | Monthly, quarterly, or annual dollar amount. |
| **Time grain** | Monthly, Quarterly, Annually. |
| **Alert thresholds** | Percentage of budget (e.g., 50%, 75%, 90%, 100%). |
| **Alert recipients** | Email addresses and/or an **Action Group** (trigger Logic Apps, Azure Functions, runbooks). |
| **Filter** | Scope by resource group, resource, meter, tag. |

> Costs are evaluated against budget thresholds **once per day**. Notifications fire within one hour of detection.

### Azure Advisor Cost Recommendations

- **Shut down underutilized VMs** -- based on CPU/network usage over 7 days.
- **Right-size VMs** -- downgrade SKU when resources are consistently underused.
- **Buy Reserved Instances** -- identify workloads that would benefit from 1-year or 3-year reservations.
- **Delete unused public IPs, unattached disks, idle ExpressRoute circuits.**

Access Advisor: Portal > **Advisor** > **Cost** tab, or:

```bash
az advisor recommendation list --category cost --output table
```

### CLI Quick Reference

```bash
# Create a budget for a subscription
az consumption budget create \
  --budget-name "monthly-5000" \
  --amount 5000 \
  --time-grain Monthly \
  --start-date "2026-04-01" \
  --end-date "2027-03-31" \
  --category Cost

# List budgets
az consumption budget list --output table

# Export cost data (create a recurring export)
az costmanagement export create \
  --name "daily-export" \
  --scope "/subscriptions/<sub-id>" \
  --storage-account-id "/subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageacct" \
  --storage-container "cost-exports" \
  --timeframe MonthToDate \
  --type ActualCost \
  --schedule-recurrence Daily \
  --schedule-status Active
```

### Portal Walkthrough -- Creating a Budget

1. Navigate to **Cost Management + Billing** > **Cost Management** > **Budgets**.
2. Click **+ Add**.
3. Select the **scope** (subscription or resource group).
4. Enter the **budget name**, **reset period** (monthly), and **amount**.
5. Set **alert conditions**: threshold type (Actual or Forecasted), % of budget, and action group.
6. Add **email recipients** for notifications.
7. Click **Create**.

---

## 7. Management Groups

[Microsoft Learn -- Management groups overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)

Management groups provide a governance scope **above subscriptions**. They let you organize subscriptions into a hierarchy and apply RBAC and Policy at scale.

### Hierarchy

```
Entra ID Tenant
 └── Root Management Group (auto-created, one per tenant)
       ├── MG: Platform
       │     ├── MG: Identity
       │     │     └── Subscription: Identity-Prod
       │     └── MG: Connectivity
       │           └── Subscription: Hub-Network
       ├── MG: Workloads
       │     ├── MG: Production
       │     │     ├── Subscription: App-A-Prod
       │     │     └── Subscription: App-B-Prod
       │     └── MG: Non-Production
       │           ├── Subscription: App-A-Dev
       │           └── Subscription: App-A-Test
       └── MG: Sandbox
             └── Subscription: Sandbox-01
```

### Key Facts

| Fact | Detail |
|---|---|
| **Max depth** | **6 levels** (not counting the root level or subscription level). |
| **Max management groups** | 10,000 per directory. |
| **Root management group** | Cannot be moved or deleted. Every tenant has exactly one. |
| **Default management group** | New subscriptions are placed in the root MG by default. You can change the default landing MG. |
| **Inheritance** | RBAC role assignments and Azure Policy assignments **cascade down** from MGs to child MGs, subscriptions, RGs, and resources. |
| **Single parent** | Each MG or subscription can have only one direct parent. |

### RBAC and Policy Inheritance

- A **Policy assigned at a management group** applies to all subscriptions, resource groups, and resources beneath it.
- An **RBAC role assigned at a management group** is inherited by all child scopes.
- You cannot override an inherited **Deny** policy at a lower scope. A child scope can add additional policies but cannot remove inherited ones.

### Who Can Manage?

- By default, only **Global Administrator** (after elevating access) and **root MG Owner** can manage the root management group.
- Recommendation: grant access at the root MG sparingly; use lower-level MGs for day-to-day governance.

### CLI Quick Reference

```bash
# Create a management group
az account management-group create \
  --name "mg-workloads" \
  --display-name "Workloads"

# Create a child management group
az account management-group create \
  --name "mg-production" \
  --display-name "Production" \
  --parent "mg-workloads"

# Move a subscription into a management group
az account management-group subscription add \
  --name "mg-production" \
  --subscription "<sub-id>"

# List management group hierarchy
az account management-group list --output table

# Show details of a management group
az account management-group show --name "mg-workloads" --expand --recurse

# Delete a management group (must be empty)
az account management-group delete --name "mg-sandbox"
```

---

## Exam Traps

These are frequently tested gotchas on AZ-104:

| Trap | Why it catches people |
|---|---|
| **Tags do NOT inherit.** | Many candidates assume RG tags flow down to resources. They don't -- you must use Azure Policy (Modify effect) to enforce inheritance. |
| **ReadOnly lock blocks more than "writes."** | It blocks any POST operation, including listing storage account keys, starting/stopping App Service, getting Application Gateway backend health. |
| **CanNotDelete lock + deployment history = blocked deployments.** | A CanNotDelete lock on a RG prevents auto-cleanup of deployment history. Once 800 entries are reached, new deployments fail. |
| **Policy effects evaluation order matters.** | Disabled -> Append/Modify -> Deny -> Audit. AuditIfNotExists and DeployIfNotExists run **after** the Resource Provider returns success. |
| **Management group depth = 6 levels.** | The limit is 6 levels, **excluding** the root level and the subscription level. Exam questions may try to trick you with "6 levels including root." |
| **Only Owner and User Access Administrator can manage locks.** | Contributor cannot create or delete locks, even though they can modify resources. |
| **Policy Deny beats everything except Disabled.** | If a policy with Deny effect targets a resource, no user -- even an Owner -- can create that resource unless the policy is removed, exempted, or disabled. |
| **Moving subscriptions to a new Entra ID tenant removes all RBAC assignments.** | Co-administrators, service admins, and all custom/built-in role assignments are wiped. |
| **RG location is metadata only.** | The resource group's region stores metadata for compliance but does not constrain where child resources are deployed. |
| **Budget alerts are not hard caps.** | Budgets generate alerts and can trigger actions, but they do **not** automatically stop spending or shut down resources. |
| **Initiatives vs. individual policies.** | You can assign individual policies **or** initiatives. Initiatives are recommended when multiple related policies should travel together. An initiative can contain both built-in and custom policies. |
| **Policy exemptions have an expiration date.** | A waiver exemption is temporary; a mitigated exemption indicates alternative compliance. Both can have optional expiration dates. |

---

## Sources

- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [Azure Policy effect basics](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-basics)
- [Azure Policy definition structure -- policy rule](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure-policy-rule)
- [Remediate non-compliant resources](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources)
- [Lock resources to protect infrastructure](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)
- [Use tags to organize Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)
- [Tag policies for Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-policies)
- [Manage resource groups -- Azure portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)
- [Move operation support for resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-support-resources)
- [Azure subscription and service limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits)
- [Management groups overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Tutorial: Create and manage budgets](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [Cost alerts -- monitor usage and spending](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/cost-mgt-alerts-monitor-usage-spending)
- [Azure Advisor cost recommendations](https://learn.microsoft.com/en-us/azure/advisor/advisor-reference-cost-recommendations)
- [Tag inheritance in Cost Management](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/enable-tag-inheritance)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 learning path -- Manage identities and governance](https://learn.microsoft.com/en-us/training/paths/az-104-manage-identities-governance/)
