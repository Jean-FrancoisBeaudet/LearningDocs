# Managed Identities

> **Exam mapping:** AZ-204 *(Implement secure cloud solutions)* · AZ-305 · AI-102 *(Secure AI services)*
> **One-liner:** Azure-managed service principals with no credentials to rotate — get a token from the IMDS endpoint and use it.
> **Related:** [app registrations](02-app-registrations-and-service-principals.md) · [auth protocols](03-authentication-protocols.md) · [RBAC](05-rbac-and-authorization.md)

## What a managed identity is

A managed identity (MI) is **a service principal in Entra ID whose credentials are owned and rotated by Azure**. Your code asks the local Azure metadata service for a token; Azure returns one signed by Entra. You never see a secret.

Two kinds:

| | **System-assigned** | **User-assigned** |
|---|---------------------|-------------------|
| Lifecycle | Tied 1:1 to the parent resource (deleted with it) | Standalone Azure resource (`Microsoft.ManagedIdentity/userAssignedIdentities`) |
| Scope | One resource | Any number of resources can attach the same UAMI |
| Use when | Single-app, simple lifecycle, easy cleanup | Shared identity across an app fleet, blue-green deploys, pre-provisioned RBAC |
| Identity sharing | Impossible | Designed for it |
| Federation (FIC) | **Not supported** | **Supported** — trust GitHub/Kubernetes/etc. via federated credential |

A resource can have **one system-assigned + many user-assigned** at the same time.

## Services that support MI (selected)

Most modern Azure compute and PaaS supports MI. Exam-relevant:

- **Compute:** VMs, VMSS, AKS (workload identity), Container Apps, Container Instances, App Service / Functions, Batch.
- **Data:** Storage (as data plane authority for some scenarios), SQL DB, Cosmos DB (data-plane RBAC), Service Bus, Event Hubs, Event Grid, Key Vault.
- **AI:** Azure OpenAI / AI Foundry, AI Search, Document Intelligence, Speech, Vision.
- **Integration:** Logic Apps, Data Factory, Synapse, APIM, App Configuration.

> Some legacy resources still don't support MI — check the support matrix before assuming.

## How code gets a token (IMDS)

Inside an Azure VM/App Service/etc., the identity endpoint is reachable at:

| Host kind | Endpoint |
|-----------|----------|
| VM / VMSS | `http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=...` |
| App Service / Functions | `IDENTITY_ENDPOINT` env var + `IDENTITY_HEADER` |
| AKS workload identity | Federated token mounted in pod, exchanged at Entra `/token` |
| Container Apps | `IDENTITY_ENDPOINT` env var |

Always go through **`DefaultAzureCredential`** (or its siblings) — it abstracts the per-host endpoint differences.

```csharp
// .NET — the canonical pattern
var credential = new DefaultAzureCredential();
var blobClient = new BlobServiceClient(
    new Uri("https://mystorage.blob.core.windows.net"), credential);

// Using a UAMI explicitly:
var cred = new DefaultAzureCredential(new DefaultAzureCredentialOptions {
    ManagedIdentityClientId = "<uami-client-id>"
});
```

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
cred = DefaultAzureCredential()
client = BlobServiceClient("https://mystorage.blob.core.windows.net",
                           credential=cred)
```

`DefaultAzureCredential` chain (in order):
1. `EnvironmentCredential` (set `AZURE_CLIENT_ID/SECRET/TENANT_ID`)
2. `WorkloadIdentityCredential` (AKS / federated)
3. `ManagedIdentityCredential` (IMDS)
4. `SharedTokenCacheCredential`
5. `VisualStudio` / `AzureCli` / `AzurePowerShell` / `AzureDeveloperCli` (local dev)
6. `InteractiveBrowserCredential` (off by default in `DefaultAzureCredential`)

Set `AZURE_TOKEN_CREDENTIALS` (or `excludeXxxCredential` options) to lock down which providers are allowed in production.

## Federated identity credentials (FIC) on a managed identity

User-assigned MIs can trust an external token issuer — same model as on app registrations. Common scenarios:

| External workload | `issuer` | `subject` example |
|-------------------|----------|-------------------|
| **GitHub Actions** | `https://token.actions.githubusercontent.com` | `repo:contoso/app:ref:refs/heads/main` or `repo:contoso/app:environment:prod` |
| **AKS workload identity** | OIDC issuer URL of the AKS cluster | `system:serviceaccount:<ns>:<sa>` |
| **Other Kubernetes** | Cluster's OIDC issuer | `system:serviceaccount:<ns>:<sa>` |
| **GitLab CI** | `https://gitlab.com` (or self-hosted) | `project_path:group/project:ref_type:branch:ref:main` |
| **Azure DevOps** | `https://vstoken.dev.azure.com/<org-id>` | `sc://<org>/<project>/<service-connection-name>` |

```bash
# Federate a UAMI to a GitHub branch
az identity federated-credential create \
  --name gh-prod \
  --identity-name uami-deploy \
  --resource-group rg-iam \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:contoso/app:environment:prod \
  --audiences api://AzureADTokenExchange
```

GitHub Actions then runs:

```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}     # the UAMI client ID
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
# No client-secret. The OIDC token from GitHub is exchanged for an Entra token.
```

> Limit: **20 FICs per managed identity** (same cap as app registrations).

## AKS workload identity (the modern pattern)

Replaces the legacy "Pod Identity" (deprecated, do not use). Steps:

1. Enable OIDC issuer + workload identity on the AKS cluster.
2. Create a UAMI; grant it Azure RBAC.
3. Add a federated credential trusting the cluster's OIDC issuer + the K8s service account subject.
4. Annotate the K8s SA with `azure.workload.identity/client-id: <uami-client-id>`.
5. Label pods with `azure.workload.identity/use: "true"` so the webhook injects the projected token volume.
6. App uses `WorkloadIdentityCredential` (or `DefaultAzureCredential`).

## Bicep snippets

```bicep
// System-assigned MI on a Function App + RBAC to a storage account
resource func 'Microsoft.Web/sites@2023-12-01' = {
  name: 'fn-orders'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: { serverFarmId: plan.id, siteConfig: { /* ... */ } }
}

resource roleAssign 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storage.id, func.id, 'blob-reader')
  scope: storage
  properties: {
    principalId: func.identity.principalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1') // Storage Blob Data Reader
  }
}

// User-assigned MI attached to a Container App
resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'uami-shared'
  location: location
}

resource app 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'ca-orders'
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: { '${uami.id}': {} }
  }
  // ...
}
```

## Common pitfalls

- **Identity cache lag** — newly granted RBAC can take 5–10 minutes to take effect (token cached in caller; backend permission cache lag).
- **Wrong client ID with multiple UAMIs** — when more than one UAMI is attached, IMDS needs an explicit `client_id` query param or `ManagedIdentityClientId` in the SDK.
- **No managed identity in local dev** — that's why `DefaultAzureCredential` falls back to AzureCliCredential for `az login` users.
- **Resource doesn't support MI** — check the [services that support MI](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-status) list.

## Exam traps

- **System-assigned MIs do not support workload identity federation** (FIC) — only user-assigned do.
- **Azure RBAC role on the MI is required** for it to do anything; assigning an MI without granting roles = 403s.
- **Cosmos DB data-plane access requires its own RBAC system** (`az cosmosdb sql role assignment`), not generic Azure RBAC.
- **Key Vault** access can be granted via either RBAC roles (preferred, default for new vaults from API `2026-02-01`) **or** legacy access policies — pick one model per vault.
- **AKS Pod Identity is deprecated** — use AKS Workload Identity.
- **`DefaultAzureCredential` fallback order matters** — in prod containers, set env vars or use `ManagedIdentityCredential` directly to avoid surprise fallbacks.
- **MI tokens are scoped to a single resource** (`scope=https://vault.azure.net/.default`, etc.) — not global.

## Sources

- [Managed identities overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [Services that support MI](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-status)
- [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Configure FIC on UAMI](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity)
- [AKS workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [DefaultAzureCredential (Azure SDK)](https://learn.microsoft.com/en-us/azure/developer/intro/azure-developer-credentials)
