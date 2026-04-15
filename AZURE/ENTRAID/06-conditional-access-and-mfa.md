# Conditional Access & MFA

> **Exam mapping:** AZ-305 · AZ-204 *(security context)*
> **One-liner:** "If signal then control" engine — Entra evaluates assignments + conditions and applies grant/session controls before issuing the token.
> **Related:** [auth protocols](03-authentication-protocols.md) · [identity protection](10-identity-protection-and-monitoring.md) · [PIM](07-privileged-identity-management.md)

## CA policy anatomy

```
Assignments  →  Conditions  →  Access controls (Grant + Session)
```

A policy is **evaluated for every sign-in** (interactive and silent) after the user authenticates but before the token is issued. All matching policies' grant controls combine with **AND**.

| Section | Examples |
|---------|----------|
| **Users** | Include/exclude users, groups, roles, guests, workload identities |
| **Target resources** | Cloud apps (e.g. Office 365, specific app reg), user actions (Register security info, Register/join device), workload identities, **authentication context** (custom labels like `c1` for "step-up to view payroll") |
| **Conditions** | User risk, sign-in risk, device platform, location (named locations), client apps (browser, mobile, modern auth, legacy), device state (compliant, hybrid joined), filter for devices, insider risk |
| **Grant controls** | Block; or require — MFA, **authentication strength**, compliant device, hybrid Entra-joined device, app protection policy, ToU, password change. Combine with AND or OR. |
| **Session controls** | Use app-enforced restrictions, Defender for Cloud Apps (CAS) conditional access app control, sign-in frequency, persistent browser session, customize CAE, disable resilience defaults, token protection (preview/GA) |

> Policies require **Entra ID P1**. Without it you only get **Security Defaults** (all-or-nothing baseline: MFA for admins always, MFA for users on risky sign-ins, block legacy auth).

## Authentication strengths (built-in)

| Strength | What satisfies it |
|----------|-------------------|
| **Multifactor authentication** | Any combination satisfying classic MFA — password + Authenticator push/OTP/SMS/voice/HW OATH/FIDO2/Windows Hello/cert. |
| **Passwordless MFA** | Authenticator (passwordless), Windows Hello, FIDO2, certificate-based auth (multi-factor). |
| **Phishing-resistant MFA** | FIDO2/passkey, Windows Hello for Business, certificate-based auth (multi-factor). **Excludes** Authenticator push, SMS, voice. |

You can also create **custom authentication strengths** to combine specific methods (e.g. "Authenticator passwordless OR FIDO2 only").

> Phishing-resistant MFA is the bar Microsoft now recommends for **all admins and high-value workloads**.

## MFA methods (current)

| Method | Phishing-resistant? | Notes |
|--------|---------------------|-------|
| **Passkey (FIDO2)** — device-bound or synced | **Yes** | The recommended method. Hardware keys + platform passkeys (Authenticator on iOS/Android). |
| **Windows Hello for Business** | **Yes** | Per-device, TPM-backed. |
| **Certificate-based authentication (CBA)** | **Yes** (when issued from trusted CA + smart card) | Cloud-native CBA replaces ADFS-based CBA. |
| **Microsoft Authenticator (push, passwordless)** | No (push), Partial (passwordless sign-in stronger) | Number matching is mandatory; geographic context shown. |
| **OATH HW/SW token** | No | TOTP. |
| **SMS / voice** | No | **Avoid** — phishable, SIM-swap risk. Microsoft is moving customers off these. |
| **Temporary Access Pass (TAP)** | n/a (bootstrapping) | Time-boxed code for new users to register a real method. |

### Per-user MFA — deprecated

The legacy **per-user MFA** model (configured in `account.activedirectory.windowsazure.com`) is being retired in favor of CA. Existing assignments still work; new tenants should use Conditional Access exclusively. Microsoft has been moving tenants off per-user MFA throughout 2025–2026.

### Security Defaults

- Free, on by default for new tenants since 2019.
- Forces MFA for admins, MFA prompt for risky users, blocks legacy auth.
- **Mutually exclusive with Conditional Access** — turn it off when you enable CA.

## Common policy patterns (memorize these scenarios)

| Goal | Policy sketch |
|------|---------------|
| Block legacy auth | Users: All. Apps: All. Conditions → Client apps: Exchange ActiveSync + Other clients. Grant: **Block**. |
| MFA for everyone | Users: All; Exclude break-glass. Apps: All. Grant: Require MFA. |
| Phishing-resistant MFA for admins | Users: Directory roles → all admin roles. Grant: Require auth strength = Phishing-resistant. |
| Compliant device for corporate apps | Apps: O365 + LOB apps. Grant: Require compliant device OR hybrid joined. |
| Block sign-in from unsupported countries | Conditions → Locations → exclude allowed countries. Grant: Block. |
| Require step-up for sensitive operation | App declares `authenticationContext` `c1`. Policy targets `c1` → Grant: Require auth strength. |
| Risky sign-in → MFA | Conditions: Sign-in risk = High/Medium. Grant: Require MFA. |
| Risky user → password change | Conditions: User risk = High. Grant: Require password change + MFA. |
| Token protection (sign-in token binding) | Session: **Require token protection for sign-in sessions** (Windows desktop apps; preview/GA in waves). |

> Always **exclude break-glass accounts** from every CA policy. Always run a new policy in **Report-only** first; check sign-in logs filter `Conditional Access = Report-only success/failure`.

## Named locations

| Type | Notes |
|------|-------|
| **IP ranges** (IPv4/IPv6 CIDR) | Mark trusted to influence MFA prompts. Limit ~2 000 CIDR ranges per location. |
| **Countries/regions** | GeoIP-based. Include "Unknown countries/regions" if needed. |
| **MFA trusted IPs** *(legacy)* | From per-user MFA — replaced by named locations. |

## Token lifetimes & sign-in frequency

- Default access token ~1h; refresh tokens up to 90d inactivity / 24h for confidential clients in some flows.
- **Configurable token lifetime policies (CTL)** for SAML tokens; for OAuth tokens use **Sign-in frequency** in CA session controls.
- **CAE** (Continuous Access Evaluation) shortens effective session for revocation events.
- **Persistent browser session** controls "Stay signed in?" prompt.

## Workload identity Conditional Access

CA can also target **service principals and managed identities** (workload identities) — restrict by IP location, risk level (workload identity risk in Identity Protection), and require named locations. Requires **Entra Workload ID Premium**.

## Bicep (Conditional Access via Microsoft Graph beta)

CA policies are managed via Microsoft Graph (`/identity/conditionalAccess/policies`) — not ARM/Bicep. Use Graph PowerShell or Terraform's `azuread_conditional_access_policy`.

```powershell
$body = @{
  displayName = "CA001 - Block legacy auth"
  state = "enabledForReportingButNotEnforced"   # report-only
  conditions = @{
    users = @{ includeUsers = @("All"); excludeGroups = @("<break-glass-group>") }
    applications = @{ includeApplications = @("All") }
    clientAppTypes = @("exchangeActiveSync","other")
  }
  grantControls = @{ operator = "OR"; builtInControls = @("block") }
} | ConvertTo-Json -Depth 10

Invoke-MgGraphRequest -Method POST `
  -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" `
  -Body $body
```

## Exam traps

- **CA needs P1.** With Free, you have only Security Defaults.
- **You can't run Security Defaults + CA at the same time.**
- **All matching policies AND together** for grant controls — a `Block` in any matching policy wins.
- **"Require MFA" ≠ "Require phishing-resistant MFA"** — only `authenticationStrength` enforces method choice.
- **Exclude break-glass accounts from every policy** — and monitor their sign-ins separately.
- **Filter for devices** uses device extension attributes; preview features may differ on the exam.
- **Per-user MFA is deprecated** — answers using it are wrong on a 2026 exam.
- **Number matching in Authenticator is mandatory** since Feb 2023 — don't propose disabling it.
- **CAE-aware tokens have longer life but can be revoked instantly** — apps must handle claims challenges.

## Sources

- [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)
- [Authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)
- [Passkeys (FIDO2)](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-passkeys-fido2)
- [Security Defaults](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults)
- [Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)
- [Per-user MFA migration](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-mfa-server-migration-utility)
