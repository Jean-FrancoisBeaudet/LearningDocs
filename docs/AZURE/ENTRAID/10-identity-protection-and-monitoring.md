# Identity Protection & Monitoring

> **Exam mapping:** AZ-305 *(Design monitoring & governance)* · AZ-500 (adjacent) · AZ-204 (sign-in/audit logs)
> **One-liner:** ML-driven risk detection on users and sign-ins (Identity Protection, P2) + four log streams piped to Log Analytics/Sentinel for everything else.
> **Related:** [conditional access](06-conditional-access-and-mfa.md) · [PIM](07-privileged-identity-management.md)

## Identity Protection (P2)

Three risk surfaces:

| Risk type | What's evaluated | Examples of detections |
|-----------|------------------|------------------------|
| **Sign-in risk** | The properties of *this* sign-in event | Anonymous IP, atypical travel, unfamiliar sign-in properties, malware-linked IP, password spray, anomalous token, suspicious browser, threat intel match |
| **User risk** | Aggregate risk across the user account | Leaked credentials (dark-web match), Microsoft threat intel on the user, anomalous user behavior, possible attempt to access primary refresh token |
| **Workload identity risk** | Risk on **service principals / managed identities** | Leaked credentials for SP, SP suspicious sign-ins, anomalous SP behavior; requires **Workload ID Premium** |

Each detection assigns a level: **None / Low / Medium / High** with a confidence score.

### Risk policies

| Policy | Where to configure | What it does |
|--------|--------------------|--------------|
| **User-risk policy** | **Conditional Access** (legacy ID Protection policies retire **October 1, 2026** — migrate now) | If user risk ≥ X → require password change + MFA |
| **Sign-in-risk policy** | Conditional Access | If sign-in risk ≥ X → require MFA |
| **MFA registration policy** | Identity Protection | Force users to register MFA methods within N days |

> **Migration notice:** the legacy ID Protection user-risk and sign-in-risk policies are being retired October 1, 2026. Recreate them as **Conditional Access** policies that consume the **User risk** / **Sign-in risk** conditions. Use **report-only** during transition.

### Remediation

- **Self-remediation:** secure password change (user risk) or MFA challenge (sign-in risk) closes the risk automatically.
- **Admin remediation:** confirm compromised, dismiss, confirm safe, reset password, block sign-in.
- **Risky users / risky sign-ins / risk detections** blades + Microsoft Graph (`/identityProtection/riskyUsers`, `/riskDetections`).

## Sign-in / audit / provisioning logs

Entra exposes four primary log streams:

| Log | What it captures | Retention (Free) | Retention (P1/P2) |
|-----|------------------|------------------|--------------------|
| **Sign-in logs** | Every interactive + non-interactive sign-in, plus SP sign-ins, MI sign-ins, ADFS, B2B redemptions | 7 days | 30 days |
| **Audit logs** | Directory changes (user/group/app/role mgmt, policy edits) | 7 days | 30 days |
| **Provisioning logs** | SCIM provisioning to/from SaaS apps and HR sources | 7 days | 30 days |
| **Risk detections** (Identity Protection) | Each detection event | n/a (P2) | 90 days |

For longer retention or correlation, **export via Diagnostic settings**.

### Diagnostic settings — destinations

| Destination | Use |
|-------------|-----|
| **Log Analytics workspace** | Kusto queries, Workbooks, alerts, Sentinel |
| **Storage account** | Cheap long-term archive |
| **Event Hub** | Stream to SIEM (Splunk, QRadar, Elastic) |
| **Partner solution** | Specific integrations |

```bash
# Send sign-in + audit logs to a Log Analytics workspace
az monitor diagnostic-settings create \
  --name "entra-to-loganalytics" \
  --resource "/providers/Microsoft.aadiam/diagnosticSettings/default" \
  --workspace $WORKSPACE_ID \
  --logs '[
    {"category":"SignInLogs","enabled":true},
    {"category":"AuditLogs","enabled":true},
    {"category":"NonInteractiveUserSignInLogs","enabled":true},
    {"category":"ServicePrincipalSignInLogs","enabled":true},
    {"category":"ManagedIdentitySignInLogs","enabled":true},
    {"category":"ProvisioningLogs","enabled":true},
    {"category":"ADFSSignInLogs","enabled":true},
    {"category":"RiskyUsers","enabled":true},
    {"category":"UserRiskEvents","enabled":true}
  ]'
```

> Sign-in log categories split by **interactive / non-interactive / SP / MI** — pipe all the ones you need; storage cost is modest compared to incident-response value.

### Useful KQL snippets

```kusto
// All risky sign-ins by user, last 7d
SigninLogs
| where TimeGenerated > ago(7d)
| where RiskLevelDuringSignIn in ("medium","high")
| summarize count() by UserPrincipalName, RiskLevelDuringSignIn
| order by count_ desc

// Failed sign-ins by app + error code
SigninLogs
| where ResultType != 0
| summarize count() by AppDisplayName, ResultType, ResultDescription
| top 50 by count_

// New app registrations in audit log
AuditLogs
| where OperationName == "Add application"
| project TimeGenerated, InitiatedBy, TargetResources

// Service principal sign-ins from outside known IPs
AADServicePrincipalSignInLogs
| where ResultType == 0
| where ipv4_is_private(IPAddress) == false
| summarize count() by ServicePrincipalName, IPAddress
```

## Entra Workbooks

Pre-built Azure Monitor workbooks for: Conditional Access insights & reporting (great for "what would this policy block in report-only?"), sign-ins by location, MFA gaps, app sign-in patterns, risk analytics, B2B activity. Backed by the Log Analytics data above.

## Microsoft Sentinel integration

For SOC scenarios, route Entra logs into **Sentinel** (built on Log Analytics + analytics rules + automation):

- Built-in **Microsoft Entra ID** data connector (free for sign-in & audit logs *into Sentinel*; data ingestion to LA priced separately for some categories).
- 100+ analytics rule templates (impossible travel, brute force, MFA fatigue, dormant-account login, mass download by guest).
- UEBA add-on for behavioral anomalies.

## Microsoft Defender for Identity

Different product. Agent installed on **on-prem Active Directory domain controllers** + ADFS / AD CS servers. Detects on-prem attacks (Kerberoasting, Pass-the-Hash, DCSync, golden ticket, lateral movement). Signals **flow into Identity Protection and XDR** so a compromised on-prem account elevates risk in the cloud.

| | Identity Protection | Defender for Identity |
|---|---------------------|----------------------|
| Scope | Entra ID (cloud) | On-prem AD + AD CS + ADFS |
| Sensor | None — cloud signals | Sensor on DCs |
| Licensing | Entra ID P2 | Microsoft Defender for Identity (per user) |
| Risk fed to | CA risk policies | Same — feeds into XDR + Identity Protection user risk |

## Recommended monitoring baseline

- Diagnostic settings → **Log Analytics + Storage** (archive).
- Daily review of **risky users** + **sign-in logs filtered by ResultType ≠ 0**.
- Sentinel rules: **impossible travel**, **MFA fatigue (rapid prompts → success)**, **app consent grant**, **role assignment outside PIM**, **sign-in from anonymous proxy**.
- Alerts on **Global Admin sign-in from non-named-location**, **break-glass account use** (any sign-in is a paging event).
- Review **CA report-only policy outcomes** weekly via Workbook before flipping to Enabled.

## Exam traps

- **Identity Protection requires P2.** Free/P1 see no risk policies, no risky-users blade.
- **Legacy ID Protection risk policies retire October 1, 2026** — answers using them are wrong; recreate in **Conditional Access**.
- **Entra free retention is 7 days** for sign-in/audit. Need 30+ days? P1/P2 + diagnostic export.
- **Different sign-in log categories.** Forgetting `NonInteractiveUserSignInLogs` / `ServicePrincipalSignInLogs` means missing the very events attackers exploit.
- **Diagnostic settings live at the tenant level**, not on a subscription — the resource path `/providers/Microsoft.aadiam/diagnosticSettings/default`.
- **Workload identity risk needs Workload ID Premium**, separate from P2.
- **Defender for Identity is on-prem**; Identity Protection is cloud. Don't conflate.
- **CAE revokes on risk** — once Identity Protection raises a user to high risk, CAE-aware resources reject the token immediately.

## Sources

- [Microsoft Entra ID Protection overview](https://learn.microsoft.com/en-us/entra/id-protection/overview-identity-protection)
- [Risk policies in CA](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies)
- [Sign-in & audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-sign-ins)
- [Diagnostic settings for Entra](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-configure-diagnostic-settings)
- [Microsoft Sentinel Entra connector](https://learn.microsoft.com/en-us/azure/sentinel/connect-azure-active-directory)
- [Defender for Identity](https://learn.microsoft.com/en-us/defender-for-identity/what-is)
