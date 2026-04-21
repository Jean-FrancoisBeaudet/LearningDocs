# Identity Governance

> **Exam mapping:** AZ-305 *(Design governance solutions)*
> **One-liner:** The "who should have what, for how long, and who said yes" layer — entitlement management, access reviews, lifecycle workflows, ToU.
> **Related:** [PIM](07-privileged-identity-management.md) · [external identities](09-external-identities.md) · [overview SKUs](00-overview.md)

## What's in Entra ID Governance

| Feature | Tier required |
|---------|---------------|
| **Entitlement management** (access packages) | **Entra ID Governance** add-on (or Entra Suite) |
| **Access reviews** (broad — any group/role/access package) | Governance |
| **Access reviews** (Entra roles + Azure roles via PIM) | P2 (limited scope) |
| **Lifecycle workflows** (joiner/mover/leaver) | Governance |
| **Separation of duties checks** | Governance |
| **Terms of use** | P1 |
| **Cross-tenant synchronization** | P1 (with Governance for advanced governance) |
| **Machine-learning recommendations** in reviews | Governance |

> Governance is an **add-on** to P1 or P2, not a standalone SKU. Every user covered by a governance feature needs the license.

## Entitlement management

### Building blocks

| Concept | What it is |
|---------|-----------|
| **Catalog** | A container that bundles related resources (groups, apps, SharePoint sites, Teams). Owned by **catalog owners** who delegate. |
| **Resource** | A group, M365 group, app, or SharePoint Online site added to a catalog. |
| **Access package** | What the user requests. Bundles selected resources with one or more **policies**. |
| **Policy** | "Who can request, who approves, expiration, access reviews on this package." Multiple policies per package (e.g. employee policy vs guest policy). |
| **Assignment** | A user's active grant of an access package (created on approval). |

### Typical flow

```
User → MyAccess portal → request "DataLake-Analyst" access package
  → Approver (manager / specific reviewer / sponsor) approves
  → User auto-added to all bundled groups, apps, sites
  → Access expires per policy; auto-removed
  → Optional periodic access review during the lifetime
```

### What it solves on the exam

- **Onboarding partners** (B2B) without manual group assignment per resource.
- **Self-service request portal** for internal apps.
- **Time-bound access** with automatic clean-up.
- **Recurring reviews** built in.

### Connected organizations

For B2B scenarios, define **connected organizations** (other Entra tenants, domains, or "all other"). Policies can scope "users from these orgs may request this package." External users get auto-invited as guests on first request.

## Access reviews

| Target | Reviewers | Outcome auto-apply |
|--------|-----------|--------------------|
| Group / Team membership | Group owners / self / specific users / managers | Remove on deny |
| App assignments | App owners | Remove on deny |
| Entra role (PIM) | Self / specific users | Remove eligibility on deny |
| Azure RBAC role (PIM) | Self / specific users | Remove eligibility on deny |
| Access package assignments | Approvers / managers | End assignment on deny |
| Inactive users | n/a (system auto-flags) | Disable account on no decision |

Settings: cadence, duration, no-decision behavior (approve / deny / take recommendation), email notifications, justification required, ML-based recommendations ("user hasn't signed in for 90 days → recommend remove").

## Lifecycle workflows

Automate **joiner/mover/leaver** events. Triggered by:

| Trigger | Example |
|---------|---------|
| **Joiner** — *N days before* `employeeHireDate` | Send welcome email, generate TAP, add to default groups |
| **Mover** — attribute change (department, manager, jobTitle) | Re-assign groups, update license, notify new manager |
| **Leaver** — *N days after* `employeeLeaveDateTime` | Remove licenses, disable account, revoke sessions, transfer manager, delete after retention |

Built-in tasks (50+): add/remove from groups, add/remove M365 license, generate TAP, send email, disable account, remove from all groups, revoke sessions, request access package assignment, run a custom HTTP task (Logic App callout).

```jsonc
// Workflow definition fragment (Graph)
{
  "category": "joiner",
  "displayName": "Pre-hire onboarding",
  "executionConditions": {
    "@odata.type": "#microsoft.graph.identityGovernance.triggerAndScopeBasedConditions",
    "scope": { "rule": "(department eq 'Sales')" },
    "trigger": {
      "@odata.type": "#microsoft.graph.identityGovernance.timeBasedAttributeTrigger",
      "timeBasedAttribute": "employeeHireDate",
      "offsetInDays": -7
    }
  },
  "tasks": [
    { "taskDefinitionId": "70b29d51-b59a-4773-9280-8841dfd3f2ea",
      "displayName": "Generate Temporary Access Pass",
      "arguments": [{ "name": "tapLifetimeMinutes", "value": "240" }] }
  ]
}
```

> Lifecycle workflows now support **reprocessing** failed/previous runs (2025–2026 enhancement).

## Terms of use

A PDF that users must accept before accessing covered apps. Configured as a **CA grant control** ("Require Terms of Use"). Tracks acceptance per version, can require periodic re-acceptance, and can be limited per device/browser.

Common scenarios: GDPR consent, contractor NDA, BYOD acceptable-use, partner data-handling.

## Cross-tenant synchronization (XTS)

Synchronizes **users between Entra tenants** in the *same multi-tenant organization* (M&A, conglomerate, partner-with-shared-staff). Uses the same provisioning engine as SaaS provisioning.

| Feature | Notes |
|---------|-------|
| Source/target | Two Entra tenants in a *Multi-tenant organization* (MTO) |
| Object created | **External member** (not guest) — `userType=Member`, but external |
| Drives | SCIM-style attribute mapping; supports filters, soft-match by `mail`/`employeeId` |
| Lifecycle | Source change → target update; source disable → target disable |
| Combined with | Lifecycle workflows can run on the synced external members |

Set up: **Multi-tenant organization** in source tenant → invite target → configure outbound XTS → target accepts and configures inbound. Cross-tenant access settings must allow B2B + the inbound tenant.

## Separation of duties (SoD)

In an access package policy, declare incompatible packages: "User assigned to AP-Finance-Reader cannot request AP-Finance-Approver." Enforced at request time. Use to mirror SoX/SOX, GDPR, FedRAMP segregation requirements.

## Recommended baseline (architect answer)

- **Catalog per business domain** with delegated catalog owners.
- **One access package per persona** (data analyst, app admin, vendor reviewer).
- **Policies** split by audience (employees vs partners) with different approval chains and lifetimes.
- **Quarterly access reviews** on all packages + all PIM eligible assignments.
- **Lifecycle workflows** for joiner/mover/leaver tied to HRIS-driven `employeeHireDate` / `employeeLeaveDateTime`.
- **Terms of use** for partner tenants.
- **XTS** within the MTO; B2B for everyone else.

## Exam traps

- **Entitlement management ≠ B2B invitation.** EM *uses* B2B but adds catalog/policy/approval.
- **An access package can include B2B guests** (via "users from connected orgs"); inviting unmanaged becomes a one-click request.
- **Access reviews on Azure RBAC require PIM** (P2) — without PIM you can't review resource roles.
- **Lifecycle workflows require Governance**, not just P2.
- **XTS creates external members**, not guests — different `userType` semantics, different licensing implications.
- **SoD checks are at request time only** — they don't auto-revoke if you flip the rule later.
- **Connected organizations and cross-tenant access settings are different things** — one is for EM scoping, the other is for B2B/B2B-DC trust at the tenant level.

## Sources

- [Entra ID Governance overview](https://learn.microsoft.com/en-us/entra/id-governance/identity-governance-overview)
- [Entitlement management](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-overview)
- [Lifecycle workflows](https://learn.microsoft.com/en-us/entra/id-governance/what-are-lifecycle-workflows)
- [Access reviews](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview)
- [Cross-tenant synchronization](https://learn.microsoft.com/en-us/entra/identity/multi-tenant-organizations/cross-tenant-synchronization-overview)
- [Terms of use](https://learn.microsoft.com/en-us/entra/identity/conditional-access/terms-of-use)
- [Governance licensing](https://learn.microsoft.com/en-us/entra/id-governance/licensing-fundamentals)
