# Microsoft Entra Users and Groups

> **Exam mapping:** AZ-104 -- "Manage Azure identities and governance" (20-25%)
> **One-liner:** Create, configure, and manage Entra ID users, groups, licenses, external identities, and self-service password reset -- the identity foundation everything else in Azure sits on.
> **Related:** [02-access (RBAC & Conditional Access)](02-access.md) | [03-subscriptions](03-subscriptions.md) | [AZURE/ENTRAID/ series](../ENTRAID/)

---

## 1. User Types

### Cloud-only vs Synced

| Aspect | Cloud-only | Synced (Entra Connect) |
|---|---|---|
| **Source of authority** | Microsoft Entra ID | On-premises AD DS |
| **Created in** | Azure portal / CLI / PowerShell | On-prem AD, then synced by [Microsoft Entra Connect](https://learn.microsoft.com/en-us/entra/identity/hybrid/whatis-azure-ad-connect) |
| **Password managed in** | Entra ID | On-prem AD (unless password hash sync or passthrough auth enabled) |
| **Editable in Entra portal** | All properties | Only cloud-managed properties (synced attributes are read-only) |
| **Deletion** | Immediate soft-delete | Must be deleted/disabled at source; sync propagates |

### Member vs Guest

| Aspect | Member | Guest |
|---|---|---|
| **UserType** | `Member` | `Guest` |
| **Typical source** | Internal employee | External collaborator (B2B invite) |
| **Default permissions** | Can read directory objects, create groups, register apps | Restricted -- cannot enumerate users/groups by default |
| **License consumption** | Requires license for premium features | Some features covered under inviting tenant's licenses; MAU-based billing for External ID |

> **Exam tip:** A user's *source* (cloud vs synced) is independent of their *UserType* (member vs guest). You can invite an external user as a **Member** or convert an internal user to **Guest**.

---

## 2. Create Users and Groups

### Create Users

#### Azure Portal

**Entra admin center** > **Users** > **All users** > **+ New user** > **Create new user**

Required fields: Display name, User principal name (must use a verified domain), password (auto-generated or custom).

#### Azure CLI

```bash
# Create a cloud user
az ad user create \
  --display-name "Alice Smith" \
  --user-principal-name alice@contoso.com \
  --password "P@ssw0rd1234!" \
  --mail-nickname alice

# List users
az ad user list --output table
```

Reference: [az ad user create](https://learn.microsoft.com/en-us/cli/azure/ad/user?view=azure-cli-latest)

#### PowerShell (Microsoft Graph)

```powershell
# Install and connect
Install-Module Microsoft.Graph -Scope CurrentUser
Connect-MgGraph -Scopes "User.ReadWrite.All"

# Create user
$passwordProfile = @{
    Password                      = "P@ssw0rd1234!"
    ForceChangePasswordNextSignIn = $true
}

New-MgUser -DisplayName "Alice Smith" `
           -UserPrincipalName "alice@contoso.com" `
           -PasswordProfile $passwordProfile `
           -AccountEnabled `
           -MailNickname "alice"
```

Reference: [Manage users with Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/manage-user?view=entra-powershell)

#### Key Properties

| Property | Notes |
|---|---|
| `UserPrincipalName` | Must contain a verified domain of the tenant |
| `DisplayName` | Required |
| `MailNickname` | Alias for mail routing |
| `UsageLocation` | **Required before assigning licenses** -- two-letter ISO country code |
| `AccountEnabled` | `true` / `false` |
| `JobTitle`, `Department`, `CompanyName` | Organizational metadata; can drive dynamic group rules |

### Create Groups

#### Azure Portal

**Entra admin center** > **Groups** > **All groups** > **+ New group**

Choose group type (Security or Microsoft 365), name, description, membership type.

#### Azure CLI

```bash
# Security group
az ad group create \
  --display-name "SG-DevTeam" \
  --mail-nickname "sg-devteam"

# Add member
az ad group member add \
  --group "SG-DevTeam" \
  --member-id <user-object-id>

# Check membership
az ad group member check \
  --group "SG-DevTeam" \
  --member-id <user-object-id>
```

Reference: [az ad group](https://learn.microsoft.com/en-us/cli/azure/ad/group?view=azure-cli-latest)

#### PowerShell (Microsoft Graph)

```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All"

# Security group with assigned membership
New-MgGroup -DisplayName "SG-DevTeam" `
            -MailEnabled:$false `
            -SecurityEnabled:$true `
            -MailNickname "sg-devteam" `
            -GroupTypes @()

# Microsoft 365 group
New-MgGroup -DisplayName "M365-Marketing" `
            -MailEnabled:$true `
            -SecurityEnabled:$false `
            -MailNickname "m365-marketing" `
            -GroupTypes @("Unified")
```

Reference: [Manage groups -- Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/manage-groups?view=entra-powershell)

---

## 3. Group Types: Security vs Microsoft 365

| Feature | Security Group | Microsoft 365 Group |
|---|---|---|
| **Purpose** | Manage access to resources (RBAC, NSGs, etc.) | Collaboration (shared mailbox, SharePoint, Teams) |
| **Members** | Users, devices, service principals, other groups | **Users only** (no devices) |
| **Mail-enabled** | Optional | Always |
| **Owners** | Optional | At least one owner recommended |
| **Can be role-assignable** | Yes (requires P1, `isAssignableToRole = true` at creation, **immutable after**) | Yes (same constraints) |
| **Nesting** | Supported | Not supported for M365 groups |

### Membership Types

| Type | How members are determined |
|---|---|
| **Assigned** | Manually added/removed by admins or owners |
| **Dynamic User** | Automatically evaluated from user attributes via rules |
| **Dynamic Device** | Automatically evaluated from device attributes (Security groups only) |

### Dynamic Membership Rules Syntax

Rules use the format: `<property> <operator> <value>`

```
# Users in the Engineering department
user.department -eq "Engineering"

# Users whose job title contains "Manager"
user.jobTitle -contains "Manager"

# Compound rule (AND / OR)
(user.department -eq "Sales") -and (user.country -eq "Canada")

# Null check
user.assignedPlans -any (assignedPlan.servicePlanId -eq "efb87545-963c-4e0d-99df-69c6916d9eb0" -and assignedPlan.capabilityStatus -eq "Enabled")
```

**Common operators:** `-eq`, `-ne`, `-contains`, `-notContains`, `-startsWith`, `-in`, `-notIn`, `-match`, `-any`, `-all`

**Limits and gotchas:**
- Maximum rule body length: **3,072 characters**
- Maximum dynamic groups per tenant: **15,000**
- Rule builder in the portal supports up to **5 expressions**; use the text box for more
- String operations are **case-insensitive**
- Processing can take **minutes to hours** depending on tenant size
- Security groups support both user and device rules; **M365 groups support user rules only**
- `memberOf` attribute rules cannot be combined with other attribute rules

Reference: [Dynamic membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership) | [Create/edit dynamic groups](https://learn.microsoft.com/en-us/entra/identity/users/groups-create-rule)

---

## 4. Manage User and Group Properties

### User Properties (Portal)

Navigate to **Users** > select user > **Properties**. Key editable fields:

- Identity: Display name, UPN, User type
- Job info: Title, Department, Company, Manager
- Contact info: Email, Phone, Street address
- Settings: Usage location, Block sign-in

For synced users, attributes mastered on-prem appear **greyed out** in the portal.

### Group Properties

Navigate to **Groups** > select group:

- **Properties:** Name, description, membership type (cannot change from dynamic to assigned once set), email, object ID
- **Members / Owners:** Add/remove (assigned groups only)
- **Dynamic membership rules:** Edit rule (dynamic groups only)
- **Licenses:** Assign/remove product licenses
- **Roles and administrators:** Assign Entra roles scoped to the group (role-assignable groups only)

### CLI Quick Reference

```bash
# Update user property
az ad user update --id alice@contoso.com --job-title "Senior Dev"

# Update group description
az ad group update --group "SG-DevTeam" --description "Development team security group"
```

### PowerShell Quick Reference

```powershell
# Update user
Update-MgUser -UserId "alice@contoso.com" -JobTitle "Senior Dev" -UsageLocation "CA"

# Update group
Update-MgGroup -GroupId "<group-object-id>" -Description "Development team security group"
```

---

## 5. Bulk Operations (CSV Upload)

[Bulk operations](https://learn.microsoft.com/en-us/entra/identity/users/users-bulk-add) are available in the Entra admin center under **Users** > **Bulk operations**:

| Operation | Description |
|---|---|
| **Bulk create** | Upload CSV to create multiple users at once |
| **Bulk invite** | Upload CSV to invite external (guest) users |
| **Bulk delete** | Upload CSV with UPNs to soft-delete users |
| **Download users** | Export user list to CSV |

### CSV Template (Bulk Create)

Download the template from the portal. Required columns:

| Column | Example |
|---|---|
| `Name [displayName]` | Alice Smith |
| `User name [userPrincipalName]` | alice@contoso.com |
| `Initial password [passwordProfile]` | P@ssw0rd1 |
| `Block sign in [accountEnabled]` | No |

**Limits ([Bulk operations service limitations](https://learn.microsoft.com/en-us/entra/fundamentals/bulk-operations-service-limitations)):**
- Each bulk operation must complete within **1 hour**
- If the job times out, split into smaller batches
- Requires at least the **User Administrator** role
- The CSV file itself can contain up to **50,000 rows**

For groups, similar bulk operations exist: **Bulk import members**, **Bulk remove members**, **Download members**.

---

## 6. Manage Licenses in Microsoft Entra ID

### Direct vs Group-Based Licensing

| Aspect | Direct Assignment | [Group-Based Licensing](https://learn.microsoft.com/en-us/entra/fundamentals/concept-group-based-licensing) |
|---|---|---|
| **How** | Assign license directly to user | Assign license to a group; members inherit |
| **Scale** | Fine for small orgs | Recommended at scale |
| **Automation** | Manual or scripted | Automatic -- add user to group, license follows |
| **Conflicts** | Immediate feedback | Errors recorded asynchronously; check via portal or Graph |
| **License required** | Any Entra edition | Requires at least **Entra ID P1** |

### Key Requirements

- **UsageLocation must be set** on the user before any license can be assigned. This is the most common "why won't my license assign?" issue.
- A user can have both direct and group-based assignments for the same product; Entra deduplicates.

### License Conflicts

[Resolving license problems](https://learn.microsoft.com/en-us/entra/fundamentals/licensing-groups-resolve-problems):

- **Insufficient licenses:** Not enough available licenses in the tenant; users enter an error state.
- **Conflicting service plans:** Two products contain overlapping service plans that cannot coexist on the same user (e.g., Exchange Online Plan 1 vs Plan 2).
- **Dependent service plan missing:** Some service plans require another plan to be enabled.
- Resolution is **always manual** -- Entra does not auto-resolve conflicts. Check **Users** > user > **Licenses** > **Resolve** or **Groups** > group > **Licenses** > **Reprocess**.

### CLI / PowerShell

```powershell
# Assign license to user (Microsoft Graph PowerShell)
Set-MgUserLicense -UserId "alice@contoso.com" `
  -AddLicenses @(@{SkuId = "<sku-guid>"}) `
  -RemoveLicenses @()

# Assign license to group
Set-MgGroupLicense -GroupId "<group-id>" `
  -AddLicenses @(@{SkuId = "<sku-guid>"}) `
  -RemoveLicenses @()

# List available SKUs
Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, PrepaidUnits
```

Reference: [Assign licenses to a group](https://learn.microsoft.com/en-us/entra/identity/users/licensing-groups-assign) | [PowerShell examples for group-based licensing](https://learn.microsoft.com/en-us/entra/identity/users/licensing-powershell-graph-examples)

---

## 7. Manage External Users (B2B)

[Microsoft Entra External ID (B2B)](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b) lets you invite external users to collaborate using their own identity (work/school account, Microsoft account, Google, etc.).

### Invite Flow

1. Admin or allowed user sends invitation (portal, CLI, PowerShell, or Graph)
2. Guest receives email with **redemption link**
3. Guest accepts and is added to the directory with `UserType = Guest`

### Redemption Order

The invited user redeems in this priority ([Invitation redemption](https://learn.microsoft.com/en-us/entra/external-id/redemption-experience)):

1. Microsoft Entra ID (if guest's home tenant exists)
2. Microsoft account
3. Email one-time passcode (OTP) -- if enabled
4. Google federation (if configured)
5. SAML/WS-Fed identity provider (if configured)
6. Direct federation

### External Collaboration Settings

**Entra admin center** > **External Identities** > **External collaboration settings**:

- **Guest invite restrictions:** Who can invite guests (Admins only, Members, Members + specific guests, Anyone including guests)
- **Collaboration restrictions:** Allow/deny invitations to specific domains (allowlist or denylist)
- **Guest user access level:** Same as members, limited access, or most restrictive

Reference: [Configure external collaboration settings](https://learn.microsoft.com/en-us/entra/external-id/external-collaboration-settings-configure)

### Access Packages (Entitlement Management)

For more governed external access, use [Access Packages](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-overview) under **Identity Governance**:

- Bundle group memberships, app access, and SharePoint sites into a single requestable package
- Define approval workflows, expiration, and access reviews
- External users can **request access** via a My Access portal link (no admin invite required)
- Requires **Entra ID P2** or **Entra ID Governance** license

### CLI / PowerShell

```bash
# Invite guest user via CLI
az ad user create --display-name "Bob External" \
  --user-principal-name "bob_partner.com#EXT#@contoso.onmicrosoft.com" \
  --password "TempP@ss1!" \
  --mail-nickname "bob_partner"
```

```powershell
# Invite guest via Microsoft Graph PowerShell
New-MgInvitation -InvitedUserEmailAddress "bob@partner.com" `
                 -InviteRedirectUrl "https://myapps.microsoft.com" `
                 -SendInvitationMessage:$true
```

### Key Properties of Guest Users

| Property | Value |
|---|---|
| `UserType` | Guest |
| `Source` | Invited user / External Microsoft Entra ID |
| `UserPrincipalName` | `user_domain#EXT#@tenant.onmicrosoft.com` |
| `CreationType` | Invitation |
| `ExternalUserState` | PendingAcceptance / Accepted |

Reference: [B2B guest user properties](https://learn.microsoft.com/en-us/entra/external-id/user-properties) | [Add B2B users](https://learn.microsoft.com/en-us/entra/external-id/add-users-administrator)

---

## 8. Configure Self-Service Password Reset (SSPR)

[Self-service password reset](https://learn.microsoft.com/en-us/entra/identity/authentication/tutorial-enable-sspr) allows users to reset their own passwords without helpdesk involvement.

### Enabling SSPR

**Entra admin center** > **Protection** > **Password reset**

| Setting | Options |
|---|---|
| **Enabled for** | None / Selected (specific groups) / All |
| **Authentication methods required** | 1 or 2 methods |
| **Methods available** | Mobile app notification, Mobile app code, Email, Mobile phone, Office phone, Security questions |
| **Security questions** | Number required to register / Number required to reset |

### Authentication Methods

| Method | Registration | Reset |
|---|---|---|
| Microsoft Authenticator (notification/code) | Yes | Yes |
| Email (alternate) | Yes | Yes |
| Mobile phone (SMS/call) | Yes | Yes |
| Office phone | Yes | Yes |
| Security questions | Yes | Yes (not available for admin accounts) |

> **Exam trap:** Admins are **always** enabled for SSPR and **always** require two methods. Admins **cannot** use security questions. These admin policies cannot be changed.

### Password Writeback

[Password writeback](https://learn.microsoft.com/en-us/entra/identity/authentication/tutorial-enable-sspr-writeback) synchronizes password changes from Entra ID back to on-premises AD DS:

- Requires **Microsoft Entra Connect** (or Cloud Sync) with writeback enabled
- Requires **Entra ID P1** license (minimum)
- Supports both SSPR and admin-initiated password resets
- Enforces on-prem password policies (complexity, history, age)
- Near real-time -- password updates propagate in seconds

### Combined Registration with MFA

[Combined security information registration](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined) provides a **single registration experience** at `https://aka.ms/mysecurityinfo` for both SSPR and MFA:

- Users register once for both services
- Managed via **Authentication methods** policy (unified), replacing legacy per-service policies
- Legacy MFA and SSPR method policies were deprecated September 30, 2025; all tenants should now use the unified [Authentication methods policy](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage)
- Can be enforced via **Conditional Access** > **User actions** > **Register security information**

### SSPR Licensing

| Feature | License Required |
|---|---|
| Cloud-only SSPR | Entra ID P1 |
| SSPR with on-prem writeback | Entra ID P1 |
| Combined registration | Any (enabled by default for new tenants) |

---

## 9. Administrative Units

[Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units) provide **scoped delegation** of directory admin roles to a subset of the organization.

### Concept

- An administrative unit is a container for users, groups, and/or devices
- You assign an Entra ID role (e.g., User Administrator, Helpdesk Administrator) **scoped to** an administrative unit
- The role holder can only manage objects inside that AU -- not tenant-wide

### Use Cases

- Regional IT teams managing users in their geography only
- Departmental admins managing their own department's groups
- School administrators in a multi-school university tenant

### Creating Administrative Units

#### Portal

**Entra admin center** > **Identity** > **Roles & admins** > **Admin units** > **+ Add**

#### PowerShell

```powershell
Connect-MgGraph -Scopes "AdministrativeUnit.ReadWrite.All"

New-MgDirectoryAdministrativeUnit -DisplayName "AU-Engineering" `
                                   -Description "Engineering department"
```

### Membership Types

| Type | Description |
|---|---|
| **Assigned** | Objects manually added |
| **Dynamic** | Objects automatically added based on rules (same syntax as dynamic groups) |

### Restricted Management AUs

[Restricted management administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-restricted-management) add an extra layer of protection: only admins explicitly assigned to the AU (or Global Admins / Privileged Role Admins) can modify the objects inside. Even tenant-level User Administrators **cannot** touch objects in a restricted management AU.

### Licensing

| Role | License |
|---|---|
| AU admin (scoped role) | **Entra ID P1** |
| AU member | **Free** (no premium license required) |

Reference: [Create or delete administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-manage) | [Manage administrative units with PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/manage-administrative-units?view=entra-powershell)

---

## Exam Traps

1. **UsageLocation is mandatory for licensing.** You cannot assign a license to a user without `UsageLocation` set. This applies to both direct and group-based licensing. If a dynamic group assigns licenses, every member must have `UsageLocation` or the assignment fails silently.

2. **Guest users and security questions.** Guests *can* be enabled for SSPR, but security questions may not be available to them depending on configuration. Admins can **never** use security questions for SSPR.

3. **Dynamic group changes are not instant.** After changing a dynamic rule or a user attribute, it may take minutes to hours for membership to recalculate. The exam may test your understanding that this is an **asynchronous** process.

4. **You cannot change a group from assigned to dynamic (or vice versa) after creation.** You must delete and recreate the group.

5. **Role-assignable groups are immutable in type.** `isAssignableToRole` must be set at creation and **cannot be changed later**. These groups must be Security or M365 type and are limited to 500 role-assignable groups per tenant.

6. **Group-based licensing requires Entra ID P1.** Direct assignment works on Free, but group-based licensing does not.

7. **Synced users cannot have their password reset in Entra unless password writeback is enabled.** Without writeback, SSPR only works for cloud-only users.

8. **B2B guests have `UserType = Guest` by default.** You can change this to Member, but the default is always Guest. Do not confuse *source* (External Entra ID, Microsoft Account, etc.) with *UserType*.

9. **M365 groups cannot contain devices.** Only security groups support device members. Dynamic device membership is also security-group-only.

10. **Bulk operations time out after 1 hour.** If you have a very large CSV, split it into batches. The portal limits bulk create/delete/invite to files with up to 50,000 rows per operation.

11. **Administrative units require P1 for scoped admins**, but not for the members within the AU.

12. **Restricted management AUs protect objects even from tenant-level admins** (except Global Admin and Privileged Role Admin). A tenant-level User Administrator **cannot** modify objects in a restricted management AU.

13. **Combined registration is now the only path.** The legacy separate registration for MFA and SSPR was deprecated in September 2025. All tenants use the unified Authentication methods policy.

---

## Sources

- [AZ-104 Study Guide -- Skills Measured](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 Learning Path: Manage identities and governance](https://learn.microsoft.com/en-us/training/paths/az-104-manage-identities-governance/)
- [az ad user -- Azure CLI](https://learn.microsoft.com/en-us/cli/azure/ad/user?view=azure-cli-latest)
- [az ad group -- Azure CLI](https://learn.microsoft.com/en-us/cli/azure/ad/group?view=azure-cli-latest)
- [Manage users -- Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/manage-user?view=entra-powershell)
- [Manage groups -- Microsoft Entra PowerShell](https://learn.microsoft.com/en-us/powershell/entra-powershell/manage-groups?view=entra-powershell)
- [Dynamic membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
- [Create or edit dynamic groups](https://learn.microsoft.com/en-us/entra/identity/users/groups-create-rule)
- [Bulk create users](https://learn.microsoft.com/en-us/entra/identity/users/users-bulk-add)
- [Bulk operations service limitations](https://learn.microsoft.com/en-us/entra/fundamentals/bulk-operations-service-limitations)
- [What is group-based licensing?](https://learn.microsoft.com/en-us/entra/fundamentals/concept-group-based-licensing)
- [Assign licenses to a group](https://learn.microsoft.com/en-us/entra/identity/users/licensing-groups-assign)
- [Resolve group license problems](https://learn.microsoft.com/en-us/entra/fundamentals/licensing-groups-resolve-problems)
- [PowerShell examples for group-based licensing](https://learn.microsoft.com/en-us/entra/identity/users/licensing-powershell-graph-examples)
- [Microsoft Entra External ID (B2B)](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)
- [Add B2B collaboration users](https://learn.microsoft.com/en-us/entra/external-id/add-users-administrator)
- [B2B guest user properties](https://learn.microsoft.com/en-us/entra/external-id/user-properties)
- [B2B invitation redemption](https://learn.microsoft.com/en-us/entra/external-id/redemption-experience)
- [Configure external collaboration settings](https://learn.microsoft.com/en-us/entra/external-id/external-collaboration-settings-configure)
- [Enable SSPR](https://learn.microsoft.com/en-us/entra/identity/authentication/tutorial-enable-sspr)
- [Password writeback](https://learn.microsoft.com/en-us/entra/identity/authentication/tutorial-enable-sspr-writeback)
- [Combined registration for SSPR and MFA](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined)
- [Authentication methods policy](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage)
- [Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)
- [Create or delete administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-manage)
- [Restricted management administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-restricted-management)
