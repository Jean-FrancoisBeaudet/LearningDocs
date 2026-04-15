# Tenants, Users, and Groups

> **Exam mapping:** AZ-204 · AZ-305 *(Design identity solutions)*
> **One-liner:** The directory primitives — tenants, domains, user types, group types — and their gotchas.
> **Related:** [overview](00-overview.md) · [external identities](09-external-identities.md) · [RBAC](05-rbac-and-authorization.md)

## Tenant types

| Type | Purpose | Notes |
|------|---------|-------|
| **Workforce** | Employees + B2B guests | The "normal" tenant. One per organization. |
| **External** (Entra External ID) | Customer-facing (CIAM) | Separate tenant flavor created in Entra admin center. |
| **Azure AD B2C** *(legacy)* | Legacy CIAM | No new tenants after **May 1, 2025**. Existing tenants supported through **2030**. |

A tenant is identified by a **tenant ID (GUID)** and one or more domains. The **home tenant** of a subscription owns RBAC for that subscription's resources.

## Domains

| Domain kind | Example | Properties |
|-------------|---------|------------|
| **Initial (default)** | `contoso.onmicrosoft.com` | Auto-created, cannot be deleted, cannot be made unverified. |
| **Custom (managed)** | `contoso.com` | Add via Entra admin center → Domain names → verify with DNS TXT/MX. |
| **Federated** | `contoso.com` (federated) | Auth delegated to ADFS / 3rd-party IdP via WS-Fed/SAML. Rare in greenfield — prefer cloud auth + PHS/PTA. |

A user's **UPN** (`alice@contoso.com`) is independent of their email — although they are usually the same, sync rules can diverge them.

## User types

| Type | `userType` | Source | Typical use |
|------|------------|--------|-------------|
| **Member** | `Member` | Cloud-created or synced from on-prem AD | Employees |
| **Guest** | `Guest` | Invited via B2B from another Entra tenant or social IdP | Partners, contractors |
| **External member** | `Member` (but external) | Invited via cross-tenant access settings as a *member* | Multi-tenant orgs (M&A, conglomerates) |
| **Cloud-only** | n/a | Created directly in Entra | Service accounts, break-glass admins |
| **Synced** | n/a | Sourced from on-prem AD via Entra Connect Sync / Cloud Sync | Standard hybrid users |

`creationType` distinguishes how the user object was made (e.g. `Invitation` for guests). Guests authenticate against their *home* tenant; the resource tenant only stores a stub object.

### Break-glass accounts (exam favorite)

- **At least two** cloud-only Global Administrators that are **excluded from all Conditional Access policies** and use **FIDO2 keys** (or a long, vaulted password + phishing-resistant MFA).
- Stored credentials in a physical safe; monitored for sign-ins via dedicated alerts.
- Excluded from MFA registration enforcement so they always sign in.

## Group types

| Group | Membership | Can assign Microsoft 365 licenses? | Can be mail-enabled? | Can be assigned to Entra roles? |
|-------|-----------|----------------------------------|----------------------|--------------------------------|
| **Security** | Users, devices, groups, SPs | Yes | No (unless mail-enabled SG, legacy) | **Yes**, if `isAssignableToRole = true` (set at creation, immutable) |
| **Microsoft 365** | Users only | Yes | Yes (always) | **Yes**, if role-assignable |
| **Distribution / mail-enabled SG** | Users | No | Yes | No |

### Membership type

- **Assigned** — manual add/remove.
- **Dynamic User** — rule like `user.department -eq "Sales"`. Requires **Entra ID P1**.
- **Dynamic Device** — same idea, for devices.

> A group can be either Assigned **or** Dynamic — not both. Dynamic membership rules are evaluated by a background processor; latency from rule change to membership update is typically minutes.

### Role-assignable groups

- Must be created with `isAssignableToRole = true` (cannot be flipped later).
- Only **Privileged Role Administrator** or **Global Admin** can manage members → prevents privilege-escalation via group hijacking.
- Cannot be a dynamic group (membership must be deterministic).

## Quick CLI / Graph examples

```bash
# Create a security group via Graph PowerShell
Connect-MgGraph -Scopes "Group.ReadWrite.All"
New-MgGroup -DisplayName "sg-storage-readers" `
            -MailEnabled:$false -MailNickname "sg-storage-readers" `
            -SecurityEnabled:$true

# Create a role-assignable group
New-MgGroup -DisplayName "rg-helpdesk-admins" -MailEnabled:$false `
            -MailNickname "rg-helpdesk-admins" -SecurityEnabled:$true `
            -IsAssignableToRole:$true

# Invite a B2B guest
New-MgInvitation -InvitedUserEmailAddress "ext@partner.com" `
                 -InviteRedirectUrl "https://myapps.microsoft.com" `
                 -SendInvitationMessage:$true
```

```bash
# Azure CLI: dynamic group rule
az ad group create \
  --display-name "dg-sales-users" \
  --mail-nickname "dg-sales-users" \
  --description "All Sales department users" \
  --membership-rule '(user.department -eq "Sales")' \
  --membership-rule-processing-state "On" \
  --group-types "DynamicMembership"
```

## Limits worth memorizing

| Thing | Limit |
|-------|-------|
| Objects per tenant (Free) | 500 000 |
| Objects per tenant (any paid SKU) | Unlimited (soft) |
| Groups a user can be member of | 1 000 (security & M365 combined) |
| Members per group | ~50 000 for Teams-backed M365 groups; >1M for plain SGs |
| Owners per group | 100 |
| Custom domains per tenant | 5 000 |
| Dynamic membership rule length | 3 072 chars |

## Exam traps

- **`isAssignableToRole` is set-once.** A regular SG cannot be retrofitted to receive role assignments — recreate.
- **Dynamic groups need P1**, even for devices.
- **Guests count toward MAU**, not seat licenses (B2B free up to 50 000 MAU per tenant).
- **B2B guest sign-ins authenticate at the home tenant** — Conditional Access in the *resource* tenant still applies (cross-tenant access settings can require MFA claim from home).
- **Synced users cannot be edited cloud-side for source-of-authority attributes** — change them in on-prem AD.
- The **initial `.onmicrosoft.com` domain is permanent**, even after you add custom domains.

## Sources

- [User type & properties](https://learn.microsoft.com/en-us/entra/external-id/user-properties)
- [Group types overview](https://learn.microsoft.com/en-us/entra/fundamentals/concept-learn-about-groups)
- [Dynamic membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
- [Role-assignable groups](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/groups-concept)
- [Service limits](https://learn.microsoft.com/en-us/entra/identity/users/directory-service-limits-restrictions)
