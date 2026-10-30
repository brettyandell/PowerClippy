# Installing PowerClippy

A step-by-step runbook with a check after every step. Finish [PREREQUISITES.md](PREREQUISITES.md) first. Plan on about an hour, most of it waiting for the environment to be created.

Replace these placeholders throughout, and put the same values in `setup/powerclippy.config.json` (copy `setup/powerclippy.config.example.json`) so the scripts pick them up:

| Placeholder | Example |
|---|---|
| `<tenant-id>` | `00000000-0000-0000-0000-000000000000` |
| `<admin-upn>` | `admin@contoso.onmicrosoft.com` |
| `<group-id>` | Object ID of your access group |
| `<domain>` | `contoso-powerclippy` (environment URL `https://<domain>.crm.dynamics.com/`) |
| `<environment-id>` | From `pac admin list` |

Run all commands from the PowerClippy project folder, kept at a **short path** (for example `C:\pc\PowerClippy`).

---

## Step 0: Create your settings file

The setup scripts read their defaults from `setup/powerclippy.config.json`, which is git-ignored. Create it from the template and fill in values as you collect them during the steps below:

```powershell
Copy-Item setup/powerclippy.config.example.json setup/powerclippy.config.json
```

Any script parameter (for example `-TenantId`) overrides the file. If a setting is missing, the script stops and names it.

## Path A: install from the solution package (recommended)

### Step 1: Sign in the Power Platform CLI

```bash
pac auth create --tenant <tenant-id>
```

**Check:** `pac auth who` shows your account.

### Step 2: Create the access group

Entra admin center → **Groups → New group**: type **Security**, name `SG-PowerClippy-Admins`, membership **Assigned**, role-assignable **No**. Add the administrator as **owner and member**. Copy the group's **Object ID** (`<group-id>`).

**Check:** the group shows 1 owner and 1 member.

### Step 3: Create the environment

```bash
pac admin create --name "PowerClippy Admin" --type Production --region unitedstates --currency USD --language English --domain <domain> --security-group-id <group-id>
```

This takes about 5 minutes.

**Check:** `pac admin list` shows *PowerClippy Admin*. Note its environment ID (`<environment-id>`).

### Step 4: Import the solution

```bash
pac solution import --path ./dist/PowerClippy.zip --environment https://<domain>.crm.dynamics.com/ --publish-changes
```

**Check:** the output ends with *"Solution Imported successfully"* and *"Published All Customizations"*.

### Step 5: Publish the agent

`--publish-changes` doesn't publish the agent itself:

```bash
pac copilot publish --bot mono_powerclippy --environment https://<domain>.crm.dynamics.com/
```

**Check:** *"Published successfully!"*. In Copilot Studio, the agent's **Channels** page shows a new *Published* time.

### Step 6: Configure the custom connector

The imported connector has placeholder (all-zero) tenant and client IDs and **no client secret**; this step fills them in. Find the imported connector's API name and redirect URL: Power Apps → the environment → **Custom connectors → PowerClippy Audit Search → Security** (redirect URL), or query the `connector` table (`connectorinternalid` is the API name).

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Configure-AuditSearch.ps1 -Step All -TenantId <tenant-id> -ConsentUserUpn <admin-upn> -EnvironmentId <environment-id> -ConnectorApiName <api-name> -RedirectUri <redirect-url>
```

Two device-code sign-ins: Microsoft Graph first, then `paconn`.

**Check:** the output ends with `CONSENT GRANTED`/`CONSENT UPDATED`, `CONNECTOR UPDATED with client ID and secret` and `DONE`.

### Step 7: Grant the roles

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Grant-PowerClippyRoles.ps1 -UserUpn <admin-upn> -TenantId <tenant-id> -GroupId <group-id> -EnvironmentUrls https://<domain>.crm.dynamics.com/
```

Review the table. Then add what's missing:

```bash
powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Grant-PowerClippyRoles.ps1 -UserUpn <admin-upn> -TenantId <tenant-id> -GroupId <group-id> -EnvironmentUrls https://<domain>.crm.dynamics.com/ -Apply
```

Add your other Dataverse environments to `-EnvironmentUrls` (comma separated) so PowerClippy can read them. See [PERMISSIONS-GUIDE.md](PERMISSIONS-GUIDE.md) for the portal alternative and PIM.

**Check:** every row is OK, except *License* (report only) and *Audit search via Entra role*, which can show NONE when the Purview row is OK. If Purview shows SKIPPED, install ExchangeOnlineManagement (see [PREREQUISITES.md](PREREQUISITES.md)) and re-run.

### Step 8: Lock down the agent

In Copilot Studio → PowerClippy:
1. **Settings → Security → Authentication:** *Authenticate with Microsoft*, sign-in required.
2. **Settings → Security → Access** (or *Share*): only **SG-PowerClippy-Admins**. Don't share with anyone else.

**Check:** the agent's settings match `PowerClippy/settings.mcs.yml` (`authenticationMode: Integrated`, `authenticationTrigger: Always`, `accessControlPolicy: GroupMembership`).

### Step 9: First run and connections

1. Copilot Studio → PowerClippy → **Test**.
2. Type `menu`, then **Full tenant audit → Everything**.
3. Sign in on each **Connect** card. For **HTTP with Microsoft Entra ID** enter `https://graph.microsoft.com` in both URL fields.
4. Make sure tools use **end-user credentials** (they're defined as `mode: Invoker`).

**Check:** the full audit returns a report with High / Medium / Low counts and never asks for an API version.

### Step 10: Add it to Microsoft 365 Copilot

The channel and store listing **aren't part of the solution package**, so set them up in each tenant:
1. Copilot Studio → PowerClippy → **Channels → Microsoft 365 and Microsoft Teams**.
2. Keep **Make agent available in Microsoft 365 Copilot** ticked → **Add channel**.
3. Optional: **Edit details**: short description, long description, developer name (under **More**), icon (`assets/PowerClippy-color-192.png`). Leave *Users can add this agent to a team* and *group and meeting chats* **off**.
4. Select **See agent in Microsoft 365** → **Add**.
5. **Don't** use *Show to my teammates* or *Show to everyone in my org*.

**Check:** Microsoft 365 Copilot → **Agents** lists PowerClippy, and `menu` works there.

### Step 11: Verify with the evaluation set

Ask the questions in [PowerClippy-Eval-Set.csv](PowerClippy-Eval-Set.csv), or upload it as a test set in Copilot Studio's agent evaluation. Expected answers are generic; replace the `<...>` placeholders with names from your tenant.

---

## Path B: build from source

Use this to rebuild from the YAML files, or if you change the schema name or publisher prefix.

1. **Steps 1–3** of Path A.
2. **Create an empty agent** named PowerClippy in the environment, with schema name `mono_powerclippy` and publisher prefix `mono` (see the build guide, Phase 5). If you use different names, replace every `mono_powerclippy.` reference in the YAML.
3. **Clone it to a short path:**
   ```bash
   pac copilot clone --bot <bot-id> --display-name pc --output-dir C:/pc --environment https://<domain>.crm.dynamics.com/
   ```
4. **Create the custom connector:**
   ```bash
   pac connector create --api-definition-file ./connector/apiDefinition.swagger.json --api-properties-file ./connector/apiProperties.json --environment https://<domain>.crm.dynamics.com/ --solution-unique-name PowerClippy
   ```
   Note the new API name; replace the old one everywhere it appears (`grep -r new-5fpowerclippy`).
5. **Create the connection references.** `setup/New-ConnectionReferences.ps1` reads all seven from `PowerClippy/connectionreferences.mcs.yml`; set `ConnectorApiName` in your config file (or pass `-CustomConnectorApiName <api-name>`) so the custom connector's reference points to your connector.
6. **Link the custom connector's reference** (needed for solution export):
   ```bash
   powershell -NoProfile -ExecutionPolicy Bypass -File ./setup/Link-CustomConnectorReference.ps1 -EnvironmentUrl https://<domain>.crm.dynamics.com -TenantId <tenant-id> -ConnectorApiName <api-name> -ConnectionReferenceLogicalName mono_powerclippy.<api-name>.auditsearch
   ```
7. **Copy the source into the clone** (keep the clone's `.mcs/` folder): `agent.mcs.yml`, `settings.mcs.yml`, `connectionreferences.mcs.yml`, `topics/`, `actions/`, `knowledge/`.
8. **Pull, push, publish:**
   ```bash
   pac copilot pull --project-dir C:/pc/PowerClippy
   ```
   ```bash
   pac copilot push --project-dir C:/pc/PowerClippy
   ```
   ```bash
   pac copilot publish --bot mono_powerclippy --environment https://<domain>.crm.dynamics.com/
   ```
9. **Add pushed components to the solution:**
   ```bash
   pac solution add-solution-component --environment https://<domain>.crm.dynamics.com/ --solutionUniqueName PowerClippy --component <botcomponent-id> --componentType botcomponent
   ```
10. **Steps 6–11** of Path A.

---

## Updating an existing installation

1. Back up first: `pac solution export --name PowerClippy --path ./local/PowerClippy-backup.zip --environment <url> --overwrite` (`local/` is git-ignored).
2. Import the new package (`pac solution import … --publish-changes`), then `pac copilot publish --bot mono_powerclippy`.
3. Test the Audit Search connection (`audit security groups`). If its sign-in fails, re-run `Configure-AuditSearch.ps1 -Step All` to set a new secret.

Verified on 30 Oct 2026: a re-import into PowerClippy Admin kept all 63 components, the connector's settings, the connection references and their connections, and the access settings.

## Uninstalling

1. Remove the Microsoft 365 channel (Copilot Studio → Channels → Microsoft 365 and Microsoft Teams → **Remove channel**).
2. Delete the agent in Copilot Studio, or delete the PowerClippy solution and the environment (Power Platform admin center → Environments → **Delete**).
3. Delete the **PowerClippy Audit Search** app registration in Entra (this removes its secrets and consent).
4. Delete SG-PowerClippy-Admins if it isn't used elsewhere.
5. Remove admin roles granted only for PowerClippy.
