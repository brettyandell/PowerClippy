# PowerClippy: required roles and permissions

Keep this file current whenever a tool, topic or connector is added. PowerClippy runs every tool **as the signed-in user** (connection mode Invoker), so what it can read is exactly what that user can read. It has no service account and no standing permissions of its own.

Check or apply everything with [`setup/Grant-PowerClippyRoles.ps1`](../setup/Grant-PowerClippyRoles.ps1):

```powershell
./setup/Grant-PowerClippyRoles.ps1                        # check only, changes nothing
./setup/Grant-PowerClippyRoles.ps1 -Apply                 # add whatever is missing (never removes)
./setup/Grant-PowerClippyRoles.ps1 -Area Dataverse,Purview -Apply
```

## 1. Roles the operator needs to use PowerClippy

| # | Role / permission | Where | Needed for | Script area |
|---|---|---|---|---|
| 1 | **Member of SG-PowerClippy-Admins** (`<group-id>`) | Microsoft Entra ID | Getting into the agent at all (agent access control = group membership) and into the PowerClippy Admin environment (security group) | `Group` |
| 2 | **Power Platform Administrator** (or Dynamics 365 Administrator, or Global Administrator) | Entra directory role | Tenant-wide reads with the Power Platform for Admins V2, Power Apps for Admins and Power Automate for Admins tools: environments, capacity, DLP, environment groups, role assignments, all apps, flows and connections. Without it these tools return only resources the user owns. PIM-eligible is fine: activate before chatting. | `Entra` |
| 3 | **System Administrator** security role in each Dataverse environment | Dataverse, per environment | *Audit - Read Dataverse rows* (System Administrator lists, pipelines, solutions) and Dataverse-backed flow lists. A Power Platform Administrator can self-elevate. | `Dataverse` |
| 4 | **Audit Reader** role group (or Audit Manager), **or** the Entra **Global Administrator** / **Compliance Administrator** role | Microsoft Purview / Entra | *Audit Log - * tools* and the unused-apps audit search (Microsoft Graph Audit Search API). | `Purview`, `Entra` |
| 5 | **Delegated Microsoft Graph consent, for this user only**: `AuditLogsQuery.Read.All`, `Group.Read.All`, `GroupMember.Read.All`, `User.Read` | Entra app *PowerClippy Audit Search* | *Audit Log - * and Groups - * tools* (custom connector). Applied by [`setup/Configure-AuditSearch.ps1`](../setup/Configure-AuditSearch.ps1), not by the roles script. | (Configure-AuditSearch) |
| 6 | **Copilot use rights** (a Microsoft 365 Copilot license, or tenant Copilot Studio billing: prepaid Copilot Credits or pay-as-you-go; a Copilot Studio user license is only for makers) | Licensing | Chatting with the published agent in Microsoft 365 Copilot. | `License` (report only) |

Notes:
- **Read-only by design.** Each role above grants write access in its own portal, but PowerClippy's tools are all GET/read operations and its instructions refuse changes. The Graph tool is restricted to GET.
- **Delegated permissions are capped by the user's roles.** `Group.Read.All` lets the connector read groups only because the user can. Granting consent does not give anyone new access.
- **Microsoft Graph via *Audit - Microsoft Graph GET*** (HTTP with Microsoft Entra ID, preauthorized) is a fallback only. Its Graph permissions are set by Microsoft and aren't documented, so group reads go through our own connector (row 5).

## 2. Roles needed once, to build or rebuild PowerClippy

| Role | Needed for |
|---|---|
| **Cloud Application Administrator** or **Application Administrator** (Privileged Role Administrator or Global Administrator for the consent grant) | Creating the *PowerClippy Audit Search* app registration, client secret and the per-user delegated consent (`Configure-AuditSearch.ps1`) |
| **Groups Administrator** (or User Administrator) | Creating SG-PowerClippy-Admins and adding members |
| **Power Platform Administrator** | Creating the PowerClippy Admin environment, binding its security group, Managed Environment and sharing settings |
| **System Administrator or System Customizer** in PowerClippy Admin | Pushing and publishing the agent (`pac copilot push / publish`), solution components, the custom connector |
| **Privileged Role Administrator** or Global Administrator | Running `Grant-PowerClippyRoles.ps1 -Apply` for the Entra role |
| **Purview Organization Management** | Running `-Apply` for the Audit Reader role group |

## 3. What each tool needs

| Tool group | Connector | Role(s) from section 1 |
|---|---|---|
| Audit - * (environments, capacity, groups of environments, role assignments, flows, inventory) | Power Platform for Admins V2 | 2 |
| Audit - List DLP policies | Power Platform for Admins | 2 |
| Audit - Read Dataverse rows | Microsoft Dataverse | 3 |
| Audit - Microsoft Graph GET | HTTP with Microsoft Entra ID (preauthorized) | 2, plus Microsoft-managed Graph permissions |
| Apps Admin - * | Power Apps for Admins | 2 |
| Flow Admin - * | Power Automate for Admins | 2 |
| Audit Log - * | PowerClippy Audit Search (custom) | 4, 5 |
| Groups - * | PowerClippy Audit Search (custom) | 5 |

## 4. Checking your current state

Run `./setup/Grant-PowerClippyRoles.ps1` (read-only) to see which requirements are met for your administrator.

- Global Administrator counts for Power Platform admin rights and for audit search, without appearing in the Purview role groups.
- The Purview check needs the ExchangeOnlineManagement module. If PowerShell can't install it to `Documents`, `Save-Module` it to `%LOCALAPPDATA%\PowerClippy\psmodules`; the script looks there too. Sign-in opens your browser (`-DisableWAM`), because the Windows sign-in broker needs a window that a script host doesn't have.
- `-Area` accepts comma lists (for example `-Area Entra,Group,License`), including with `powershell -File`.
- **Least privilege:** hold Global Administrator through PIM only, and give the operator Power Platform Administrator plus Audit Reader; the script checks and applies both. PIM eligible assignments need **Microsoft Entra ID P2** (or Entra ID Governance / Entra Suite).

## 5. Adding a new user later

The agent is meant for one administrator. To add another, do all of these, in order:
1. Run `Grant-PowerClippyRoles.ps1 -UserUpn <upn> -Apply`.
2. Run `Configure-AuditSearch.ps1 -ConsentUserUpn <upn> -Step All` to add their per-user consent. This also adds a new client secret and puts it into the connector; delete the older secrets afterwards. (Don't use `-Step Graph` alone: its secret is kept only in memory and would be left unused.)
3. They sign in to each connection the first time they chat.
