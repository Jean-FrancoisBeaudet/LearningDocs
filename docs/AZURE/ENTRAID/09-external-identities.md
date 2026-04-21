# External Identities

> **Exam mapping:** AZ-305 *(Design identity for external users)* · AI-102 *(multi-tenant AI apps)*
> **One-liner:** Three distinct paths: **B2B collaboration** (work-with partners), **B2B direct connect** (Teams shared channels), and **Entra External ID** (your customer-facing apps — replaces Azure AD B2C).
> **Related:** [tenants/users](01-tenants-users-groups.md) · [governance](08-identity-governance.md) · [conditional access](06-conditional-access-and-mfa.md)

## The three external-identity models

| Model | Use case | Where users live | Pricing |
|-------|----------|------------------|---------|
| **B2B collaboration** | Invite partner users into your workforce tenant as **guests** | Home tenant authenticates; stub object in resource tenant | First **50 000 MAU/tenant free**, then per-MAU |
| **B2B direct connect** | **Teams shared channels** with another Entra org (no guest object) | Both tenants; trust is configured in cross-tenant access settings | Currently no extra cost |
| **Entra External ID (CIAM)** | Your **customer-facing app** sign-up/sign-in | Dedicated **External tenant** | First 50 000 MAU free per tenant, then per-MAU |

## B2B collaboration — guests

- Invite by email; user redeems via `myaccount.microsoft.com` or app first-sign-in.
- `userType = Guest`, `creationType = Invitation`.
- Can be made an **external member** instead (`userType = Member`, external) for multi-tenant orgs where the partner is effectively staff.
- Authentication happens at the **home tenant**; resource tenant CA still applies and can require an MFA claim from home (cross-tenant trust).

### Cross-tenant access settings (the security knob)

For each external Entra tenant you can configure **inbound** and **outbound** rules:

| Setting | Inbound (guests coming into your tenant) | Outbound (your users going to theirs) |
|---------|-----------------------------------------|---------------------------------------|
| **B2B collaboration** | Allow / block users, groups, apps | Same |
| **B2B direct connect** | Allow Teams shared channels | Same |
| **Trust settings** | Trust home tenant's MFA claim, compliant device claim, hybrid-joined device claim | n/a |
| **Tenant restrictions v2** | Block sign-in to other tenants' apps from your network/devices | n/a |

> Trusting MFA from the home tenant means your CA "require MFA" is satisfied without re-prompting the guest — improves UX.

### Identity providers for B2B guests

Default: invitee redeems with their Entra account. Additional supported IdPs:

- **Microsoft account (MSA)** — personal Outlook/Live accounts.
- **One-Time Passcode (email OTP)** — for users with no MS-managed identity (partners with random email domains). Falls back if no other IdP.
- **Google federation** — for `gmail.com` guests.
- **SAML/WS-Fed federation** with a partner IdP (per-domain).

## B2B direct connect

- For **Teams shared channels** *only* (currently).
- No guest object created in your tenant — both sides use **mutual** trust based on cross-tenant access settings.
- Users from Org B can be added to a shared channel hosted in Org A's Team without ever appearing in Org A's directory as a user.
- Requires **outbound enabled** in user's home org and **inbound enabled** in resource org for B2B direct connect.

## Entra External ID (the CIAM successor)

Microsoft's **next-gen customer identity** product. Built on top of the Entra ID platform (not the old B2C engine), unifies CIAM with Entra's modern features (CA, Identity Protection, ML risk, Verified ID).

### Status vs Azure AD B2C

| | **Azure AD B2C** (legacy) | **Entra External ID** |
|---|--------------------------|----------------------|
| New tenants | **Not available after May 1, 2025** | Yes — via Entra admin center |
| Support | Until at least **2030** for existing customers | Active development |
| Engine | B2C-specific stack with custom policies (XML "Identity Experience Framework") | Entra ID stack with **user flows** (and limited custom extensions) |
| Identity providers | Many social + SAML/OIDC + custom | Email, Google, Facebook, Apple (preview), SAML/OIDC federation, custom |
| Custom policies (IEF) | Yes — extremely powerful, complex | Not yet at parity; **custom authentication extensions** + token augmentation cover common cases |
| Conditional Access | Limited (B2C had its own custom policies for risk) | Full Entra CA + Identity Protection |
| Pricing | MAU-based | MAU-based — first 50 k MAU free per External tenant |

> **Exam answer rule:** for any *new* customer-facing app, the answer is **Entra External ID**, not Azure AD B2C.

### External tenant setup (high level)

1. From Entra admin center → Identity → Manage tenants → **Create** → choose **External**.
2. Configure user flow: pick attributes to collect at sign-up, identity providers, branding (logo, colors, custom domain).
3. Register your app in the external tenant; integrate with MSAL using the tenant's authority URL: `https://<tenant>.ciamlogin.com/<tenant>.onmicrosoft.com`.
4. Optionally add **custom authentication extensions** (Azure Function/REST callout) at sign-up or token-issuance time for custom claims, account linking, or external profile lookup.

### App integration sample (MSAL.js for SPA)

```js
const msalConfig = {
  auth: {
    clientId: "<app-id>",
    authority: "https://contoso.ciamlogin.com/contoso.onmicrosoft.com",
    knownAuthorities: ["contoso.ciamlogin.com"],
    redirectUri: "https://app.contoso.com/redirect"
  }
};
```

### Custom authentication extensions

REST endpoints (commonly Azure Functions) that Entra calls during sign-up or before token issuance to:

- Augment the token with claims from your CRM/database.
- Perform custom validation (e.g. block disposable email domains).
- Trigger account linking with an existing internal user record.

Secured with HTTPS + access token verification on the function side. Replaces a major chunk of what B2C IEF custom policies were used for.

## Comparison cheat-sheet (pick the right model)

| Scenario | Use |
|----------|-----|
| Partner users access your SharePoint and Teams | **B2B collaboration** (guests) |
| Joint working group in Teams shared channel with another company | **B2B direct connect** |
| Your subsidiaries are separate Entra tenants but share staff | **Cross-tenant sync (XTS)** + B2B direct connect |
| Build a consumer mobile app with email/Google sign-in | **Entra External ID** |
| Existing Azure AD B2C app with rich custom policies | **Stay on B2C** for now; plan migration to External ID as parity grows |
| Need MFA claim trust to skip re-prompting partner users | Configure **trust settings** in cross-tenant access |

## Limits / quotas

| Thing | Value |
|-------|-------|
| Free B2B MAU per workforce tenant | **50 000** |
| Free MAU per External (CIAM) tenant | **50 000** |
| External tenants per directory | Subject to subscription limits; typically dozens |
| B2B email OTP code lifetime | 30 minutes |

## Exam traps

- **Azure AD B2C is closed to new customers (May 1, 2025).** Choose **Entra External ID** for new CIAM scenarios.
- **B2C is *not* renamed to External ID** — they are separate products with different engines.
- **B2B direct connect is Teams-only today**; outside Teams, use B2B collaboration.
- **Cross-tenant trust settings can satisfy CA's MFA requirement from the home tenant** — the "require MFA" grant doesn't always re-prompt.
- **External members ≠ guests.** XTS creates external members for multi-tenant orgs.
- **B2C custom policies (IEF) do not exist in External ID** — use **custom authentication extensions** instead.
- **Pricing is MAU**, not per-seat — a user counts once per month regardless of sign-in count.
- **Tenant restrictions v2** lets you block your users from signing into *other* tenants' SaaS using corporate identities — useful for data exfil prevention.

## Sources

- [B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)
- [Cross-tenant access settings](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-overview)
- [B2B direct connect](https://learn.microsoft.com/en-us/entra/external-id/b2b-direct-connect-overview)
- [Entra External ID overview](https://learn.microsoft.com/en-us/entra/external-id/customers/overview-customers-ciam)
- [External ID FAQ (B2C status)](https://learn.microsoft.com/en-us/entra/external-id/customers/faq-customers)
- [Custom authentication extensions](https://learn.microsoft.com/en-us/entra/identity-platform/custom-extension-overview)
- [Tenant restrictions v2](https://learn.microsoft.com/en-us/entra/external-id/tenant-restrictions-v2)
