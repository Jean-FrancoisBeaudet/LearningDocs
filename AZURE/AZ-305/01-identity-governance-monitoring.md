# AZ-305 — Design Identity, Governance, and Monitoring Solutions (25–30%)

> Exam domain weight: **25–30%** of [AZ-305](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/). English version refresh: **April 17, 2026**. Passing score 700.
> Official [AZ-305 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305) · [Learning path: Design identity, governance, monitor](https://learn.microsoft.com/en-us/training/paths/design-identity-governance-monitor-solutions/)
> Scope: Entra ID topology, hybrid identity, authN/authZ design, governance ([Policy](https://learn.microsoft.com/en-us/azure/governance/policy/overview) + [Deployment Stacks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks)), cost governance, and the [Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/overview) stack (Log Analytics, App Insights, diagnostics, alerts, Defender for Cloud).

---

## 1. Tenant / management-group / subscription topology

Design order, top-down: **[Tenant](https://learn.microsoft.com/en-us/entra/fundamentals/whatis) root → [Management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview) → Subscriptions → Resource groups → Resources**. RBAC, Policy, and cost constructs inherit down this hierarchy.

### [Enterprise-scale landing zone (CAF)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) default shape

```
Tenant Root
└── Top-Level MG ("Contoso")
    ├── Platform
    │   ├── Identity       (domain controllers, Entra Connect)
    │   ├── Management     (Log Analytics, automation)
    │   └── Connectivity   (hub vNet, Firewall, ER/VPN)
    ├── Landing Zones
    │   ├── Corp           (private, peered)
    │   └── Online         (internet-facing)
    ├── Decommissioned
    └── Sandbox
```

Limits to remember ([MG limits](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview#hierarchy-of-management-groups-and-subscriptions)):
- **6 MG levels** (excluding tenant root and subscription level).
- **1 parent** per MG; any MG can hold subscriptions *or* MGs.
- Policy/RBAC assigned at MG level applies to **every** descendant subscription — use for platform-wide guardrails, not workload-specific rules.

### Tenant design choices

| Scenario | Recommendation |
|---|---|
| M&A / regulatory isolation / sovereign data | Multiple tenants, use [**Entra External ID**](https://learn.microsoft.com/en-us/entra/external-id/external-identities-overview) / B2B for cross-tenant access |
| Dev/test separation with same identity plane | **Single tenant**, separate MGs + subscriptions |
| Customer-facing SaaS | [**Entra External ID for customers**](https://learn.microsoft.com/en-us/entra/external-id/customers/overview-customers-ciam) (formerly Azure AD B2C) — separate from workforce tenant |
| [Azure Arc](https://learn.microsoft.com/en-us/azure/azure-arc/overview) / on-prem / multi-cloud | Single tenant; onboard resources to the **Platform/Management** MG |

> **Exam framing:** if the question mentions "isolate billing + identity + compliance boundary," that's a **new tenant**. If it's just "separate environments" it's **new subscription / MG**.

---

## 2. Hybrid identity — PHS / PTA / Federation / Cloud Sync

As of 2026, [**Microsoft Entra Cloud Sync**](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync) is Microsoft's strategic direction; [**Entra Connect Sync**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect) (on-prem sync engine) must be upgraded to [**≥ 2.5.79.0 by 30 Sep 2026**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history) or sync breaks. [Federation (AD FS)](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-fed) is de-emphasized — recommend only when you truly need it.

### Authentication method decision matrix

[Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)

| Method | Where auth happens | Pros | Cons | Pick when |
|---|---|---|---|---|
| [**Password Hash Sync (PHS)**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-phs) | Cloud | Simplest, cheapest, supports [leaked-credential detection](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks), survives on-prem outage | Password hashes (of hashes) stored in Entra ID | Default choice; security-team must accept hash sync |
| [**Pass-Through Auth (PTA)**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-pta) | On-prem AD via agents | Passwords stay on-prem, enforces on-prem password policy | Requires HA agents, on-prem outage breaks cloud sign-in | Regulatory mandate that passwords cannot leave on-prem |
| [**Federation (AD FS)**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-fed) | On-prem AD FS | Smart-card / 3rd-party MFA / complex claims | Heavy infra, brittle, largest attack surface | Only when PHS/PTA can't meet a specific requirement (e.g., smart-card-only login) |
| [**Seamless SSO**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sso) | Add-on | Silent sign-in on domain-joined devices | Add-on only (combine with PHS or PTA) | Always enable with PHS/PTA for user experience |

### [Entra Connect Sync vs Entra Cloud Sync](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync#comparison-between-microsoft-entra-connect-and-cloud-sync)

| Capability | Connect Sync | Cloud Sync |
|---|---|---|
| Installer | Heavy on-prem server + SQL | Lightweight agent(s) |
| [Multiple disconnected forests](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/plan-cloud-sync-topologies) | No (needs multi-forest trust) | **Yes** (multiple agents, multiple forests) |
| HA | Staging-mode server | **Active-active agents** |
| Sync frequency | Every 30 min | **Every 2 min** |
| [Group writeback](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-group-writeback-v2), device writeback, Exchange hybrid | ✅ | ⚠️ partial — check current gap list before committing |
| Directory extension attributes, custom sync rules | ✅ | Limited |
| Future direction | Legacy / upgrade mandatory by Sep 2026 | **Strategic** |

**Decision rule:** Default to **Cloud Sync** for greenfield or simple topologies and multi-forest. Stay on **Connect Sync** when you rely on its advanced features (complex sync rules, device writeback, Exchange hybrid group writeback) — but plan migration. You *can* [run both in parallel](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/plan-cloud-sync-topologies#existing-hybrid-customer-with-microsoft-entra-connect-sync) for disjoint OU scopes during migration.

---

## 3. Conditional Access + MFA + PIM

### [Conditional Access (CA)](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview) design principles

- Always design CA around [**signals → conditions → controls**](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policies).
- Require [MFA for all admins](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-mfa-strength) (or better: require [**Entra PIM**](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)-elevated session + [phishing-resistant MFA](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)).
- [Block legacy auth](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-legacy-authentication) universally.
- Require [**compliant device**](https://learn.microsoft.com/en-us/mem/intune/protect/device-compliance-get-started) (Intune) or [**Hybrid Entra Join**](https://learn.microsoft.com/en-us/entra/identity/devices/concept-hybrid-join) for privileged roles + sensitive apps.
- Use [**named locations**](https://learn.microsoft.com/en-us/entra/identity/conditional-access/location-condition) and [**sign-in risk / user risk**](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks) ([Identity Protection](https://learn.microsoft.com/en-us/entra/id-protection/overview-identity-protection), P2) instead of static IP allow-lists.
- Pilot with [**Report-only**](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only) mode; exclude a [break-glass account](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) from every policy.

[Baseline policy set](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policy-common) to memorize:

| Policy | Users | Apps | Condition | Control |
|---|---|---|---|---|
| Block legacy auth | All | All | Client apps: Exchange ActiveSync + other clients | Block |
| MFA for admins | Directory roles | All | — | Require MFA (phishing-resistant) |
| Risky sign-ins | All | All | Sign-in risk: High | Block / Password change |
| Device compliance for sensitive | All | O365, Azure mgmt | — | Require compliant device |
| Session controls | All | O365 | Unmanaged device | App-enforced restrictions / sign-in freq |

### [MFA](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-mfa-howitworks)

- [**Phishing-resistant MFA**](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths#built-in-authentication-strengths) ([FIDO2](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-passwordless#fido2-security-keys), [Windows Hello](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-passwordless-deployment), [certificate-based](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-certificate-based-authentication)) is the target state for admins and high-value apps.
- Enforce MFA via [**CA**](https://learn.microsoft.com/en-us/entra/identity/conditional-access/howto-conditional-access-policy-all-users-mfa), not per-user MFA (legacy) and not via [Security Defaults](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults) when CA is in use.
- [**Authentication strengths**](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths) (built-in or custom) let CA require specific methods per app.

### [Privileged Identity Management (PIM)](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure) — Entra ID P2

- [**Eligible** vs **active**](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user) assignments: principals are eligible, must **activate** with approval + MFA + reason for time-boxed window.
- Scope: [Entra roles](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user), [**Azure RBAC roles**](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-resource-roles-assign-roles), and [PIM for **Groups**](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/concept-pim-for-groups) (activate group membership).
- Design: 0 permanent Global Admins ([break-glass](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) x2 cloud-only accounts excluded), break-glass monitored via Log Analytics alert.
- Review cadence: [**Access reviews**](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview) quarterly for privileged roles, annually for app access.

---

## 4. RBAC vs ABAC vs Entra roles

| Layer | What it controls | Examples | Scope |
|---|---|---|---|
| [**Entra roles**](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/custom-overview) | Directory objects (users, apps, tenants) | [Global Admin, User Admin, Application Admin](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference) | Tenant / [administrative unit](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units) |
| [**Azure RBAC**](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) (roles) | Azure resources via ARM | [Owner, Contributor, Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles), [Storage Blob Data Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/storage) | MG / Subscription / RG / Resource |
| [**Azure ABAC**](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) (role *conditions*) | Narrows RBAC via attribute conditions | Restrict Blob read to blobs tagged `Project=Alpha` | RG / Resource (conditions on RBAC assignment) |

Rules of thumb:
- Prefer [**built-in**](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles) roles; [custom roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles) only when a built-in clearly over- or under-privileges.
- Use [**Administrative Units**](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units) to delegate Entra role scope (e.g., helpdesk for a region).
- Use [**ABAC**](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-format) to avoid role explosion: one role + condition beats 20 custom roles.
- [**Deny assignments**](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) are created only by [**Azure managed apps**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/overview) and [**Deployment Stacks**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks#protect-managed-resources-against-deletion) — you can't author them directly.

---

## 5. Managed identities & workload identity federation

Default: **never use client secrets or certificates you have to rotate**. See [Managed identities overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview).

| Option | Use when |
|---|---|
| [**System-assigned MI**](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview#managed-identity-types) | 1:1 lifecycle with a single Azure resource |
| [**User-assigned MI**](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities) | Shared across resources, pre-created in IaC, cross-resource identity reuse |
| [**Workload Identity Federation (WIF)**](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) | Workload runs **outside Azure** ([GitHub Actions](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp#github-actions), GitLab, [Kubernetes](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview), other clouds) — federates OIDC tokens to an Entra app or user-assigned MI, no secrets |

### Bicep — user-assigned MI + Key Vault data-plane role

Refs: [`Microsoft.ManagedIdentity/userAssignedIdentities`](https://learn.microsoft.com/en-us/azure/templates/microsoft.managedidentity/userassignedidentities) · [`Microsoft.Authorization/roleAssignments`](https://learn.microsoft.com/en-us/azure/templates/microsoft.authorization/roleassignments) · [Key Vault built-in roles](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)

```bicep
param location string = resourceGroup().location

resource uami 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'uami-app-prod'
  location: location
}

resource kv 'Microsoft.KeyVault/vaults@2024-04-01-preview' existing = {
  name: 'kv-contoso-prod'
}

// Key Vault Secrets User
resource roleAssign 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(kv.id, uami.id, '4633458b-17de-408a-b874-0445c86b69e6')
  scope: kv
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: uami.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### CLI — federate GitHub Actions to a user-assigned MI (no secrets)

Refs: [`az identity federated-credential`](https://learn.microsoft.com/en-us/cli/azure/identity/federated-credential) · [Configure GitHub Actions OIDC](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp#github-actions) · [`azure/login@v2`](https://github.com/Azure/login)

```bash
# Create federated credential on the UAMI
az identity federated-credential create \
  --name "github-main" \
  --identity-name "uami-app-prod" \
  --resource-group "rg-app-prod" \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:contoso/app:ref:refs/heads/main" \
  --audiences "api://AzureADTokenExchange"
```

In the workflow: `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` — no `client-secret`.

---

## 6. Azure Policy + initiatives + remediation + Deployment Stacks

### [Policy](https://learn.microsoft.com/en-us/azure/governance/policy/overview) mental model

- [**Definition**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure) → JSON rule (if/then with effect).
- [**Initiative (Set)**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/initiative-definition-structure) → bundle of definitions (e.g., [*Microsoft cloud security benchmark*](https://learn.microsoft.com/en-us/security/benchmark/azure/overview), [*CIS*](https://learn.microsoft.com/en-us/azure/governance/policy/samples/cis-azure-2-0-0), [*NIST 800-53*](https://learn.microsoft.com/en-us/azure/governance/policy/samples/nist-sp-800-53-r5)).
- [**Assignment**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/assignment-structure) → scoped to MG / subscription / RG.
- [**Effects**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects): `Audit`, `Deny`, `AuditIfNotExists`, [`DeployIfNotExists` (DINE)](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deploy-if-not-exists), `Modify`, `Append`, `Disabled`, `Manual`.
- [**Exemptions**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/exemption-structure) (waivers) vs [**Exclusions**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/assignment-structure#excluded-scopes) (scope carve-outs) — different tools, both auditable.

DINE and Modify require a [**managed identity on the assignment**](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources) with rights to remediate. [Remediation tasks](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources) fix *existing* non-compliant resources; effects only block *new* ones.

### Bicep — assign an initiative at an MG with remediation MI

Refs: [`Microsoft.Authorization/policyAssignments`](https://learn.microsoft.com/en-us/azure/templates/microsoft.authorization/policyassignments) · [MCSB built-in initiative](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

```bicep
targetScope = 'managementGroup'

resource assign 'Microsoft.Authorization/policyAssignments@2024-05-01' = {
  name: 'mcsb-baseline'
  location: 'eastus'
  identity: { type: 'SystemAssigned' }
  properties: {
    displayName: 'Microsoft cloud security benchmark'
    policyDefinitionId: '/providers/Microsoft.Authorization/policySetDefinitions/1f3afdf9-d0c9-4c3d-847f-89da613e70a8'
    enforcementMode: 'Default'
    nonComplianceMessages: [
      { message: 'Must meet MCSB baseline to pass audit.' }
    ]
  }
}

// Grant MI rights to remediate
resource role 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid('mcsb-baseline', 'contributor')
  properties: {
    roleDefinitionId: '/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c'
    principalId: assign.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

CLI remediation ([`az policy remediation`](https://learn.microsoft.com/en-us/cli/azure/policy/remediation)):
```bash
az policy remediation create \
  --name remediate-mcsb \
  --policy-assignment "$(az policy assignment show -n mcsb-baseline --query id -o tsv)"
```

### [Deployment Stacks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks) (GA) — the Blueprints replacement

[**Azure Blueprints is deprecated, retirement July 2026**](https://learn.microsoft.com/en-us/azure/governance/blueprints/overview) — migrate to **Deployment Stacks + Policy + Bicep modules**. [Migration guide](https://learn.microsoft.com/en-us/azure/governance/blueprints/resources/migrate-from-blueprints).

- Stack = collection of ARM/Bicep resources managed as a unit.
- [`--deny-settings-mode`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks#protect-managed-resources-against-deletion) `denyDelete|denyWriteAndDelete|none` creates a [**deny assignment**](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) protecting managed resources.
- [`--action-on-unmanage`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks#control-detachment-and-deletion) `detachAll|deleteAll|deleteResources` controls what happens to resources that leave the stack.

[`az stack sub create`](https://learn.microsoft.com/en-us/cli/azure/stack/sub):
```bash
az stack sub create \
  --name platform-landing-zone \
  --location eastus \
  --template-file main.bicep \
  --deny-settings-mode denyDelete \
  --action-on-unmanage deleteResources
```

| Concern | Policy | Deployment Stack |
|---|---|---|
| Enforce config rules across many subs | ✅ | ❌ |
| Guarantee resource can't be deleted | Indirect (deny effect on operations) | ✅ via deny assignment |
| Audit compliance posture | ✅ | ❌ |
| Deploy + lifecycle-manage a set | ❌ | ✅ |

Use them **together**: stacks for *"this set of resources exists and is protected"*, policy for *"any resource anywhere must comply"*.

---

## 7. Cost governance — budgets, reservations, savings plans

| Tool | What it does | When |
|---|---|---|
| [**Cost Management + budgets**](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets) | Threshold alerts (actual + forecast) at MG/sub/RG scope; trigger [Action Groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups) or runbooks | Always — day 1 |
| [**Reservations (RI)**](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/save-compute-costs-reservations) | 1/3-year commit for specific SKU/region; up to ~72% off | Predictable, steady VM/SQL/Cosmos/Storage workloads |
| [**Savings Plan for compute**](https://learn.microsoft.com/en-us/azure/cost-management-billing/savings-plan/savings-plan-compute-overview) | Hourly $ commit, flexes across VM/Functions Premium/App Service/Container Instances across regions | Variable compute mix, want flexibility at ~modest discount |
| [**Azure Hybrid Benefit**](https://learn.microsoft.com/en-us/azure/cost-management-billing/scope-level/azure-hybrid-benefit) | BYO Windows Server / SQL licenses | Existing EA/SA customers |
| [**Spot VMs**](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms) | Deeply discounted, evictable | Stateless batch / CI |
| [**Tagging + chargeback**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources) | Tag-driven cost allocation via Policy [`Modify`](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#tags)/[`Append`](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects#append) | Always — enforce with policy |

**Decision:** RIs when SKU+region are stable, **Savings Plan** when the mix shifts. Budgets are not enforcement — they alert; enforcement comes from Policy ([restrict SKUs](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#compute)/regions).

---

## 8. [Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview) design

### [Centralized vs federated](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design)

| Model | When |
|---|---|
| **Single centralized workspace** | Small-to-medium org, one compliance boundary, simplest RBAC; [**Microsoft's default recommendation**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design#design-strategy) |
| **Regional workspaces** | [Data residency / sovereignty](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-security) requirements force data to stay in-region |
| **Per-BU workspaces** | Hard isolation of ops teams, separate chargeback, separate retention |
| **Hybrid (hub + spokes)** | Central SOC workspace for security (Sentinel), spokes for app ops; use [**workspace cross-query**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cross-workspace-query) |

Principles:
- **Minimize workspace count.** Every split costs you cross-query complexity, duplicated ingestion, and fragmented RBAC.
- [**Sentinel = dedicated workspace**](https://learn.microsoft.com/en-us/azure/sentinel/design-your-workspace-architecture) (separate from general ops) and enable it at workspace scope.
- [**Table-level RBAC**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access#table-level-azure-rbac) + [**resource-context RBAC**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access#access-modes) let you share one workspace across teams safely.
- [**Commitment tiers**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs#commitment-tiers) (100 GB/day, 200, 300, 400, 500, 1 000, 2 000, 5 000 GB/day) cut ingestion cost vs pay-as-you-go; pick based on P90 ingest.
- [**Basic Logs**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-table-plans) / [**Auxiliary Logs**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-table-plans#auxiliary-table-plan) table plan for high-volume, low-query-frequency data (e.g., firewall/NSG flow logs), cheap ingest with limited query + [search job](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/search-jobs).
- [**Data Collection Rules (DCRs)**](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview) are the modern ingestion pipeline (MMA agent is retired — use [**Azure Monitor Agent**](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-overview)).

### [Retention](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-archive)

- **Interactive retention**: up to 730 days (2 years).
- **Archive (long-term)**: up to 12 years via [search jobs / restore](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/restore).

---

## 9. Application Insights — [workspace-based mode](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource)

- [**Classic App Insights retired 29 Feb 2024**](https://azure.microsoft.com/en-us/updates/we-re-retiring-classic-application-insights-on-29-february-2024/). Only **workspace-based** resources can be created now; existing classic resources were migrated.
- Workspace-based = telemetry stored in a Log Analytics workspace (same [KQL](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/), cross-query with infra logs).
- Design: one workspace per env (or shared platform workspace), multiple App Insights components sending to it.
- [Sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling): adaptive by default; tune via SDK or [Ingestion Sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling#ingestion-sampling) for cost control.
- Use [**connection strings**](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sdk-connection-string) (not instrumentation keys alone — they're deprecated for new features).

---

## 10. [Diagnostic settings routing](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings)

Every Azure resource with [`Microsoft.Insights/diagnosticSettings`](https://learn.microsoft.com/en-us/azure/templates/microsoft.insights/diagnosticsettings) can route **metrics + logs** to up to **5 destinations per setting**:

| Destination | Use |
|---|---|
| [**Log Analytics workspace**](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs#send-to-log-analytics-workspace) | Query, dashboards, alerts, Sentinel |
| [**Storage account**](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs#send-to-azure-storage) | Long-term cheap archive, regulatory retention |
| [**Event Hubs**](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs#send-to-azure-event-hubs) | Stream to SIEM / 3rd-party (Splunk, Elastic) |
| [**Partner solutions**](https://learn.microsoft.com/en-us/azure/partner-solutions/overview) | Datadog, etc. |

Enforce with [`DeployIfNotExists` policy](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings-policy) (built-in initiative: [*Enable Azure Monitor for VMs*](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-enable-policy) / *for Resource Manager*).

---

## 11. Alerting & action groups

| Alert type | Signal source |
|---|---|
| [**Metric alerts**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-metric-overview) | Platform or custom metrics (near-real-time) |
| [**Log alerts**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-log) | Scheduled KQL queries against Log Analytics / App Insights |
| [**Activity log alerts**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/activity-log-alerts) | Control-plane events (e.g., service health, resource delete) |
| [**Smart detection**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/proactive-diagnostics) | AI-driven App Insights anomalies |

[**Action Groups**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups) are the reusable notification/automation target: email/SMS/push, [webhook](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups#webhook), [ITSM](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/itsmc-overview) (ServiceNow), [**Logic App**](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-overview), [**Azure Function**](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview), [**Automation Runbook**](https://learn.microsoft.com/en-us/azure/automation/automation-runbook-types), [**Event Hub**](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about). Design one action group per severity/team and reuse across alerts.

Common misstep: creating 1 action group per alert. Don't — reuse.

---

## 12. [Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction) — posture

Two plan tiers for [**CSPM**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-cloud-security-posture-management) (posture):

| Plan | Includes |
|---|---|
| [**Foundational CSPM**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-cloud-security-posture-management#plan-availability) (free, on by default) | [Secure Score](https://learn.microsoft.com/en-us/azure/defender-for-cloud/secure-score-security-controls), [Microsoft cloud security benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/overview) assessments, [recommendations](https://learn.microsoft.com/en-us/azure/defender-for-cloud/review-security-recommendations) |
| [**Defender CSPM**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/tutorial-enable-cspm-plan) (paid) | [Attack path analysis](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-attack-path), [Cloud Security Graph](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-attack-path#what-is-cloud-security-graph), [agentless scanning](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-agentless-data-collection), [data-aware posture](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-data-security-posture), [code-to-cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction), DevOps posture, AI security posture |

Separate [**workload protection (CWP)**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction#protect-cloud-workloads) plans per resource type ([Servers P1/P2](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-servers-introduction), [Storage](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-storage-introduction), [SQL](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-sql-introduction), [App Service](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-app-service-introduction), [Key Vault](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-key-vault-introduction), [Containers](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction), [APIs](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-apis-introduction), [Cosmos](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-defender-for-cosmos), [DNS](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-dns-introduction)…).

Design guidance:
- Enable Defender plans at **subscription** scope via [Policy initiative *Configure Microsoft Defender for Cloud plans*](https://learn.microsoft.com/en-us/azure/defender-for-cloud/auto-deploy-vulnerability-assessment).
- [**Secure Score**](https://learn.microsoft.com/en-us/azure/defender-for-cloud/secure-score-security-controls) is your KPI — surface in exec dashboards ([Workbooks](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview)).
- Microsoft is unifying CSPM views into [**Microsoft Security Exposure Management**](https://learn.microsoft.com/en-us/security-exposure-management/microsoft-security-exposure-management) (single pane across Defender XDR + Defender for Cloud) — note for current state but AZ-305 still tests Defender for Cloud terminology directly.
- [Pricing](https://azure.microsoft.com/en-us/pricing/details/defender-for-cloud/) · [What's new](https://learn.microsoft.com/en-us/azure/defender-for-cloud/release-notes)

---

## Exam traps

- **"Passwords must never leave on-prem"** → [PTA](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-pta) or [Federation](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-fed), **not PHS**. If the option says PHS-only, that's the distractor.
- [**PHS**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-phs) is still the recommended default for resilience (cloud auth survives on-prem outage) and for [**leaked-credential detection**](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks) in Identity Protection.
- [**Federation (AD FS)**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-fed) is almost never the right answer in 2026 unless a specific requirement (smartcard / 3rd-party MFA) explicitly needs it.
- [**Entra Cloud Sync**](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/plan-cloud-sync-topologies) is the go-to for **multiple disconnected forests** — Connect Sync can't handle disconnected forests without trust.
- [**Azure Blueprints**](https://learn.microsoft.com/en-us/azure/governance/blueprints/overview): if you see it as an answer, it's wrong — **deprecated, retiring July 2026**. Correct answer is [**Deployment Stacks**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks) (± Policy).
- [**Deny assignments**](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) cannot be created directly — only via [**Managed Apps**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/overview) or [**Deployment Stacks**](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks#protect-managed-resources-against-deletion).
- [**Policy DINE/Modify**](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources) requires a **managed identity** on the assignment; forgetting this is a classic gotcha.
- Policy [**effects**](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects) don't retroactively fix — you must create a [**remediation task**](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources).
- [**Reservations**](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/save-compute-costs-reservations) are SKU+region locked; if workload mix changes, the right answer is [**Savings Plan for compute**](https://learn.microsoft.com/en-us/azure/cost-management-billing/savings-plan/savings-plan-compute-overview), not RI.
- [**Budgets are alerts, not enforcement**](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets). Enforcement needs Policy (restrict SKUs/regions).
- [**Classic Application Insights is gone (Feb 2024)**](https://azure.microsoft.com/en-us/updates/we-re-retiring-classic-application-insights-on-29-february-2024/). If an answer says "create classic App Insights resource" or "use instrumentation key only" — it's wrong. Use [**workspace-based**](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource) + [**connection string**](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sdk-connection-string).
- [**Log Analytics workspace count**](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design): default to **fewer**. Options suggesting "one workspace per subscription" or "per resource group" are usually distractors.
- [**Sentinel**](https://learn.microsoft.com/en-us/azure/sentinel/design-your-workspace-architecture) should live in its **own** Log Analytics workspace (security data + pricing model differ from ops).
- **[Diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings) ≠ Activity log export.** Activity log is a separate source; diagnostic settings now cover it, but older questions split them.
- [**Action Groups**](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups): reuse across alerts — answers that pair one Action Group per alert rule are the distractor.
- [**PIM**](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure) vs [**Access Reviews**](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview) vs [**Entitlement Management**](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-overview): PIM = JIT privileged elevation; Access Reviews = periodic attestation of *existing* access; Entitlement Management = lifecycle of *packages* of access (esp. external users). Don't swap them.
- [**AU (Administrative Units)**](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units) scope **Entra** roles, **not** Azure RBAC. Don't pick AU for "delegate VM contributor for APAC."
- [**ABAC conditions**](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) apply only to a subset of data-plane roles (Blob/Queue primarily). If the question is Compute or SQL, ABAC is wrong — use custom RBAC or separate RGs.
- [**Managed Identity supported?**](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identities-status) Not everywhere. If an option says "assign MI to [service X]" always sanity-check — several PaaS services still need a workaround.
- [**Workload Identity Federation**](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation): OIDC issuer/subject/audience triplet. Wrong `subject` (e.g., missing branch ref) = silent auth failure. Exam loves this detail.
- [**Entra Connect**](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history) must be **≥ 2.5.79.0 by Sep 30, 2026**. Any answer that keeps an older agent alive past that date is wrong.

---

## Sources — grouped for AZ-305 revision

### Exam & study
- [Exam AZ-305 — home](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/)
- [AZ-305 study guide (skills measured)](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305)
- [Microsoft Learn path: Design identity, governance, monitor](https://learn.microsoft.com/en-us/training/paths/design-identity-governance-monitor-solutions/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/) · [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)

### Identity
- [Choose the right authentication method](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn)
- [Entra Cloud Sync — what it is](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync) · [Topologies](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/plan-cloud-sync-topologies)
- [Entra Connect version history (Sep 2026 deadline)](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history)
- [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview) · [Authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)
- [PIM overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure) · [Identity Protection](https://learn.microsoft.com/en-us/entra/id-protection/overview-identity-protection)
- [Access Reviews](https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview) · [Entitlement Management](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-overview)
- [Managed identities overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) · [Workload Identity Federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)

### Governance
- [Management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview) · [Enterprise-scale landing zones](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) · [ABAC conditions](https://learn.microsoft.com/en-us/azure/role-based-access-control/conditions-overview) · [Deny assignments](https://learn.microsoft.com/en-us/azure/role-based-access-control/deny-assignments)
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview) · [Effects](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects) · [Remediation](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources)
- [Deployment Stacks](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks) · [`az stack` CLI](https://learn.microsoft.com/en-us/cli/azure/stack/sub) · [Blueprints deprecation](https://learn.microsoft.com/en-us/azure/governance/blueprints/overview)
- [Microsoft cloud security benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)

### Cost
- [Cost Management + Billing](https://learn.microsoft.com/en-us/azure/cost-management-billing/cost-management-billing-overview) · [Budgets](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [Reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/save-compute-costs-reservations) · [Savings Plan for compute](https://learn.microsoft.com/en-us/azure/cost-management-billing/savings-plan/savings-plan-compute-overview) · [Hybrid Benefit](https://learn.microsoft.com/en-us/azure/cost-management-billing/scope-level/azure-hybrid-benefit)

### Monitoring
- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Log Analytics workspace design](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-design) · [Table plans (Basic/Aux)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-table-plans) · [Commitment tiers](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs#commitment-tiers)
- [Data Collection Rules](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview) · [Azure Monitor Agent](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-overview)
- [App Insights workspace-based](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource) · [Classic retirement notice](https://azure.microsoft.com/en-us/updates/we-re-retiring-classic-application-insights-on-29-february-2024/)
- [Diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings) · [Diagnostic settings policy](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings-policy)
- [Alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview) · [Action Groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [Microsoft Sentinel workspace architecture](https://learn.microsoft.com/en-us/azure/sentinel/design-your-workspace-architecture)

### Defender for Cloud
- [Defender for Cloud intro](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction) · [CSPM concept](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-cloud-security-posture-management)
- [Enable Defender CSPM](https://learn.microsoft.com/en-us/azure/defender-for-cloud/tutorial-enable-cspm-plan) · [Secure Score](https://learn.microsoft.com/en-us/azure/defender-for-cloud/secure-score-security-controls) · [Attack path analysis](https://learn.microsoft.com/en-us/azure/defender-for-cloud/concept-attack-path)
- [Pricing](https://azure.microsoft.com/en-us/pricing/details/defender-for-cloud/) · [What's new](https://learn.microsoft.com/en-us/azure/defender-for-cloud/release-notes)
- [Microsoft Security Exposure Management](https://learn.microsoft.com/en-us/security-exposure-management/microsoft-security-exposure-management)
