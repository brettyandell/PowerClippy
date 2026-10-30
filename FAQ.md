# PowerClippy FAQ

## General

**What is PowerClippy?**
A private Copilot Studio agent that audits a Microsoft 365 / Power Platform tenant: environments, capacity, DLP, security groups, agents, admin roles, connections, flows, apps and unused apps. It runs inside Microsoft 365 Copilot, Teams and Copilot Studio.

**Can PowerClippy change anything in my tenant?**
No. Every tool is a read operation, write operations on the admin connectors aren't exposed, the Graph tool is limited to GET, and the instructions refuse change requests. `make a change` only records a *planned* change; you make it yourself.

**Whose permissions does it use?**
Yours. Every tool runs with your sign-in (end-user credentials). PowerClippy has no service account, so it can't see anything you can't.

**Who can use it?**
Only members of SG-PowerClippy-Admins. The agent's access control and the environment's security group both use that group, and sign-in is always required.

**Where is the data stored?**
PowerClippy reads live data on each request and doesn't keep a copy. Conversation transcripts follow Copilot Studio's normal storage in the PowerClippy environment's Dataverse.

**How is PowerClippy different from the CoE Starter Kit?**
The CoE kit copies inventory into Dataverse on a schedule with flows. PowerClippy queries live APIs when you ask, needs no sync flows, and answers in chat. Its unused-apps report uses the same Purview audit source the kit uses (see `CoE-AuditLog-ReverseEngineering.md`).

## Access and installation

**I can't find PowerClippy in Microsoft 365 Copilot.**
Check that you're in SG-PowerClippy-Admins, that the **Microsoft 365 and Microsoft Teams** channel is added, and that you selected **Add** from *See agent in Microsoft 365*. Then look under **Agents** in the Copilot sidebar. New agents can take a few minutes to appear in the desktop app; restart it.

**The "See agent" link says "We couldn't find this agent".**
Adding or re-adding the channel issues a new Microsoft 365 title ID, and old links stop working. Open the link from the channel page again, or read the current ID from the bot record (`applicationmanifestinformation.microsoft365.titleId`).

**What if someone uses PowerClippy with an account that isn't an admin?**
PowerClippy runs every tool with the account they connect. When that account is refused, or returns nothing where an admin would see data, PowerClippy says which role is missing and asks them to reconnect with an account that has it: **Connect** / **Set up connection** on the card (or the connection management page) → **Create new connection** or **Reconnect** → **Use another account** in the sign-in window. They can check at any time by saying `check my access`. The full tenant audit runs this check first. They still have to be in SG-PowerClippy-Admins to open the agent at all.

**Can I use my everyday account for chat and my admin account for the tools?**
Yes. The agent uses your Microsoft 365 sign-in, but each tool connection can be signed in with a different account. The admin account needs the PowerClippy roles, and its own consent for the Audit Search connector (`Configure-AuditSearch.ps1 -ConsentUserUpn <admin account> -Step All`).

**Do I need to share the agent to use it?**
No. The owner can use it without sharing. Don't use *Show to my teammates* or *Show to everyone in my org*.

**Can I install it in another tenant?**
Yes. Import `dist/PowerClippy.zip`, then run `Configure-AuditSearch.ps1` with the new tenant's values (the package has no secret and the old client ID), grant roles, and add the Microsoft 365 channel. See [INSTALLATION.md](INSTALLATION.md).

**Why isn't the store listing (description, icon) in the package?**
It belongs to the Microsoft 365 / Teams channel, which Copilot Studio stores outside the solution. Set it under **Channels → Microsoft 365 and Microsoft Teams → Edit details** after import.

**Can more than one person use it?**
Yes. For each person: add them to the group, run `Grant-PowerClippyRoles.ps1 -UserUpn <upn> -Apply`, and run `Configure-AuditSearch.ps1 -ConsentUserUpn <upn> -Step All` for their Graph consent (this adds a new client secret, so delete old ones). Each person signs in to the connections themselves.

## Permissions

**Which roles do I need?**
Member of SG-PowerClippy-Admins; Power Platform Administrator (or Dynamics 365 Administrator or Global Administrator); System Administrator in each Dataverse environment; audit search rights (Audit Reader, Audit Manager, Global Administrator or Compliance Administrator); per-user Graph consent; a Copilot license. Full list: [PowerClippy-Roles.md](PowerClippy-Roles.md). How to grant: [PERMISSIONS-GUIDE.md](PERMISSIONS-GUIDE.md).

**Does `Grant-PowerClippyRoles.ps1` change anything?**
Not unless you add `-Apply`. Even then it only adds missing roles and never removes any.

**I'm a Global Administrator, but the script said Audit Reader is MISSING.**
Global Administrator grants audit search through the Entra role, which doesn't appear as Purview role-group membership. The current script reports *Audit search via Entra role: OK* in the Entra area. Nothing more is needed.

**Can I use PIM?**
Yes, and it's recommended if you have **Microsoft Entra ID P2** (or Entra ID Governance / Entra Suite), which eligible assignments require. Make Power Platform Administrator eligible and activate it before chatting; the script recognizes eligible assignments.

**Why does the custom connector need `Group.Read.All`?**
To read security groups, members and owners for the Security groups audit. It's delegated, so it never exceeds your own access, and consent is for your account only.

## Using it

**Where do I start?**
Every new chat starts with a welcome and an access status card (in Microsoft 365 Copilot, select the **Start here** suggested prompt). Then pick from the menu, say `menu` or `settings`, or just ask.

**Why does a new chat ask me to connect right away?**
The access check at the start of each chat uses one tool, Power Platform for Admins V2. If you've never connected it, or your sign-in expired, Copilot shows a Connect card once; after that the check is silent. Other connections still ask only when you first use them.

**The status card says Dataverse and Purview are "checked on use". Why not now?**
Checking them means running a real Dataverse query or starting a Purview audit search. PowerClippy checks them the first time you ask a question that needs them, and asks you to reconnect if they're refused.

**Why are some answers empty?**
Usually your roles: a PIM role isn't active, or you lack System Administrator in that environment. Run `Grant-PowerClippyRoles.ps1`.

**Why doesn't a new agent or app show up?**
The Power Platform inventory refreshes about every 20 minutes and lists published agents only.

**How far back does flow run history go?**
28 days by default, for solution-aware flows.

**The unused-apps report refused to answer.**
That's deliberate when no audit records came back (auditing off, search failed, or no audit rights). It won't call every app unused just because data is missing. Run `unused apps`, wait a few minutes, then `check unused apps`.

**It asked me which API version to use.**
A topic or tool isn't passing `api-version`. It should always use `2024-10-01` (Power Platform admin connectors) or `2016-11-01` (Power Apps / Power Automate for Admins). Report it; the fix is to bind the value and republish.

**How does it answer general questions?**
From eight Microsoft Learn sites (Power Platform admin and guidance, Power Apps, Power Automate, Copilot Studio, connectors, AI Builder, Power Pages), with a link. If neither the tools nor Learn settle it, it says it doesn't know. Web browsing is off.

## Maintenance

**When does the client secret expire?**
180 days after `Configure-AuditSearch.ps1` created it (the script prints the date). Re-run `Configure-AuditSearch.ps1 -Step All` before then, and delete old secrets in the app registration.

**How do I back up PowerClippy?**
Export the solution (`setup/Export-CleanSolution.ps1`, or `pac solution export` into the git-ignored `local/` folder) and keep the project folder, which contains the YAML source, scripts, connector definition and docs.

**How do I change a topic or tool?**
Edit the YAML in a short-path copy, then `pac copilot pull`, `push`, `publish`, add new components to the solution, and copy the folder back. Rules and pitfalls are in the README (*Making changes*) and the build guide.

**Something broke after a push.**
See the README troubleshooting table and the build guide's recovery section. The usual causes are a long path, a missing `pull`, or an empty YAML file.
