# PowerClippy: audit topics and tool setup

PowerClippy now has deterministic, read-only audit logic: a main menu, a **Full Tenant Audit** that loops through the areas you choose, nine area topics that apply fixed risk rules, and a change-request guardrail. Everything is live in **PowerClippy Admin** (`https://<powerclippy-env>.crm.dynamics.com/`) and packaged in `dist/PowerClippy.zip`.

## What's in the agent

| Topic | Triggered by (examples) | What it does |
|---|---|---|
| PowerClippy Menu | "menu", "help", "what can you do" | Lists everything below and routes to it |
| Full Tenant Audit | "full tenant audit", "governance audit" | Asks **Everything** or **Let me choose** (a pick-until-Done loop), runs each area in order, then sends one report with high / medium / low counts |
| Audit Area - Environments | "audit environments" | Lists every environment. 🟠 non-default environments with no security group; 🟢 production environments keeping backups for only 7 days |
| Audit Area - Capacity | "audit capacity" | Database, file and log use. 🔴 85% or more; 🟠 70% or more |
| Audit Area - Environment groups | "audit environment groups" | Groups and their environments. 🟢 production or sandbox environments in no group (no group rules apply) |
| Audit Area - DLP policies | "audit dlp policies" | Scope and connector counts per policy. 🔴 no policies, or policies that block nothing; 🟠 no policy covering all environments; 🟠 unauthenticated agent chat not blocked |
| Audit Area - Security groups | "audit security groups" | Entra security groups with member, user and guest counts (first 60), which environments each group gates, and owners of gating groups. Uses the Groups - * tools. 🟠 guests in a gating group · 🟢 empty groups, ownerless gating groups |
| Audit Area - Agents | "audit agents", "agent sharing review" | Every Copilot Studio agent in the tenant (Power Platform inventory). 🔴 shared with the entire tenant; 🔴 no sign-in; 🟠 tools on the maker's connection; 🟢 web search on |
| Audit Area - Admin roles | "who has admin roles" | Power Platform RBAC assignments per environment, split into environment-level roles (by role name) and app/flow sharing (resource-level). 🟠 more than 5 environment-level owner/admin assignments. Points to the Dataverse System Administrator question for full admins. |
| Audit Area - Connections | "audit connections" | Connections and connectors per environment. 🟢 expired connections |
| Audit Area - Flows | "audit flows", "how many flows are there" | Cloud flows per environment by state (on = `Published`, off = `Draft`, `Suspended`). Checks run history for up to 50 flows that are on in each environment. 🟠 suspended flows (with reason); 🟠 flows with failed runs; 🟢 flows that are on but have no runs in the retained history (28 days by default, solution-aware flows only) |
| Unused Apps - Start search | "unused apps", "apps not used in 90 days" | Asks for 30, 60, 90 or 180 days, then starts a read-only Purview audit search for `LaunchPowerApp`. **Needs the approvals in "Unused apps: your approval steps".** |
| Unused Apps - Results | "check unused apps" | If the search hasn't finished, reports its status. If it has, reads up to 20 pages of 500 records, collects distinct app IDs, and lists canvas apps from the inventory with no launches and created before the period. Apps created during the period are listed as too new to judge. **Refuses to report** when zero records came back, when no app IDs could be read, or when the search failed. Flags partial results (page cap, record limit, unreadable records). |
| Change Request Guardrail | "make a change", "unshare an agent" | States that PowerClippy is read-only, asks for the change, and logs it as *planned* only after you choose **confirm** |

Each area topic also works on its own; it only shows its section when it isn't running inside the full audit.

## Tools (all read-only, all set to end-user credentials)

| Tool | Connector · operation | Connection reference |
|---|---|---|
| Audit - List environments | Power Platform for Admins V2 · `ListEnvironmentsForUser` | `mono_powerclippy.shared_powerplatformadminv2.ppadminv2` |
| Audit - Tenant capacity | V2 · `GetTenantCapacityDetails` | same |
| Audit - List environment groups | V2 · `ListEnvironmentGroups` | same |
| Audit - List connections | V2 · `ListConnections` | same |
| Audit - List environment role assignments | V2 · `ListEnvironmentRoleAssignments` | same |
| Audit - List role definitions | V2 · `ListRoleDefinitions` | same |
| Audit - Query resource inventory | V2 · `QueryResources` | same |
| Audit - List cloud flows | V2 · `ListCloudFlows` | same |
| Audit - List flow runs | V2 · `ListFlowRuns` | same |
| Apps Admin - List apps / Get app / App permissions / List connections / Connection permissions / List custom connectors / Custom connector permissions | Power Apps for Admins · `Get-AdminApps` / `Get-AdminApp` / `Get-AdminAppRoleAssignment` / `Get-AdminConnections` / `Get-AdminConnectionRoleAssignment` / `Get-AdminConnectors` / `Get-AdminConnectorRoleAssignment` (api-version 2016-11-01; custom connectors 2017-05-01 / 2017-06-01) | `mono_powerclippy.shared_powerappsforadmins.paadmin` |
| Flow Admin - Flow owners / Flow user details | Power Automate for Admins · `Get-AdminFlowOwnerRole` / `Get-AdminFlowUserDetails` (read-only operations only; the connector's enable/disable/delete operations are deliberately not exposed) | `mono_powerclippy.shared_microsoftflowforadmins.flowadmin` |
| Audit Log - Start query / Get query status / List records | PowerClippy Audit Search (custom) · `CreateAuditLogQuery` / `GetAuditLogQuery` / `ListAuditLogRecords` | `mono_powerclippy.shared_new-5fpowerclippy-20audit-20search-5faf942d83cebc15d6.auditsearch` |
| Groups - List security groups / List group members / List group owners | PowerClippy Audit Search (custom) · `ListSecurityGroups` / `ListGroupMembers` / `ListGroupOwners` (delegated `Group.Read.All`, `GroupMember.Read.All`) | same |
| Audit - List DLP policies | Power Platform for Admins · `ListPoliciesV2` | `mono_powerclippy.shared_powerplatformforadmins.ppadmin` |
| Audit - Microsoft Graph GET (fallback only) | HTTP with Microsoft Entra ID (preauthorized) · `InvokeHttp` | `mono_powerclippy.shared_webcontents.graph` |
| Audit - Read Dataverse rows | Microsoft Dataverse · `ListRecordsWithOrganization` | `mono_powerclippy.shared_commondataserviceforapps.dataverse` |

No tool can write. The Graph tool is only ever called with `GET` from topics, and the agent instructions forbid other methods.

**Knowledge sources (Microsoft Learn, public websites):** `learn.microsoft.com/en-us/` power-apps, power-automate, microsoft-copilot-studio, connectors, ai-builder, power-pages (added in Copilot Studio) plus `learn.microsoft.com/power-platform/admin` and `/power-platform/guidance` (in `knowledge/`). The agent instructions tell PowerClippy to search these whenever a question can't be answered from tools or there is any doubt, to cite the page URL, and to say "I don't know" rather than guess if Learn doesn't settle it.

**Instructions:** `agent.mcs.yml` now has a "Which tool to use" section mapping every question type to a connector/tool, including the required `api-version` for each connector, and a "When you are not sure" section with the Learn fallback.

## Unused apps: setup

The *PowerClippy Audit Search* custom connector (definition in `connector/`) calls the Microsoft Graph **v1.0** Audit Search API (create query, get status, list records) **as the signed-in user**. To make it work:

1. **App registration, permissions, per-user consent and client secret:** run `setup/Configure-AuditSearch.ps1 -Step All` (README 6.4, docs/INSTALLATION.md Step 6). It requests the delegated permissions `AuditLogsQuery.Read.All`, `Group.Read.All`, `GroupMember.Read.All` and `User.Read`, and grants consent **for one user only**, never tenant-wide. Delegated means PowerClippy can only read what that user can read, and only while they're signed in.
2. **Audit search rights for the user:** Purview *Audit Reader* or *Audit Manager*, or Entra *Global Administrator* / *Compliance Administrator* (docs/PERMISSIONS-GUIDE.md section 4).
3. **Auditing must be on.** Purview Audit (Standard) is on by default. Microsoft's Power Apps activity-log page also asks you to turn on auditing for the Power Platform environments you want to monitor.
4. **Test:** say `unused apps`, pick **90 days**, sign in on the connection card, wait a few minutes, then say `check unused apps`.

Optional alternative to a client secret: a **managed identity** with a federated credential on the app registration (Power Apps → Custom connectors → PowerClippy Audit Search → Security → Managed identity). The scripts use a client secret.

**What the first live run will settle** (based on the docs, not yet observed):
- **Where the app ID lives.** The v1.0 docs describe `auditData` only as an abstract type. The topic reads `auditData.PropertyCollection`, as the CoE kit does on the beta API. If v1.0 shapes it differently, the topic reports "couldn't read an app ID" rather than a wrong answer.
- **Paging.** The topic passes the `$skiptoken` from `@odata.nextLink`. If that token comes back URL-encoded in a way Graph rejects, paging stops after page 1, and the report says it's partial.
- **Limits.** Graph allows at least 200 audit queries a day, 50 running at once, and 1,000,000 records per query. A query that hits the record cap still reports `succeeded`; the topic checks `isRecordCountLimitExceeded`.

**Known tooling issue:** `pac copilot push` now logs a path-too-long error while caching the custom connector's definition locally (`.mcs/.connectors-download.json`). Pushes still apply; confirm with `modifiedon` after each push.

## First run

See README section 7: open the agent's **Test** pane, say `menu`, and sign in on each **Connect** card (seven connection references; for **HTTP with Microsoft Entra ID** use `https://graph.microsoft.com` for both URLs). Tools use end-user credentials (`mode: Invoker`).

## Known limits

- **Dataverse tables can't be read by topics.** The Dataverse connector's output schema only resolves at run time, so the Admin roles topic reports Power Platform RBAC assignments, not Dataverse security roles. The *Audit - Read Dataverse rows* tool description carries a ready-made FetchXML for System Administrators, which the orchestrator uses. Ask follow-ups such as "who has System Administrator in Production environment?" or "list deployment pipelines in <environment>": the AI orchestrator uses the Dataverse tool for those.
- **Graph permissions.** Group reads now go through our own PowerClippy Audit Search connector with delegated `Group.Read.All` and `GroupMember.Read.All`, consented for the admin only. The preauthorized HTTP connector's Graph permissions are set by Microsoft and aren't documented, so it is a fallback only.
- **Agent inventory lag.** The Power Platform inventory refreshes about every 20 minutes and shows published agent versions only.
- **Speed.** Admin roles and Connections call each environment one at a time; Security groups calls Graph once per group (capped at 60), plus once more for the owners of each group that gates an environment.
- **You need admin rights.** Results reflect what your account can see. Every required role is listed in [docs/PowerClippy-Roles.md](docs/PowerClippy-Roles.md); check or apply them with `setup/Grant-PowerClippyRoles.ps1` (check-only unless `-Apply`).

## Editing PowerClippy from now on

**Push from a short path.** `pac copilot push` fails, and can even **delete tools in the cloud**, when the workspace path is long: Windows' 260-character limit is hit for the longest tool filenames once pac resolves the app-redirected path (for example under an app-redirected AppData folder). A crashed push once deleted three tools in the cloud; pushing again from a short path recreated them. Keep the working copy at a short path such as `C:\pc\PowerClippy` and copy it back to the project folder afterwards.

`PowerClippy/` holds the agent's source. Its `.mcs/` sync folder isn't committed, so work in a fresh `pac copilot clone` of your agent at a short path and copy the source into it. `pac copilot pack` no longer works on it, because pack rejects tools and connection references. Use this loop instead:

```bash
pac copilot push --project-dir ./PowerClippy
```

```bash
pac copilot publish --bot mono_powerclippy --environment https://<powerclippy-env>.crm.dynamics.com/
```

Publish with the schema name (`mono_powerclippy`): publishing by bot ID hits a CLI bug. Read publish errors from the bot record's `synchronizationstatus.lastFinishedPublishOperation.diagnosticDetails`.

To refresh `dist/PowerClippy.zip`, run:

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Remove-TestCasesFromSolution.ps1 -Apply
```
```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Export-CleanSolution.ps1
```
```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Test-NoTenantData.ps1
```
(see *Publishing to GitHub*). A plain `pac solution export` to `dist/PowerClippy.zip` would include your tenant's connector IDs and test cases.

New topics or tools pushed later land outside the solution. Add each one before exporting, using `--componentType botcomponent`; the numeric type code fails:

```bash
pac solution add-solution-component --environment https://<powerclippy-env>.crm.dynamics.com/ --solutionUniqueName PowerClippy --component <botcomponent id> --componentType botcomponent
```

Lessons from this build: a new connection reference must exist in Dataverse before `push` (add it to the solution's `customizations.xml` and import); push fails with "does not exist in connectionreference entity" otherwise. Inside double-quoted YAML strings, a Power Fx `", "` ends the string; use single-quoted YAML. In a `Foreach` over a list of text values, the loop variable is plain text (`Topic.Area`, not `Topic.Area.Value`).

More lessons:
- **`api-version` on admin-connector tools:** every Power Platform for Admins / V2 operation has a required `api-version` query parameter. If a topic doesn't bind it, the agent asks the user "which API version?". Every `BeginDialog` to those tools now binds `api-version: 2024-10-01` (the connector's documented default), and each tool's description says never to ask for it. Keep doing this for any new tool on those connectors.
- **Push conflicts:** solution imports and `add-solution-component` count as cloud changes. Run `pac copilot pull --project-dir ./PowerClippy` before the next push, or push fails with "The agent was changed in the cloud".
- **Missed edits:** push can skip an edited file. After pushing, check `modifiedon` on the bot components; if a file was missed, make a small real edit to it and push again.
- **Flow state labels:** the Power Platform API reports a flow that's on as `Published` (Draft / Published / Suspended), not Dataverse's `Activated`.
- **Run status spellings:** two exist, Success (Dataverse table) and Succeeded (API enum). The topic matches both, plus `Failed` / `FailedWithRetry`.
