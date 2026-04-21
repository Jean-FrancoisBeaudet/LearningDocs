# Exam Cheatsheet

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 *(→ AI-103)* · AI-300
> **One-liner:** The traps Microsoft exams love to set on Microsoft Entra ID, with the "real answer" for each.
> **Related:** links out to every other note in this folder.

## The 20 things most likely to show up

### 1. Brand drift
- **Azure AD = Microsoft Entra ID** (rename July 2023). [`00-overview.md`](00-overview.md)
- **Azure AD B2C ≠ Entra External ID** — separate products. New CIAM = **Entra External ID**; B2C closed to new tenants **May 1, 2025**, supported through 2030.
- **MSOnline / AzureAD PowerShell are retired** (mid-2025) — answers must use **Microsoft Graph PowerShell** or **Microsoft Entra PowerShell**.
- **Azure AD Connect → Entra Connect Sync** (and the lighter **Entra Cloud Sync**).

### 2. SKU gates
- **Conditional Access:** P1+. [`00-overview.md`](00-overview.md)
- **Dynamic groups:** P1+.
- **Identity Protection + risk policies:** **P2**. [`10-identity-protection-and-monitoring.md`](10-identity-protection-and-monitoring.md)
- **PIM** (Entra roles, Azure roles, Groups): **P2** for everyone with eligibility. [`07-privileged-identity-management.md`](07-privileged-identity-management.md)
- **Entitlement management, lifecycle workflows, full access reviews:** **Entra ID Governance** add-on. [`08-identity-governance.md`](08-identity-governance.md)
- **Entra Suite** = P2 + Governance + Internet Access + Private Access + Verified ID Premium.

### 3. App reg vs service principal
- App object = **global blueprint** (one in publisher tenant). [`02-app-registrations-and-service-principals.md`](02-app-registrations-and-service-principals.md)
- Service principal = **per-tenant identity** (one per consumer tenant on first consent).
- Same `clientId` everywhere; SP `objectId` differs per tenant.

### 4. Credentials priority
1. **Federated identity credential** (workload identity federation) — for GitHub Actions / AKS / external workloads.
2. **Certificate** in Key Vault.
3. **Client secret** — last resort, max **24 months**, never inline.

### 5. Managed identity rules
- **System-assigned** = 1:1 with parent resource, deleted with it. [`04-managed-identities.md`](04-managed-identities.md)
- **User-assigned** = standalone, shareable, **only kind that supports federated credentials**.
- Up to **20 federated credentials** per identity (MI or app reg).
- AKS: use **Workload Identity**, not deprecated Pod Identity.
- `DefaultAzureCredential` chain falls back to AzureCli locally — set env vars in prod to lock down.

### 6. Two RBAC systems
- **Entra (directory) roles** for identity admin (Global Admin, User Admin). [`05-rbac-and-authorization.md`](05-rbac-and-authorization.md)
- **Azure RBAC** for resource access (Owner, Contributor, Reader, data plane roles).
- Global Admin **doesn't** auto-grant Azure RBAC — must elevate at root.
- **Owner** can grant access; **Contributor** cannot. Use **RBAC Administrator** for delegated grant rights.
- Limits: 4 000 role assignments per subscription, 5 000 custom roles per tenant.

### 7. ABAC conditions
- Add conditions to RBAC for storage tags / Key Vault attributes. [`05-rbac-and-authorization.md`](05-rbac-and-authorization.md)
- Use `--condition-version 2.0`.
- Reduces role-assignment sprawl.

### 8. Key Vault auth model
- **New vaults from API `2026-02-01` default to Azure RBAC.** Old vaults still on access policies until migrated.
- All control-plane API versions before `2026-02-01` retire **February 27, 2027**.

### 9. OAuth flows
- **Auth code + PKCE** for SPA/mobile/web. [`03-authentication-protocols.md`](03-authentication-protocols.md)
- **Client credentials** for daemons; only `/.default` scope.
- **OBO** for tiered APIs.
- **Device code** for browserless (CLI/IoT).
- **ROPC + Implicit are deprecated** — never the right answer.

### 10. Token validation
- Validate signature, `iss`, `aud`, `exp`, `nbf`, `tid`.
- App-only token → check `roles`. Delegated → check `scp`.
- Use `Microsoft.Identity.Web` in ASP.NET Core to get this for free.
- ID tokens are **not** for calling APIs.

### 11. CAE (Continuous Access Evaluation)
- Tokens revoked in near-real-time on password change, role change, MFA enable, risk detection. [`03-authentication-protocols.md`](03-authentication-protocols.md)
- Apps must handle **claims challenges** (`WWW-Authenticate: Bearer error="insufficient_claims"`) and reacquire.
- Effective lifetime ~28h on CAE-aware resources, with instant revocation.

### 12. Conditional Access design
- All matching policies AND together; **Block** wins. [`06-conditional-access-and-mfa.md`](06-conditional-access-and-mfa.md)
- **Always exclude break-glass accounts**.
- Always start in **Report-only**, monitor with Workbook, then enable.
- **Per-user MFA is deprecated** — use CA exclusively.
- **Authentication strength** (MFA / Passwordless MFA / Phishing-resistant MFA) is the right control for "MFA flavor" requirements.

### 13. Phishing-resistant MFA
- Satisfied by **FIDO2/passkey, Windows Hello for Business, certificate-based auth**.
- **Not** satisfied by Authenticator push, SMS, or voice.
- Microsoft's recommended bar for all admins.

### 14. PIM essentials
- **Eligible vs Active** — eligible has zero standing privilege until activation. [`07-privileged-identity-management.md`](07-privileged-identity-management.md)
- Activation may require **MFA, justification, ticket, approval**.
- Modern pattern: PIM points to a **Conditional Access authentication context**, not the per-role MFA toggle.
- **PIM for Groups** is the lever for non-built-in roles or bundles of roles.

### 15. Identity Governance pieces
- **Catalog → Access package → Policy → Assignment**. [`08-identity-governance.md`](08-identity-governance.md)
- **Lifecycle workflows** automate joiner/mover/leaver — uses `employeeHireDate` / `employeeLeaveDateTime`.
- **Cross-tenant sync (XTS)** creates **external members** (not guests) for multi-tenant orgs.
- **SoD checks** at request time only.

### 16. External identity decision tree
- Partner needs SharePoint/Teams access → **B2B collaboration** (guests). [`09-external-identities.md`](09-external-identities.md)
- Joint Teams shared channel → **B2B direct connect** (no guest object).
- Customer-facing app → **Entra External ID** (not B2C).
- Subsidiary tenants share staff → **XTS** + B2B direct connect.

### 17. Cross-tenant access settings
- Configure **inbound + outbound** B2B and B2B-DC per external tenant.
- **Trust home tenant's MFA / compliant device / hybrid-joined claim** to skip re-prompting partner users.
- **Tenant restrictions v2** blocks your users from signing into other tenants from your network/devices.

### 18. Identity Protection migration
- **Legacy ID Protection user-risk and sign-in-risk policies retire October 1, 2026.** [`10-identity-protection-and-monitoring.md`](10-identity-protection-and-monitoring.md)
- Recreate as **Conditional Access** policies consuming `User risk` / `Sign-in risk` conditions.
- Workload identity risk needs **Workload ID Premium**.

### 19. Logs & retention
- Free retention **7 days**, P1/P2 **30 days**. Long term → Diagnostic settings to **Log Analytics / Storage / Event Hub**.
- Stream all categories: Sign-in (interactive + non-interactive + SP + MI), Audit, Provisioning, RiskyUsers, UserRiskEvents.
- Tenant-level resource path: `/providers/Microsoft.aadiam/diagnosticSettings/default`.

### 20. Defender for Identity vs Identity Protection
- **Defender for Identity** = on-prem AD/ADCS/ADFS sensor.
- **Identity Protection** = cloud Entra signals.
- Both feed XDR + Conditional Access risk.

## Scenario drill patterns

**"Senior dev needs occasional Subscription Owner."**
→ Create **eligible PIM assignment**; activation requires **approval + phishing-resistant MFA + ticket**, max 4h. Not "give them Owner permanently".

**"GitHub Actions deploys to Azure with no client secret."**
→ Create UAMI **federated identity credential** trusting the GitHub OIDC issuer + `repo:org/repo:environment:prod`. Use `azure/login@v2` with `client-id` + `tenant-id`. No `client-secret`.

**"Function App calling Key Vault and Storage."**
→ **System-assigned managed identity** + RBAC roles (Key Vault Secrets User, Storage Blob Data Reader). `DefaultAzureCredential` in code. Disable Key Vault access policies; use RBAC. Disable storage shared key.

**"Partner contractor needs to use our internal app for 90 days."**
→ **Entitlement management access package** with B2B policy; expires after 90 days; manager approval; quarterly access review. Connected organization for the partner.

**"Build a public-facing customer portal with Google + email sign-in."**
→ **Entra External ID** (not B2C). MSAL + `<tenant>.ciamlogin.com` authority. Custom auth extension if you need to enrich the token from your CRM.

**"Block legacy auth across the org."**
→ CA policy: All users (exclude break-glass), All apps, **Client apps = Exchange ActiveSync + Other clients**, Grant = Block. Roll out in Report-only first.

**"Force MFA only when user signs in from a new country."**
→ CA with **Sign-in risk** condition (Identity Protection signals 'unfamiliar location'). Grant: Require MFA. Self-remediation closes risk on success.

**"Daemon needs to read all users from Microsoft Graph."**
→ App registration + **Application** permission `User.Read.All` + **admin consent**. Client credentials flow with `https://graph.microsoft.com/.default` scope. Authenticate with **federated credential** (or Managed Identity if running in Azure).

**"Mid-tier API needs to call Graph as the user."**
→ **OBO flow.** Mid-tier holds delegated permission to Graph; admin-consents; uses MSAL `AcquireTokenOnBehalfOf` with the incoming user assertion.

**"Audit who activated which privileged role last quarter."**
→ Microsoft Graph `/roleManagement/directory/roleAssignmentScheduleRequests` or PIM audit history. Or KQL on Log Analytics `AuditLogs` filtered by `OperationName contains "PIM"`.

## Final study tips

- Memorize **SKU → feature** mapping cold (P1 vs P2 vs Governance vs Suite).
- Memorize the **OAuth flow → app shape** matrix.
- Know the **deprecations**: per-user MFA, ROPC, implicit, MSOnline/AzureAD PowerShell, AAD Pod Identity, Azure AD B2C for new tenants.
- Know **what's tenant-level vs subscription-level vs resource-level** — Microsoft loves hiding the question in this distinction.
- For any question with "guest", "partner", "external user", "customer" — pick **B2B collab / B2B-DC / XTS / External ID** correctly. The four-way decision is exam catnip.
- Be able to read a Bicep snippet for a Function App with `identity: { type: 'SystemAssigned' }` and a `roleAssignment` block, and spot misconfigs (wrong `principalType`, wrong `roleDefinitionId`, scope inheritance issues).

## Sources

All linked from the individual topic notes. Start with:
- [Microsoft Entra fundamentals (Microsoft Learn)](https://learn.microsoft.com/en-us/entra/fundamentals/)
- [AZ-204 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)
- [AZ-305 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305)
- [Microsoft identity platform docs](https://learn.microsoft.com/en-us/entra/identity-platform/)
- [Entra ID Governance docs](https://learn.microsoft.com/en-us/entra/id-governance/)
