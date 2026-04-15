# App Registrations & Service Principals

> **Exam mapping:** AZ-204 *(Implement secure cloud solutions — auth)* · AZ-305
> **One-liner:** App registration = the global "blueprint" of an app; service principal = the per-tenant identity that actually authenticates and gets RBAC.
> **Related:** [auth protocols](03-authentication-protocols.md) · [managed identities](04-managed-identities.md) · [RBAC](05-rbac-and-authorization.md)

## App object vs service principal — the key distinction

| | **Application object** | **Service principal (SP)** |
|---|------------------------|----------------------------|
| Where it lives | The **home tenant** (one global object) | **Each tenant** that consents to the app gets its own SP |
| Microsoft Graph entity | `application` | `servicePrincipal` |
| Holds | App config: redirect URIs, secrets/certs, requested API permissions, branding, app roles, optional claims | The local instance: granted permissions, role assignments, sign-in policy |
| Visible in | App registrations blade | **Enterprise applications** blade |
| Identifier | **App (client) ID** — same in all tenants | **Object ID** — different per tenant |

> Single-tenant apps still have both — the app object and an SP, both in your tenant.
> Multi-tenant apps have **one** app object in the publisher tenant + **N** SPs across consumer tenants (created on first admin consent / first user sign-in).

## Tenancy choices

| Setting | Audience |
|---------|----------|
| `AzureADMyOrg` | Single tenant — only your tenant's users/SPs |
| `AzureADMultipleOrgs` | Multi-tenant — any work/school account (any Entra tenant) |
| `AzureADandPersonalMicrosoftAccount` | Above + personal MSAs (Outlook.com, Xbox) — uses `/common` endpoint |
| `PersonalMicrosoftAccount` | MSAs only |

## Redirect URIs

- Must match exactly (down to trailing slash and case for path).
- Platform types: **Web** (uses client secret/cert), **SPA** (uses PKCE, no secret), **Mobile/Desktop** (uses `https://login.microsoftonline.com/common/oauth2/nativeclient` or custom scheme), **Public client**.
- Wildcards: limited support, mostly for dev. Avoid in prod.
- Localhost over HTTP allowed (dev only).

## Credentials — three options (in preference order)

| Credential | Lifetime | Storage | When to use |
|------------|----------|---------|-------------|
| **Federated identity credential (FIC)** — workload identity federation | No secret at all (token exchange) | None — trusts an external IdP (GitHub, AKS, GitLab, Azure DevOps, Kubernetes) | **Always preferred for CI/CD and external workloads.** Zero secret rotation. |
| **Certificate** | Up to ~years (you choose) | Key Vault + cert auto-rotation | Better than secrets when FIC isn't possible. |
| **Client secret** | Max **24 months** (UX caps shorter for new ones — 6/12/24) | Key Vault — never inline | Last resort. Rotate aggressively. |

```bash
# Add a federated credential trusting GitHub Actions
az ad app federated-credential create --id <APP_ID> --parameters '{
  "name": "gh-main-deploy",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:contoso/myapp:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

> Each app/MI supports up to **20 federated identity credentials**.

## API permissions: delegated vs application

| | **Delegated** | **Application** |
|---|---------------|-----------------|
| When the app calls API | On behalf of a signed-in user | As itself, no user present |
| Token type | User token (carries `scp` claim) | App-only token (carries `roles` claim) |
| Effective permissions | Intersection of (app permission) ∩ (user's actual rights) | Full granted scope, no user filter |
| Consent | User consent (some) or admin consent | **Admin consent always required** |
| Auth flow | Auth code + PKCE, OBO, device code, etc. | Client credentials |

Example Graph permissions:

| Need | Permission | Type |
|------|-----------|------|
| Read the signed-in user's profile | `User.Read` | Delegated |
| Read all users in tenant from a daemon | `User.Read.All` | Application (admin consent) |
| Send mail as user | `Mail.Send` | Delegated |
| Send mail as anyone in tenant | `Mail.Send` | Application |

### Admin consent

- Required for: app permissions, delegated permissions marked "admin consent required", any permissions when admin consent workflow is enabled.
- Granted via **admin consent URL** `https://login.microsoftonline.com/{tenant}/v2.0/adminconsent?client_id={app_id}` or the Entra admin center "Grant admin consent" button.
- Recorded as an `oauth2PermissionGrant` (delegated, tenant-wide) or `appRoleAssignment` (application).

## App roles & scopes (exposing your own API)

When **your** app is the resource:

- Define **scopes** (`oauth2PermissionScopes`) for delegated access — what users can grant.
- Define **app roles** (`appRoles`) for app-only access (other apps/SPs assigned to a role) **and** for users/groups assigned to a role (claim `roles` on the user's token).
- Validate `aud` (audience) and either `scp` (delegated) or `roles` (app) in the API.

```jsonc
// Manifest fragment exposing one scope and one app role
"oauth2PermissionScopes": [{
  "id": "11111111-1111-1111-1111-111111111111",
  "value": "Orders.Read",
  "type": "User",
  "adminConsentDisplayName": "Read orders",
  "userConsentDisplayName": "Read your orders"
}],
"appRoles": [{
  "id": "22222222-2222-2222-2222-222222222222",
  "value": "Orders.Write.All",
  "displayName": "Write all orders",
  "allowedMemberTypes": ["Application"]
}]
```

## Managed Identity vs SP with secret

| Feature | Managed Identity | App Reg + secret |
|---------|------------------|------------------|
| Secret to manage | None (Azure-managed) | Yes |
| Cross-tenant | No (single tenant where MI lives, except via FIC) | Yes |
| Federation to external IdP | Yes (since 2023 GA) — see [`04-managed-identities.md`](04-managed-identities.md) | n/a |
| Use from outside Azure | No (unless via FIC trusting MI) | Yes |
| Role assignments | Yes | Yes |

**Rule of thumb:** if it runs in Azure, use a managed identity. If it runs outside Azure, use an app reg + **federated credential**. Only use a secret/cert if neither is possible.

## CLI quick reference

```bash
# Create app reg + SP in one go
APP_ID=$(az ad app create --display-name "myapi" --query appId -o tsv)
az ad sp create --id $APP_ID

# Add an Azure RBAC role to the SP
az role assignment create --assignee $APP_ID \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/$SUB/resourceGroups/rg-data

# Grant a Microsoft Graph application permission
az ad app permission add --id $APP_ID \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions df021288-bdef-4463-88db-98f22de89214=Role  # User.Read.All app

az ad app permission admin-consent --id $APP_ID
```

## Exam traps

- **App ID = client ID = same across tenants.** SP `objectId` differs per tenant.
- **Multi-tenant SP creation happens on first sign-in/consent**, not when the app is registered.
- **Application permissions always require admin consent**, no exception.
- **Client secrets are capped at 24 months**; the portal hides longer values that older `az ad app credential reset` calls could create.
- **You cannot extend a client secret** — only create a new one and roll over.
- **Federated credentials and managed identities both reduce secret sprawl.** Prefer FIC for GitHub Actions/AKS/external; prefer MI for in-Azure compute.
- A **certificate uploaded to an app reg** can be expressed as either thumbprint + private key, or via a `keyCredential` JWK; rolling certs requires brief overlap.

## Sources

- [Application and service principal objects](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Permissions and consent](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)
- [App roles vs scopes](https://learn.microsoft.com/en-us/entra/identity-platform/howto-add-app-roles-in-apps)
- [Federated credential limits](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-considerations)
