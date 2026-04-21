# ARM Templates and Bicep

> **Exam mapping:** AZ-104 --> "Deploy and manage Azure compute resources" (20-25%)
> **One-liner:** Use Azure Resource Manager templates (JSON) or Bicep files (DSL) to declaratively define, deploy, modify, export, and convert infrastructure as code for repeatable, idempotent Azure deployments.
> **Related:** [08-virtual-machines](./08-virtual-machines.md) | [09-containers](./09-containers.md) | [10-app-service](./10-app-service.md)

---

## 1. Azure Resource Manager Overview

Azure Resource Manager (ARM) is the **deployment and management service** for Azure. Every request -- whether from the portal, CLI, PowerShell, SDKs, or REST APIs -- goes through ARM, which authenticates and authorises the request, then sends it to the appropriate resource provider.

**Reference:** [ARM overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)

### 1.1 Declarative vs Imperative

| Approach | Description | Example |
|----------|-------------|---------|
| **Declarative** | Define the *desired end state*; the engine figures out how to get there | ARM templates, Bicep, Terraform |
| **Imperative** | Define the *step-by-step commands* to execute | Azure CLI scripts, PowerShell scripts |

ARM templates and Bicep are **declarative** -- you describe *what* you want, not *how* to create it.

### 1.2 Idempotent Deployments

Deploying the same template multiple times produces the **same result**. If the resource already exists in the desired state, ARM makes no changes. This makes templates safe to re-run.

### 1.3 Deployment Modes

ARM supports two deployment modes, and the exam tests this heavily.

| Mode | Behaviour | Risk Level |
|------|-----------|------------|
| **Incremental** (default) | Adds or updates resources in the template; **leaves existing resources untouched** if not in the template | Low -- safe for iterative updates |
| **Complete** | Adds or updates resources in the template; **deletes resources in the resource group that are NOT in the template** | High -- can destroy resources unintentionally |

> **Exam trap:** Incremental mode is the **default**. Complete mode **deletes** resources not in the template. Only root-level templates support Complete mode -- linked/nested templates always use Incremental. Microsoft has announced that Complete mode will be gradually deprecated in favour of [deployment stacks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks).

**Key detail on Incremental redeployment:** When redeploying an *existing* resource in Incremental mode, **all properties are reapplied**. Properties you omit from the template are reset to their default values -- they are not left unchanged. This is a common misconception.

**Reference:** [Deployment modes](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-modes)

### 1.4 Deployment Scopes

Templates can target four scopes:

| Scope | CLI Command | Use Case |
|-------|-------------|----------|
| **Resource group** | `az deployment group create` | Most common -- deploy resources into an RG |
| **Subscription** | `az deployment sub create` | Create resource groups, assign policies |
| **Management group** | `az deployment mg create` | Apply policies across subscriptions |
| **Tenant** | `az deployment tenant create` | Configure tenant-wide settings, management group hierarchy |

**Reference:** [Deployment scopes](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-to-resource-group)

---

## 2. ARM Template Structure

ARM templates are **JSON files** with a defined schema. The maximum template size is **4 MB** after expansion.

**Reference:** [Template structure and syntax](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax)

### 2.1 Top-Level Elements

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { },
  "variables": { },
  "functions": [ ],
  "resources": [ ],
  "outputs": { }
}
```

| Element | Required | Description |
|---------|----------|-------------|
| `$schema` | Yes | URL of the JSON schema file that describes the template language version. Use `2019-04-01` for resource group deployments. |
| `contentVersion` | Yes | Version of the template content (e.g., `"1.0.0.0"`). Any value is allowed; used for your own tracking. |
| `parameters` | No | Values provided at deployment time. Max **256** parameters. |
| `variables` | No | Values constructed from parameters or other expressions for reuse within the template. |
| `functions` | No | User-defined functions available within the template. |
| `resources` | Yes | The Azure resources to deploy or update. |
| `outputs` | No | Values returned after deployment. Max **64** outputs. |

### 2.2 Parameters in Detail

```json
"parameters": {
  "storageName": {
    "type": "string",
    "minLength": 3,
    "maxLength": 24,
    "metadata": {
      "description": "Globally unique storage account name."
    }
  },
  "storageSku": {
    "type": "string",
    "defaultValue": "Standard_LRS",
    "allowedValues": [
      "Standard_LRS",
      "Standard_GRS",
      "Standard_ZRS",
      "Premium_LRS"
    ]
  },
  "location": {
    "type": "string",
    "defaultValue": "[resourceGroup().location]"
  }
}
```

**Supported parameter types:** `string`, `int`, `bool`, `object`, `array`, `secureString`, `secureObject`.

> Use `secureString` for passwords and secrets -- they are not logged in deployment history.

**Reference:** [Parameters in ARM templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/parameters)

### 2.3 Resources Section

```json
"resources": [
  {
    "type": "Microsoft.Storage/storageAccounts",
    "apiVersion": "2023-01-01",
    "name": "[parameters('storageName')]",
    "location": "[parameters('location')]",
    "sku": {
      "name": "[parameters('storageSku')]"
    },
    "kind": "StorageV2",
    "properties": {
      "supportsHttpsTrafficOnly": true
    }
  }
]
```

Key fields for each resource:
- **type** -- Resource provider namespace and type (e.g., `Microsoft.Storage/storageAccounts`).
- **apiVersion** -- The REST API version to use (determines available properties).
- **name** -- Resource name (must be unique within scope).
- **location** -- Azure region.

### 2.4 Outputs Section

```json
"outputs": {
  "storageEndpoint": {
    "type": "string",
    "value": "[reference(parameters('storageName')).primaryEndpoints.blob]"
  }
}
```

Outputs are displayed after deployment completes and can be consumed by other templates or scripts.

---

## 3. Bicep Fundamentals

Bicep is a **domain-specific language (DSL)** that compiles (transpiles) to ARM JSON. It is **not** a separate deployment engine -- under the hood, the same ARM API processes the resulting JSON.

**Reference:** [Bicep overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)

### 3.1 Why Bicep over ARM JSON?

- Cleaner, more concise syntax (roughly 50% fewer lines).
- No need to manage `dependsOn` manually -- Bicep infers dependencies from symbolic references.
- First-class support for modules, loops, and conditionals.
- Intellisense in VS Code with the Bicep extension.
- Transparent -- compiles 1:1 to ARM JSON; no state file (unlike Terraform).

### 3.2 Syntax Comparison: Storage Account

**ARM JSON:**

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 24
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {
        "supportsHttpsTrafficOnly": true
      }
    }
  ],
  "outputs": {
    "storageId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageName'))]"
    }
  }
}
```

**Bicep equivalent:**

```bicep
@minLength(3)
@maxLength(24)
param storageName string

param location string = resourceGroup().location

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
  }
}

output storageId string = sa.id
```

> Notice: no `$schema`, no `contentVersion`, no bracket expressions `[...]`, no explicit `dependsOn`. Bicep resolves references via the symbolic name `sa`.

### 3.3 Key Bicep Keywords

| Keyword | Purpose | Example |
|---------|---------|---------|
| `param` | Input parameter | `param location string = resourceGroup().location` |
| `var` | Variable | `var storageSuffix = 'sa${uniqueString(resourceGroup().id)}'` |
| `resource` | Resource declaration | `resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = { ... }` |
| `output` | Return value | `output blobEndpoint string = sa.properties.primaryEndpoints.blob` |
| `module` | Reference another Bicep file | `module network './network.bicep' = { ... }` |
| `existing` | Reference a resource that already exists | `resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = { name: 'myVnet' }` |
| `targetScope` | Deployment scope | `targetScope = 'subscription'` |

### 3.4 String Interpolation

Bicep uses `${}` inside single-quoted strings:

```bicep
var storageName = 'sa${uniqueString(resourceGroup().id)}'
```

This replaces ARM JSON's `concat()` and `format()` functions.

### 3.5 Conditional Deployments (`if`)

```bicep
param deployDiagnostics bool = true

resource diagStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = if (deployDiagnostics) {
  name: 'diagsa${uniqueString(resourceGroup().id)}'
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {}
}
```

When `deployDiagnostics` is `false`, the resource is **not created** (and not deleted if it already exists).

**Reference:** [Conditional deployment in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/conditional-resource-deployment)

### 3.6 Loops (`for`)

```bicep
param storageNames array = [
  'storageone'
  'storagetwo'
  'storagethree'
]

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageNames: {
  name: name
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {}
}]
```

Loops work on resources, modules, variables, properties, and outputs. You can combine `for` with `if` for filtered loops.

**Reference:** [Iterative loops in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/loops)

### 3.7 Modules

Modules encapsulate a Bicep file as a reusable unit:

```bicep
module storageModule './modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    storageName: 'myuniquesa'
    location: resourceGroup().location
  }
}

output storageId string = storageModule.outputs.storageId
```

Modules can also reference **Bicep registries** (Azure Container Registry) and **template specs**.

**Reference:** [Bicep modules](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules)

### 3.8 Referencing Existing Resources

```bicep
resource existingVnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: 'myExistingVnet'
}

output vnetId string = existingVnet.id
```

Use `existing` to read properties of a resource you did not create in this template (e.g., to get a subnet ID for a NIC).

---

## 4. Interpreting Templates

When reading a template on the exam, focus on:

### 4.1 Resource Type and API Version

```json
"type": "Microsoft.Compute/virtualMachines",
"apiVersion": "2023-09-01"
```

The **type** tells you what resource is being deployed. The **apiVersion** determines which properties are available.

### 4.2 Dependencies

**Explicit (ARM JSON):**

```json
"dependsOn": [
  "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
]
```

**Implicit (Bicep):** Bicep automatically detects dependencies from symbolic references:

```bicep
resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = { ... }

resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  properties: {
    networkProfile: {
      networkInterfaces: [{ id: nic.id }]  // implicit dependency on nic
    }
  }
}
```

### 4.3 Common Template Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `resourceGroup()` | Returns current RG object (name, location, id) | `"[resourceGroup().location]"` |
| `subscription()` | Returns current subscription object | `"[subscription().subscriptionId]"` |
| `resourceId()` | Constructs a resource ID | `"[resourceId('Microsoft.Storage/storageAccounts', parameters('name'))]"` |
| `reference()` | Returns the runtime state of a deployed resource | `"[reference(parameters('name')).primaryEndpoints.blob]"` |
| `concat()` | Concatenates strings or arrays | `"[concat('storage', uniqueString(resourceGroup().id))]"` |
| `format()` | Formats a string with placeholders | `"[format('sa{0}', uniqueString(resourceGroup().id))]"` |
| `uniqueString()` | Deterministic hash (13 chars) from input values | `"[uniqueString(resourceGroup().id)]"` |
| `parameters()` | Retrieves a parameter value | `"[parameters('location')]"` |
| `variables()` | Retrieves a variable value | `"[variables('vnetName')]"` |

> In Bicep, most of these are replaced by direct property access (e.g., `sa.properties.primaryEndpoints.blob` instead of `reference()`), but `resourceGroup()`, `subscription()`, and `uniqueString()` are still used directly.

**Reference:** [Template functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions)

---

## 5. Modifying Templates

### 5.1 Adding or Removing Resources

- **ARM JSON:** Add a new object to the `resources` array, or remove an existing one.
- **Bicep:** Add a new `resource` block, or delete one. Dependencies update automatically.

### 5.2 Changing Parameters

Add new parameters to accept dynamic input, update `defaultValue` or `allowedValues`, or convert hard-coded values into parameters for reusability.

### 5.3 Updating Properties

Change SKUs, sizes, settings, or tags within a resource definition. Always verify the property is valid for the chosen `apiVersion`.

### 5.4 Nested and Linked Templates (ARM JSON)

| Type | Description | Where Stored |
|------|-------------|--------------|
| **Nested** | Template defined inline within the parent template's `resources` array | Embedded in parent |
| **Linked** | Template stored externally and referenced via URL | Blob storage, template spec |

```json
{
  "type": "Microsoft.Resources/deployments",
  "apiVersion": "2021-04-01",
  "name": "linkedDeployment",
  "properties": {
    "mode": "Incremental",
    "templateLink": {
      "uri": "https://mystorageaccount.blob.core.windows.net/templates/storage.json",
      "contentVersion": "1.0.0.0"
    },
    "parameters": {
      "storageName": { "value": "linkedsa" }
    }
  }
}
```

> Linked/nested templates **always deploy in Incremental mode**, even if the parent uses Complete mode.

**Reference:** [Linked templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates)

---

## 6. Deploying Templates

### 6.1 Azure CLI

```bash
# Deploy ARM JSON template
az deployment group create \
  --resource-group myRG \
  --template-file main.json \
  --parameters storageName=myuniquesa

# Deploy Bicep file
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters storageName=myuniquesa

# Deploy with a parameter file (ARM JSON)
az deployment group create \
  --resource-group myRG \
  --template-file main.json \
  --parameters @main.parameters.json

# Deploy with a Bicep parameter file
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters main.bicepparam
```

**Reference:** [Deploy with Azure CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-cli)

### 6.2 Azure PowerShell

```powershell
# Deploy ARM JSON template
New-AzResourceGroupDeployment `
  -ResourceGroupName "myRG" `
  -TemplateFile "main.json" `
  -storageName "myuniquesa"

# Deploy Bicep file
New-AzResourceGroupDeployment `
  -ResourceGroupName "myRG" `
  -TemplateFile "main.bicep" `
  -storageName "myuniquesa"

# Deploy with parameter file
New-AzResourceGroupDeployment `
  -ResourceGroupName "myRG" `
  -TemplateFile "main.json" `
  -TemplateParameterFile "main.parameters.json"
```

**Reference:** [Deploy with PowerShell](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-powershell)

### 6.3 Azure Portal

Navigate to **Deploy a custom template** --> paste or upload your template --> fill in parameters --> Review + create. The portal UI generates a form from the template's `parameters` section.

### 6.4 Parameter Files

**ARM JSON parameter file** (`main.parameters.json`):

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageName": {
      "value": "myuniquesa"
    },
    "storageSku": {
      "value": "Standard_GRS"
    }
  }
}
```

**Bicep parameter file** (`main.bicepparam`):

```bicep
using './main.bicep'

param storageName = 'myuniquesa'
param storageSku = 'Standard_GRS'
```

> `.bicepparam` files use the `using` keyword to reference their corresponding Bicep file. This is a newer format -- ARM JSON parameter files work with both ARM and Bicep templates.

### 6.5 What-If Preview

The **what-if** operation shows what changes a deployment *would* make **without actually executing** them.

```bash
# CLI
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters storageName=myuniquesa

# Or inline with deploy
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --confirm-with-what-if
```

```powershell
# PowerShell
New-AzResourceGroupDeployment `
  -ResourceGroupName "myRG" `
  -TemplateFile "main.bicep" `
  -WhatIf
```

What-if output uses colour-coded change types:
- **Create** (green +) -- new resource
- **Delete** (red -) -- resource will be removed (Complete mode)
- **Modify** (yellow ~) -- property changes
- **No change** -- resource exists and matches
- **Ignore** -- not relevant to the deployment

> **Exam trap:** What-if is a **read-only preview** -- it does not execute any changes. It also validates the template for errors.

**Reference:** [What-if operation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if)

---

## 7. Export and Convert

### 7.1 Export Template from Portal

**From a resource:** Resource blade --> **Export template** --> view/download the ARM JSON.

**From a resource group:** Resource group --> **Export template** --> exports all resources in the RG as a single ARM template.

### 7.2 Export via CLI

```bash
# Export all resources in a resource group
az group export --name myRG --output-folder ./exported

# Export a specific resource
az group export --name myRG \
  --resource-ids "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mysa"
```

**Reference:** [Export template in CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/export-template-cli)

### 7.3 Export Limitations

Exported templates are a **starting point**, not production-ready:

- Not all resource types export cleanly (e.g., Azure Data Factory resources).
- **Passwords and secrets** are stripped -- you must add `secureString` parameters manually.
- Schema may not reflect the latest API version for a resource type.
- **200 resource limit** per resource group export.
- Some properties may be missing or require manual adjustment.
- Export is **not guaranteed to produce a re-deployable template** without modifications.

**Reference:** [Export template from portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/export-template-portal)

### 7.4 Convert ARM JSON to Bicep (Decompile)

```bash
az bicep decompile --file main.json
```

This produces a `main.bicep` file. Decompilation is **best-effort** -- you may need to fix warnings, errors, and missing features in the output.

**Reference:** [Decompile to Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/decompile)

### 7.5 Convert Bicep to ARM JSON (Build)

```bash
az bicep build --file main.bicep
```

This produces a `main.json` ARM template. This is what happens automatically when you deploy a `.bicep` file -- the CLI builds it before submitting to ARM.

### 7.6 Portal Export to Bicep

The Azure portal can also export directly to Bicep format (preview). Navigate to **Export template** and select the **Bicep** tab.

**Reference:** [Export Bicep from portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/export-bicep-portal)

---

## Exam Traps

1. **Complete vs Incremental mode** -- Complete mode **deletes** resources not in the template. Incremental (default) leaves them alone. This is the most tested concept. Even in Incremental mode, properties you omit from an existing resource are **reset to defaults**, not preserved.

2. **What-if does not execute** -- It is a preview/validation tool only. It shows what *would* change but makes no modifications.

3. **Bicep is ARM, not Terraform** -- Bicep compiles to ARM JSON. It uses the same ARM deployment engine. There is no state file (unlike Terraform). Bicep is a Microsoft-native tool.

4. **Export limitations** -- Exported templates are not production-ready. Secrets are removed, some resource types are unsupported, and properties may be incomplete. Always review and test before redeploying.

5. **Parameter file formats** -- ARM JSON uses `.parameters.json` with a `$schema`; Bicep uses `.bicepparam` with a `using` keyword. ARM JSON parameter files work with both template types.

6. **Linked/nested templates always use Incremental mode** -- Even if the parent deployment uses Complete mode.

7. **`az bicep decompile`** converts ARM JSON --> Bicep; **`az bicep build`** converts Bicep --> ARM JSON. Do not confuse the direction.

8. **Deployment scope matters** -- `az deployment group create` targets a resource group. To create resource groups themselves, you need `az deployment sub create` (subscription scope).

9. **Template size limit** -- 4 MB after expansion for ARM JSON templates.

10. **`secureString` vs `string`** -- Use `secureString` for passwords and keys. Values are not logged in deployment history or accessible via `reference()`.

---

## Sources

- [Azure Resource Manager overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)
- [ARM template structure and syntax](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax)
- [Parameters in ARM templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/parameters)
- [Deployment modes](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-modes)
- [Bicep overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
- [Bicep file structure and syntax](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/file)
- [Conditional deployment in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/conditional-resource-deployment)
- [Iterative loops in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/loops)
- [Bicep modules](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules)
- [Deploy with Azure CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-cli)
- [Deploy with PowerShell](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-powershell)
- [What-if operation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if)
- [Linked templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates)
- [Export template from portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/export-template-portal)
- [Export template in CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/export-template-cli)
- [Export Bicep from portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/export-bicep-portal)
- [Decompile to Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/decompile)
- [Bicep CLI commands](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-cli)
- [Template functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
