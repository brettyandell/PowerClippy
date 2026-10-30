# PowerClippy permissions setup guide

Step-by-step instructions for every permission PowerClippy needs, with both the **portal** way and the **script** way. The reference list (what each role is for, and the current state) is [PowerClippy-Roles.md](PowerClippy-Roles.md).

> **How permissions work in PowerClippy.** Every tool runs **as the signed-in user** (end-user credentials). PowerClippy has no service account and no permissions of its own: it sees exactly what the user can see. Missing roles don't cause errors so much as **empty or partial answers**.

---

## Quick path: the script

From the project folder:

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Grant-PowerClippyRoles.ps1
```

This **only checks** and prints a table (OK / MISSING / REPORT). Nothing changes. To add only what's missing (it never removes anything):

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Grant-PowerClippyRoles.ps1 -Apply
```

| Parameter | Use |
|---|---|
| `-Area` | Limit to `Entra`, `Group`, `License`, `Dataverse`, `Purview` (comma list allowed, e.g. `-Area Entra,Group`) |
| `-UserUpn` | The user to check (default: `AdminUpn` in `setup/powerclippy.config.json`) |
| `-EnvironmentUrls` | Dataverse environments to check for System Administrator (default: `EnvironmentUrls` in the config file) |
| `-GroupId` | Object ID of the access group (default: `GroupId` in the config file) |
| `-TenantId` | Tenant ID (default: `TenantId` in the config file) |

Sign-ins: one **device code** for Microsoft Graph (Entra, Group, License), your existing `pac auth` profile (Dataverse), and a **browser sign-in** for Purview.

What `-Apply` does per area:

| Area | Check | `-Apply` adds |
|---|---|---|
| Entra | Power Platform Administrator, Dynamics 365 Administrator or Global Administrator (active or PIM-eligible); also reports Global/Compliance Administrator for audit search | Power Platform Administrator, **permanent active** (consider PIM instead, below) |
| Group | Membership of the access group | The user as a member |
| License | Lists assigned license SKUs | Nothing (report only) |
| Dataverse | System Administrator in each environment | `pac admin assign-user --role "System Administrator"` |
| Purview | Membership of Audit Reader, Audit Manager or Organization Management | The user to **Audit Reader** (takes up to an hour) |

---

## 1. Access group membership

**Why:** PowerClippy's access control is "members of SG-PowerClippy-Admins", and the PowerClippy environment is locked to the same group.

**Create the group (installation only):** Entra admin center → **Groups → All groups → New group**
- Group type **Security**, name `SG-PowerClippy-Admins`, membership type **Assigned**.
- Microsoft Entra roles can be assigned to the group: **No**.
- Owners and members: the administrator.

**Add a member:** Entra admin center → **Groups → SG-PowerClippy-Admins → Members → Add members**, or `Grant-PowerClippyRoles.ps1 -Area Group -Apply`.

**Check:** the group's **Members** page, or `-Area Group`.

## 2. Power Platform admin rights (Entra role)

**Why:** the Power Platform for Admins (V2), Power Apps for Admins and Power Automate for Admins connectors return tenant-wide data only to admins.

**Portal:** Entra admin center → **Roles & admins → Power Platform Administrator → Add assignments** → select the user.

**Recommended: make it PIM-eligible instead of permanent.** PIM eligible assignments need **Microsoft Entra ID P2** (or Entra ID Governance / Entra Suite); on Free or P1 tenants roles can only be assigned as active. Entra admin center → **ID Governance → Privileged Identity Management → Microsoft Entra roles → Roles → Power Platform Administrator → Add assignments** → role *Power Platform Administrator* → assignment type **Eligible**. The user then activates the role (**My roles → Activate**) before using PowerClippy.

**Already a Global Administrator?** Nothing to do: Global Administrator includes these rights.

**Script:** `Grant-PowerClippyRoles.ps1 -Area Entra -Apply` (adds a permanent active assignment).

## 3. Dataverse System Administrator (per environment)

**Why:** reading Dataverse tables (who has System Administrator, deployment pipelines, solutions) and some flow data.

**Portal:** Power Platform admin center → **Environments → (environment) → Settings → Users + permissions → Users** → select the user → **Manage security roles** → tick **System Administrator** → Save.

**Self-elevate (Power Platform Administrators):** Power Platform admin center → **Environments → (environment)** → **Membership → Add me**, or:

```bash
pac admin self-elevate --environment https://<org>.crm.dynamics.com/
```

**Assign to someone:**

```bash
pac admin assign-user --environment https://<org>.crm.dynamics.com/ --user <upn> --role "System Administrator"
```

**Script:** `Grant-PowerClippyRoles.ps1 -Area Dataverse -Apply` (all environments in `-EnvironmentUrls`).

**Note:** an environment with a security group only accepts users in that group.

## 4. Audit log search (Purview)

**Why:** the *Audit Log - * tools* and the unused-apps report use the Microsoft Graph Audit Search API, which needs audit search rights.

Any **one** of these is enough:
- Purview role group **Audit Reader** (search only, least privilege) or **Audit Manager**.
- Entra role **Global Administrator** or **Compliance Administrator** (these don't appear as role-group members).

**Portal:** Microsoft Purview portal → **Settings → Roles and scopes → Role groups → Audit Reader → Edit → Choose users** → add the user → Save.

**Script:** `Grant-PowerClippyRoles.ps1 -Area Purview -Apply`. It needs the ExchangeOnlineManagement module; if PowerShell can't install to `Documents`, run:

```powershell
Save-Module ExchangeOnlineManagement -Repository PSGallery -Path "$env:LOCALAPPDATA\PowerClippy\psmodules"
```

The script finds it there. Changes can take **up to an hour** to apply.

## 5. Delegated Microsoft Graph consent (custom connector)

**Why:** the *PowerClippy Audit Search* connector calls Microsoft Graph as the user for audit searches and group reads.

| Permission (delegated) | Used for |
|---|---|
| `AuditLogsQuery.Read.All` | Purview audit search (unused apps) |
| `Group.Read.All` | Listing security groups and owners |
| `GroupMember.Read.All` | Listing group members |
| `User.Read` | Sign-in |

Consent is granted **for one user only** (`consentType: Principal`), never tenant-wide. Delegated permissions never exceed the user's own rights.

**Script (recommended):**

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Configure-AuditSearch.ps1 -Step All
```

This creates or updates the app registration, requests the permissions, grants per-user consent, adds a client secret (held in memory only) and updates the connector. It asks for two device-code sign-ins.

**For another user:** add `-ConsentUserUpn <upn>`. Each run adds a new client secret; delete old ones afterwards (Entra admin center → **App registrations → PowerClippy Audit Search → Certificates & secrets**).

**Check:** Entra admin center → **Enterprise applications → PowerClippy Audit Search → Permissions → User consent** lists the four permissions for the user.

## 6. Licensing

**Portal:** Microsoft 365 admin center → **Users → Active users → (user) → Licenses and apps** → assign **Microsoft 365 Copilot**. Without it, usage is billed to the tenant's Copilot Studio capacity (prepaid Copilot Credits or pay-as-you-go linked to the environment); a Copilot Studio user license is only needed by makers.

**Check:** `Grant-PowerClippyRoles.ps1 -Area License` lists the SKUs.

## 7. Connections (first chat)

Not a role, but the last permission step: the first time each tool runs, PowerClippy shows a **Connect** card. Sign in with the account that has the roles above. That can be a separate admin account: choose **Use another account** in the sign-in window. If the connected account lacks a role, PowerClippy says so and asks you to reconnect; `check my access` tests it at any time (see the README, *Connecting with the right account*). For **HTTP with Microsoft Entra ID**, use `https://graph.microsoft.com` for both URL fields.

---

## Least-privilege setup (recommended for production)

| Need | Least privilege |
|---|---|
| Tenant-wide Power Platform reads | Power Platform Administrator, **PIM-eligible** (Entra ID P2) |
| Dataverse reads | System Administrator in each environment (self-elevate when needed) |
| Audit search | Purview **Audit Reader** |
| Agent access | Member of SG-PowerClippy-Admins |
| Global Administrator | Emergency use only, through PIM |

## Removing access

1. Remove the user from SG-PowerClippy-Admins.
2. Remove their per-user consent: Entra admin center → **Enterprise applications → PowerClippy Audit Search → Permissions**, or delete the user's `oauth2PermissionGrant`.
3. Remove admin roles they no longer need (Entra, Dataverse, Purview).
4. Their connections stop working once their roles and consent are gone.
