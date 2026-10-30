# How the CoE Starter Kit tracks Power Apps usage

Reverse engineered from the shipped package, for rebuilding the capability in PowerClippy. Every statement below is marked **Observed** (read directly from the shipped files), **Verified** (confirmed on Microsoft Learn), or **Inferred** (a conclusion drawn from observed facts).

## Evidence base

| Item | Value |
|---|---|
| Kit status | **Verified**: "no longer actively maintained"; issues not reviewed; capabilities moving to Power Platform admin center Inventory, Usage, Monitor and Actions ([starter-kit](https://learn.microsoft.com/power-platform/guidance/coe/starter-kit)) |
| Artifact | `CenterofExcellenceCoreComponents_4.50.9_managed.zip`, release *CoE Starter Kit February 2026* (latest, published 2026-02-10) |
| Size / SHA-256 | 28,975,074 bytes / `cd2b8def489e8f8e71200bd17e954abb4d0ca1b1b4dc0c907e6202dce37e7d0c`, equal to GitHub's published asset digest |
| REA inventory | `rea-agents@6.8.0 inspect-artifact`, provider *REA safe artifact graph*: 469 artifacts, 535 occurrences, all member hashes verified. Inspection `ai_8dfe5514…f646`, inventory Evidence `ev_50932289…3d28`, manifest `agm_88dd8c19…20cc` |
| Members analysed (SHA-256 matched REA) | `Workflows/AdminAuditLogsSyncAuditLogsV2-BCCF2957-….json` (127,557 B) · `Workflows/AdminAuditLogsUpdateDataV2-1D8BF7B1-….json` (23,054 B) · `Workflows/AdminAuditLogsOffice365ManagementAPISubscription-CCCA4A5C-….json` · `Workflows/AdminAuditLogsLoadeventsfromexportedAuditLogCSVfil-A6875DF5-….json` · `customizations.xml` · 8 `admin_AuditLogs*` environment variable definitions |
| Method | Flow definitions and table metadata parsed as data; nothing from the package was executed |

## 1. Collection: *Admin \| Audit Logs \| Sync Audit Logs (V2)*

**Trigger (Observed):** Recurrence, every 1 hour.

**Credentials (Observed):** A Microsoft Entra app registration, using client ID `admin_AuditLogsClientID` and either a plain-text secret (`admin_AuditLogsClientSecret`) or an Azure Key Vault secret (`admin_AuditLogsClientAzureSecret`). HTTP actions use `ActiveDirectoryOAuth`, so this is application (app-only) authentication, not the signed-in user.

**Time window (Observed):**
- `start = utcNow − (max(60, EndTimeMinutesAgo) + MinutesToLookBack)` minutes
- `end = utcNow − EndTimeMinutesAgo` minutes
- The shipped defaults are `EndTimeMinutesAgo = 0` and `MinutesToLookBack = 65`, so each hourly run covers **125 minutes**, overlapping the previous run.
- **Inferred:** the overlap is deliberate (late-arriving events), and it's safe because writes are keyed on the audit event ID (see below).
- Two pieces of stale code: the action description still says "65 mins", and a `minutes_back` compose action is computed but referenced nowhere (0 references).

**Branch A, Graph (`admin_AuditLogsUseGraphAPI = true`; shipped default is `false`):**
1. `POST {Graph}/beta/security/auditLog/queries` with body `{filterStartDateTime, filterEndDateTime, operationFilters: ["LaunchPowerApp","DeletePowerApp"]}`. Retries until it gets HTTP 201 (5 tries, PT5M).
2. Poll `GET …/queries/{id}` every 60 s until `status = succeeded`. Up to 120 polls (PT1H), after which the flow terminates "Audit Log query wait time exceeded".
3. Page `GET …/queries/{id}/records?$top=500` following `@odata.nextLink`, for up to 100 pages.
4. Keep records where `auditData.RecordType = 256`.
5. App ID: the `Value` of the `auditData.PropertyCollection` entry named `powerplatform.analytics.resource.power_app.id` (or `'no app id'`).

**Branch B, Office 365 Management Activity API (the default):**
1. `GET {manage.office.com}/api/v1.0/{tenant}/activity/feed/subscriptions/content?contentType=Audit.General&startTime&endTime`, following the `NextPageUri` header to collect content IDs.
2. For each content blob: `GET …/activity/feed/audit/{contentId}`.
3. Keep events where `RecordType = 256` and `Operation ∈ {LaunchPowerApp, DeletePowerApp}`. Extract the app ID the same way.

**Writes (Observed), 25 events processed in parallel:**

| Event | Action |
|---|---|
| `LaunchPowerApp` | **Upsert** into `admin_auditlogs` with **row ID = the audit event `Id`**. Fields: `admin_creationtime`, `admin_appid`, lookup `admin_AppLookup` → `admin_apps(appId)`, `admin_operation`, `admin_title` ("Power App Launch {appId} - {time}"), `admin_userupn`, `admin_workload`, plus `admin_userid` on the Graph branch only. The Graph branch writes only if the app already exists in `admin_apps`. The O365 branch attempts the upsert and swallows the failure ("APP DOES NOT EXISTS") when it doesn't. |
| `DeletePowerApp` | If the app exists in `admin_apps`: set `admin_appdeleted = true`, `admin_appdeletedon = event time`. No audit row is written. |

**Inferred consequences:**
- Launches of apps the kit hasn't inventoried yet are **dropped, not queued**.
- Because the row key is the event ID, re-reading the same window never creates duplicates.

**Run bookkeeping (Observed):** Success or failure is recorded in `admin_coesolutionmetadatas` (`admin_lastrun`, `admin_lastrunpassed`). HTTP failures are logged to `admin_syncflowerrorses`.

## 2. Derivation: *Admin \| Audit Logs \| Update Data (V2)*

**Trigger (Observed):** Dataverse webhook on `admin_auditlog`. It fires on `message 4` = *Added or Modified* and `scope 4` = *Organization* (both **Verified** against the `callbackregistration` choice tables), watching columns `admin_operation, admin_userupn`, with filter `admin_operation eq 'LaunchPowerApp'`.

**App derivation (Observed):**
1. Get `admin_apps(appId)`, selecting last-launched, display name, premium flag and environment name.
2. **Last launched:** set `admin_apps.admin_applastlaunchedon = event time` **only if** it is empty or older than the event. Out-of-order events can't move it backwards.
3. Enrich the audit row: `admin_appname`, `admin_appenvironment` (environment display name), `admin_appispremium` (`true` when `admin_usespremiumapi = 'True'`).

**User derivation (Observed):**
1. Split `admin_userupn` on `#` and take the part after it, if any. **Inferred:** this handles guest UPN formats.
2. Office 365 Users `SearchUserV2`. Only if **exactly one** match: `UserProfile_V2` → write `admin_usercity`, `admin_usercountry`, `admin_userdepartment`, `admin_userid`, `admin_userjobtitle`, `admin_username`, and `admin_userisguest` (`userType = 'Guest'`).

## 3. Supporting flows

**Office 365 Management API Subscription (Observed):** Manual trigger. The input text chooses the operation:
- `start`: `POST …/subscriptions/start?contentType=Audit.General`
- `stop`: `POST …/subscriptions/stop?contentType=Audit.General`
- `list`: `GET …/subscriptions/list`, then reports whether an `Audit.General` subscription exists

**Verified:** after starting, the first content blobs can take up to 12 hours, and blobs aren't in event order ([Activity API reference](https://learn.microsoft.com/office/office-365-management-api/office-365-management-activity-api-reference)).

**Load events from exported Audit Log CSV (Observed):** Manual trigger that takes a Purview audit-search CSV export.
- Splits lines, detects the newer `PropertyCollection` format, and parses the app ID, creation time, operation, UPN and workload by string position.
- For `LaunchPowerApp` rows whose app exists, upserts `admin_auditlogs` and updates `admin_applastlaunchedon` using the same "only if newer" rule.

This is the kit's only way to backfill history.

## 4. Data model (Observed, from `customizations.xml`)

**`admin_auditlog`** ("Audit Log", user-owned, key `admin_auditlogid` = audit event ID):

| Column | Type | Column | Type |
|---|---|---|---|
| `admin_creationtime` | datetime | `admin_userupn` | text 300 |
| `admin_operation` | text 100 | `admin_userid` | text 100 |
| `admin_appid` | text 100 | `admin_username` | text 300 |
| `admin_applookup` | lookup → admin_app | `admin_userdepartment` / `city` / `country` / `jobtitle` | text 300 |
| `admin_appname` | text 300 | `admin_userisguest` | bit |
| `admin_appenvironment` | text 100 | `admin_workload` | text 300 |
| `admin_appispremium` | bit | `admin_title` | text 300 |

The table also has an `admin_powerappsapplogs` lookup column, which none of the analysed flows write.

**`admin_app`** ("PowerApps App", user-owned, key `admin_appid`). **Inferred** from the flows' `GetItem`/`UpdateRecord` calls: the key holds the Power Apps app GUID. The columns used are:
- `admin_applastlaunchedon` (datetime)
- `admin_appdeleted` (bit) and `admin_appdeletedon` (datetime)
- `admin_displayname`, `admin_usespremiumapi`, and the `admin_appenvironment` lookup

## 5. The source event, as currently documented

- **Verified:** Power Apps activity records carry `PropertyCollection` entries such as `powerplatform.analytics.resource.power_app.id`, `…environment.id`, `enduser.principal_name`, `powerplatform.analytics.resource.type = PowerApp` and `type = PowerPlatformAdministratorActivityRecord` ([Power Apps activity logs](https://learn.microsoft.com/power-platform/admin/activity-logging-auditing/activity-logs-power-apps)).
- **Verified:** record type **256 = PowerPlatformAdministratorActivity**; **45 = PowerAppsApp** is the older Power Apps record type ([schema](https://learn.microsoft.com/office/office-365-management-api/office-365-management-activity-api-schema)).
- **Inferred:** the kit's `RecordType = 256` filter targets the current record format. Launches logged only under the legacy type 45 would not be captured.
- **Verified prerequisites:**
  - auditing turned on for each Power Platform environment;
  - the reader holds the Purview *Audit Logs* or *View-Only Audit Logs* role;
  - the Graph audit query API is GA at `/v1.0/security/auditLog/queries` (the kit still calls `/beta`) and supports **delegated** work-account permissions as well as application permissions (`AuditLogsQuery.Read.All` covers all services) ([create auditLogQuery](https://learn.microsoft.com/graph/api/security-auditcoreroot-post-auditlogqueries?view=graph-rest-1.0)).

## 6. What this means for PowerClippy

**The kit's model:** collect launches continuously into a store, keyed by event ID, and keep a per-app "last launched" high-water mark. That gives unlimited lookback **from the day collection starts**. Purview search only reaches back as far as audit retention allows (the CoE docs say up to one year, depending on license).

**Options, best first:**

1. **On-demand audit query** (no store, nothing to deploy beyond an app registration):
   - The admin asks "unused apps in the last 90 days". PowerClippy creates a v1.0 audit-log query for `LaunchPowerApp` over that range, polls it until it succeeds, pages the records and takes the distinct `power_app.id` values. It then lists inventory canvas apps that aren't among them.
   - Needs a Graph connection with `AuditLogsQuery.Read.All` (delegated) through a custom connector or app registration, plus the Purview audit role.
   - **Inferred:** the built-in preauthorized HTTP connector won't carry this scope.
   - Queries are asynchronous (the kit allows up to an hour), so this belongs in an **agent flow**, not a topic loop.
2. **Admin center Usage page** (preview since 2026-02-13; **Verified**): distinct users per app over the last **28 days**. No API is documented, so PowerClippy can only point to it.
3. **Rebuild the kit's store:** an hourly flow writing a slimmed `AuditLog(eventId, appId, time, upn)` table plus `LastLaunched(appId)`. This is the only option that builds history beyond Purview retention, but it's a standing deployment with an app secret.

**Open items (not verified):**
- Exact Purview audit retention for this tenant's licenses.
- Whether every launch in this tenant arrives as type 256.
- Whether delegated `auditLogQuery` works from a Copilot Studio connection without further admin consent.

Each needs a live test.
