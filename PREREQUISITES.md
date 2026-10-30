# PowerClippy prerequisites

Work through this checklist **before** installing. Every item says how to check it. For why each role is needed, see [PowerClippy-Roles.md](PowerClippy-Roles.md); for how to grant them, see [PERMISSIONS-GUIDE.md](PERMISSIONS-GUIDE.md).

## 1. Tenant

- [ ] **Microsoft 365 tenant with Power Platform.** Check: you can open https://admin.powerplatform.microsoft.com.
- [ ] **Dataverse capacity for one Production environment** (about 1 GB). Check: Power Platform admin center → **Licensing → Capacity** shows free database capacity. (A Developer environment uses no capacity but isn't recommended for a tool you depend on.)
- [ ] **Purview auditing on** (needed only for the unused-apps report). Audit (Standard) is on by default. Check in Exchange Online PowerShell: `Get-AdminAuditLogConfig | Format-List UnifiedAuditLogIngestionEnabled` returns `True`.
- [ ] **Power Platform apps allowed in Teams / Microsoft 365** (to use PowerClippy in Microsoft 365 Copilot). Check: Teams admin center → **Teams apps → Manage apps** doesn't block apps built with Power Platform.

## 2. The person who will use PowerClippy

- [ ] **License to chat in Microsoft 365 Copilot:** a Microsoft 365 Copilot license, or (without one) tenant Copilot Studio billing: prepaid Copilot Credits or pay-as-you-go linked to the environment. A Copilot Studio user license is for makers, not for people who only chat with the agent. Check: Microsoft 365 admin center → **Users → Active users → (user) → Licenses**, or run `setup/Grant-PowerClippyRoles.ps1 -Area License`.
- [ ] **Power Platform admin rights:** Power Platform Administrator, Dynamics 365 Administrator or Global Administrator (active, or PIM-eligible and activated before use).
- [ ] **System Administrator** security role in each Dataverse environment you want PowerClippy to read.
- [ ] **Audit log search rights:** Purview *Audit Reader* or *Audit Manager*, or Entra *Global Administrator* / *Compliance Administrator*.
- [ ] **Member of the access group** (SG-PowerClippy-Admins, created during installation).

One command checks all of the above (read-only). It reads your tenant ID and admin account from `setup/powerclippy.config.json`, so create that first (`Copy-Item setup/powerclippy.config.example.json setup/powerclippy.config.json`, then fill in `TenantId`, `AdminUpn`, `GroupId` and `EnvironmentUrls`), or pass `-TenantId` and `-UserUpn`:

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Grant-PowerClippyRoles.ps1
```

## 3. The person installing it (once)

| Needed for | Role |
|---|---|
| Creating the access group | Groups Administrator (or User Administrator) |
| Creating the environment and binding the group | Power Platform Administrator |
| Creating the *PowerClippy Audit Search* app registration | Cloud Application Administrator or Application Administrator |
| Granting the per-user Graph consent | Privileged Role Administrator or Global Administrator |
| Importing, pushing and publishing the agent | System Administrator in the PowerClippy environment |
| `Grant-PowerClippyRoles.ps1 -Apply` (Entra role / Purview role group) | Privileged Role Administrator or Global Administrator / Purview Organization Management |

A Global Administrator who is also Power Platform Administrator can do all of it.

## 4. Installing PC (Windows)

| Tool | Version tested | How to get it | Check |
|---|---|---|---|
| Windows PowerShell | 5.1 | Built into Windows | `$PSVersionTable.PSVersion` |
| Power Platform CLI (`pac`) | 2.14.1 | `winget install Microsoft.PowerAppsCLI` or Power Platform Tools for VS Code | `pac help` |
| `pac` signed in to the tenant | | `pac auth create --environment <url>` | `pac auth who` |
| Python 3 | 3.x | python.org or `winget install Python.Python.3.12` | `python --version` |
| ExchangeOnlineManagement (Purview role check only) | 3.10.1 | `Install-Module ExchangeOnlineManagement -Scope CurrentUser` | `Get-Module -ListAvailable ExchangeOnlineManagement` |

`Configure-AuditSearch.ps1` installs `paconn` itself into `%LOCALAPPDATA%\PowerClippy\paconn-venv`. No Microsoft Graph PowerShell module is needed.

**Keep the working folder path short** (for example `C:\pc`). `pac copilot clone` and `push` fail on paths over 260 characters, and a failed push can delete tools in the cloud.

**Browser:** a normal browser for the device-code sign-ins (https://login.microsoft.com/device) and for Copilot Studio.

## 5. Decisions to make first

| Decision | Default in this project |
|---|---|
| Environment name, region, URL | e.g. PowerClippy Admin, United States, `contoso-powerclippy` |
| Access group name | Your choice (the docs call it SG-PowerClippy-Admins) |
| Who uses PowerClippy | One administrator (add more later, see the FAQ) |
| Admin roles: permanent or PIM | Permanent today; PIM recommended (needs Entra ID P2 or Entra ID Governance) |
| Publisher prefix / schema name | `mono` / `mono_powerclippy` (keep these unless you rename every reference) |
