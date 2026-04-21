# Privileged Identity Management (PIM)

> **Exam mapping:** AZ-305 *(Design governance)* · AZ-500 (adjacent) · AZ-204 (rare)
> **One-liner:** Just-in-time, time-boxed, approval-gated activation of privileged roles — turns "permanent admin" into "eligible, activate when you need it".
> **Related:** [RBAC](05-rbac-and-authorization.md) · [identity governance](08-identity-governance.md) · [conditional access](06-conditional-access-and-mfa.md)

## Why PIM exists

Standing privilege = standing risk. PIM enforces **eligibility** (you *could* be admin) separated from **activation** (you *are* admin right now, for the next N hours). Combined with approval, MFA, justification, and ticket linking, it slashes the blast radius of credential theft.

## Three flavors

| PIM scope | What it manages | Where you configure it |
|-----------|-----------------|------------------------|
| **PIM for Microsoft Entra roles** | Directory roles (Global Admin, User Admin, etc.) | Entra admin center → PIM → Entra roles |
| **PIM for Azure resources** | Azure RBAC at MG/sub/RG/resource scopes | Entra admin center → PIM → Azure resources |
| **PIM for Groups** | Role-assignable security/M365 groups (membership *or* ownership) | Entra admin center → PIM → Groups |

## Licensing

- **Entra ID P2** (or **Entra ID Governance**) is required for **every user with eligible or time-bound assignments**. The admin who configures PIM also needs P2.
- Without P2, you can only do permanent active assignments.
- License check happens at activation; if the user's P2 lapses, eligibility removal kicks in.

## Assignment types

| Type | Behavior |
|------|----------|
| **Eligible (permanent)** | User can activate any time; eligibility never expires. |
| **Eligible (time-bound)** | Eligibility expires on a date. |
| **Active (permanent)** | Standing admin — what PIM exists to *replace*. Use only for break-glass. |
| **Active (time-bound)** | Direct active assignment with a hard expiry — useful for short-term contractors. |

After activation, the user holds an **active** assignment for the activation window (default max 8h, configurable up to 24h for Entra roles).

## Activation flow

```
User → "Activate role" in PIM
  → Required: justification, optional ticket #, MFA challenge if configured
  → If approval required: pending → approver(s) approve in Entra/Teams
  → Active for [requested duration ≤ max]
  → Audit log entry; expiry triggers automatic deactivation
```

Activation **settings per role** include:

- Activation maximum duration (1–24 hours).
- Require **justification**, **ticket** (and which ticketing system), **MFA** (or **authentication context** — preferred, lets CA choose the strength), **approval** (specific approvers).
- On activation: send notifications to assigned admins / requestors / approvers.
- On assignment: notify when someone is made eligible.

## Access reviews for privileged roles

PIM bundles **recurring access reviews**:

- Reviewer: self, manager, group owner, or named users.
- Cadence: weekly / monthly / quarterly / annual / one-time.
- Outcome auto-apply: remove if reviewer says "deny", or "no decision = deny".

Pair with **scheduled discovery** to find privileged roles assigned outside PIM.

## Workflow with Conditional Access

The modern pattern: instead of "Require MFA on activation," PIM points to a **Conditional Access authentication context** (e.g. `c1` = "Privileged role activation") and CA decides the strength (phishing-resistant MFA, compliant device, etc.). Lets you centralize policy.

## Just-in-time for Azure resources

Eligible at any scope (MG, sub, RG, resource). Common pattern:

- **Permanent eligible:** Subscription Owner role for the SRE team.
- **Approval required** by the platform team lead.
- **Max activation:** 4 hours, extendable.
- **MFA + justification + ticket from ServiceNow.**

```bash
# List eligible Azure RBAC role schedules (PIM)
az rest --method GET --uri "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.Authorization/roleEligibilityScheduleInstances?api-version=2020-10-01"

# Activate (create a roleAssignmentScheduleRequest)
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.Authorization/roleAssignmentScheduleRequests/$(uuidgen)?api-version=2020-10-01" \
  --body @activate.json
```

## PIM for Groups (the powerful one)

Lets you make a **role-assignable group** the unit of privilege:

1. Create a role-assignable SG (`isAssignableToRole = true`).
2. Assign it Azure RBAC roles or Entra directory roles.
3. Onboard the group to PIM.
4. Add users as **eligible members** of the group.
5. Activation grants temporary group membership → temporary role inheritance.

This is how you do JIT for **non-built-in roles**, **collections of roles**, or **owner-of-many-things** patterns. Also useful for delegating PIM administration to a team without giving them tenant-level PIM admin.

## Reporting & alerts

| Built-in alert | Trigger |
|----------------|---------|
| **Too many global administrators** | Above a threshold |
| **Roles assigned outside PIM** | Direct active assignment bypassing PIM |
| **Roles activated too frequently** | Same user over a window |
| **Administrators aren't using their privileged roles** | Stale eligibility |
| **Invalid license assignments** | Eligible user without P2 |

PIM audit history is queryable via Microsoft Graph (`/roleManagement/directory/roleAssignmentScheduleRequests`).

## Best-practice baseline

- **Zero permanent active Global Admins** except 2 break-glass accounts.
- All other admins **eligible**, with **approval + MFA + justification**.
- **Quarterly access reviews** on every privileged role.
- Grant via **role-assignable groups** (PIM for Groups) where possible — easier to audit.
- Use **authentication context** in CA for the MFA bar, not the per-role checkbox.

## Exam traps

- **PIM requires P2 (or Entra ID Governance) for every benefiting user**, not just the admin configuring it.
- **PIM eligibility is not the same as activation** — eligibility alone grants no permission.
- **Time-bound active assignments still grant standing privilege** for their window — prefer eligible.
- **Role-assignable groups must be created with `isAssignableToRole = true`** — can't flip later.
- **PIM for Azure resources is at any scope** including individual resources, not only subscription/RG.
- **Approval blocks activation** — design "skeleton" coverage to avoid 2 a.m. on-call deadlock.
- **Removing the P2 license removes eligible assignments** — license expiry can silently lock people out.

## Sources

- [What is PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [PIM deployment plan](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-deployment-plan)
- [Activate Entra roles](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-activate-role)
- [PIM for Groups](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/groups-features)
- [PIM licensing](https://learn.microsoft.com/en-us/entra/id-governance/licensing-fundamentals)
- [Authentication context with PIM](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-cloud-apps#authentication-context)
