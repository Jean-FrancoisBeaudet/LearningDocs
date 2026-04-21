# Microsoft Entra ID — Overview

> **Exam mapping:** AZ-204 *(Implement user authentication and authorization)* · AZ-305 *(Design identity, governance, and monitoring solutions)* · AI-102 / AI-103 *(Secure AI solutions)* · AI-300 *(GenAIOps identity & governance)*
> **One-liner:** Microsoft's cloud identity and access management (IAM) service — the IdP that fronts every Azure, Microsoft 365, and (via OIDC/SAML) third-party SaaS sign-in.
> **Related:** [tenants & users](01-tenants-users-groups.md) · [app registrations](02-app-registrations-and-service-principals.md) · [auth protocols](03-authentication-protocols.md) · [exam cheatsheet](11-exam-cheatsheet.md)

## What Entra ID is

Microsoft Entra ID is the **rebranded Azure Active Directory** (rename announced **July 11, 2023**; the underlying service, tenant IDs, app IDs, and APIs are unchanged). "Entra" is also the umbrella brand for an entire identity product family (Entra ID, Entra ID Governance, Entra External ID, Entra Verified ID, Entra Permissions Management, Entra Internet/Private Access, Entra Workload ID).

Core responsibilities:

| Capability | Example |
|------------|---------|
| **Authentication** | OIDC/OAuth2/SAML/WS-Fed sign-in for users and workloads |
| **Authorization** | Issues access tokens whose `scp`/`roles` claims drive app-side checks |
| **Directory** | Stores users, groups, devices, apps, service principals |
| **Federation hub** | Federates on-prem AD via Entra Connect Sync / Cloud Sync |
| **Governance** | Lifecycle workflows, access reviews, PIM (with right SKU) |
| **Posture** | Conditional Access, Identity Protection, CAE |

## Brand & terminology drift (call out on exams)

| Old | Current (use this) |
|-----|--------------------|
| Azure Active Directory / Azure AD | **Microsoft Entra ID** |
| Azure AD B2C | **Microsoft Entra External ID** (B2C closed to new tenants **May 1, 2025**, supported through **at least 2030**) |
| Azure AD Connect | **Microsoft Entra Connect Sync** (also: **Entra Cloud Sync** as the lightweight successor) |
| MSOnline / AzureAD PowerShell | **Microsoft Graph PowerShell** or **Microsoft Entra PowerShell** (both legacy modules retired by mid-2025) |
| portal.azure.com → AAD blade | **entra.microsoft.com** (Entra admin center) |
| Azure AD Premium P1/P2 | **Microsoft Entra ID P1 / P2** |
| Enterprise application | (unchanged) the **service principal** UX surface |

## SKUs and pricing

Pricing is per user/month; annual commitment unless noted. Verify on the [Entra pricing page](https://www.microsoft.com/security/business/microsoft-entra-pricing) for region-specific numbers.

| SKU | List price | What you get on top of the previous tier |
|-----|-----------|------------------------------------------|
| **Entra ID Free** | $0 | SSO, MFA via Security Defaults, basic reports, B2B (50k MAU free), 500 k objects. **No** Conditional Access, **no** group-based licensing. |
| **Entra ID P1** | ~$6 | Conditional Access, dynamic groups, group-based licensing, self-service password reset with on-prem write-back, Entra Connect Sync, Application Proxy, Cloud App Discovery. |
| **Entra ID P2** | ~$9 | **Identity Protection** (risk policies), **Privileged Identity Management** (PIM), basic access reviews on Entra roles. |
| **Entra ID Governance** | ~$7 add-on (requires P1 or P2 base) | Entitlement management (access packages), full access reviews on any resource, lifecycle workflows, separation-of-duties checks, machine-learning recommendations. |
| **Entra Suite** | ~$12 | Bundles **P2 + Governance + Internet Access + Private Access + Verified ID Premium**. Cheaper than buying components separately. |
| **External ID (CIAM)** | MAU-based: first **50 000 MAU free**, then per-MAU pricing | Customer-facing identity (Entra External ID). Separate billing model. |

> Bundling: M365 E3 includes Entra ID P1; M365 E5 includes Entra ID P2; M365 E7 (GA May 1, 2026) includes the **full Entra Suite**.

## The tenant concept

A **tenant** = a dedicated, isolated instance of Entra ID. One tenant per Microsoft 365 org by default. Identified by:

- **Tenant ID** (GUID) — immutable.
- **Initial domain** — `<name>.onmicrosoft.com`, can never be removed.
- **Custom verified domains** — bring your own (`contoso.com`); requires DNS TXT proof.

Tenant **types**:

| Type | Use case |
|------|----------|
| **Workforce** | Default — employees, B2B guests. |
| **External (CIAM)** | New tenant flavor for Entra External ID customer-facing apps. Distinct from B2C tenants. |
| **Azure AD B2C** | Legacy CIAM tenant — closed to new customers May 2025. |

A subscription is **trusted** by exactly one home tenant at a time (movable). Resources in that subscription authenticate principals from that tenant (or guests invited into it).

## Exam mapping cheatsheet

| Exam | What they ask about Entra ID |
|------|------------------------------|
| **AZ-204** | App registration vs SP, MSAL flows (auth code + PKCE, client credentials, OBO), managed identities + Key Vault, Microsoft Graph permissions (delegated vs application), `DefaultAzureCredential`. |
| **AZ-305** | SKU choice, Conditional Access design, PIM, hybrid identity (Connect Sync vs Cloud Sync), B2B vs B2B direct connect vs External ID, identity governance. |
| **AI-102 / AI-103** | Securing Azure AI services with managed identity + RBAC, key-less auth to AOAI/AI Foundry, isolating tenants for multi-tenant AI apps. |
| **AI-300** | Identity for agent workloads, OBO into Microsoft Graph, governance/audit of GenAI access. |

## Where to manage what

| Surface | Use for |
|---------|---------|
| **entra.microsoft.com** | Most identity admin tasks (the modern home). |
| **portal.azure.com** | Subscriptions, resource RBAC (Azure roles), some legacy AAD blades. |
| **admin.microsoft.com** | M365 license assignment, mailbox-level stuff. |
| **Microsoft Graph** (`graph.microsoft.com`) | The single API for everything Entra (use `Microsoft.Graph` SDK / `Microsoft Graph PowerShell`). |
| **Azure CLI / Bicep** | Scripted IAM (Azure roles, MIs, app registrations via `az ad`). |

## Exam traps

- The **rename to Entra ID is cosmetic** — tenant IDs, app IDs, endpoints (`login.microsoftonline.com`) are unchanged. Old "Azure AD" answer keys still apply.
- **Conditional Access requires P1+** — Security Defaults is the Free-tier substitute and is **all-or-nothing** (you can't keep CA + Security Defaults at the same time).
- **PIM requires P2** — both for the admin who configures it *and* for every user with eligible/active assignments.
- **Identity Governance features need a P1/P2 base** — Governance is an add-on, not standalone.
- **Azure AD B2C is not the same as Entra External ID** — they are separate products. New CIAM projects must use External ID.
- **MSOnline / AzureAD PowerShell are dead** — any answer that uses `Connect-MsolService` or `Connect-AzureAD` is wrong on a 2026 exam.

## Sources

- [Microsoft Entra ID overview](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)
- [Azure AD is now Microsoft Entra ID (announcement)](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/azure-ad-is-becoming-microsoft-entra-id/3829057)
- [Entra licensing fundamentals](https://learn.microsoft.com/en-us/entra/fundamentals/licensing)
- [Entra plans and pricing](https://www.microsoft.com/security/business/microsoft-entra-pricing)
- [Entra External ID FAQ](https://learn.microsoft.com/en-us/entra/external-id/customers/faq-customers)
- [MSOnline / AzureAD PowerShell retirement](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/action-required-msonline-and-azuread-powershell-retirement---2025-info-and-resou/4364991)
