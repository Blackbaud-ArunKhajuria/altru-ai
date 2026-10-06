---
name: "altru-bugs-handling"
description: "Analyzes Altru product sprint bugs against a catalogue of proven historical fix patterns (P-1…P-8) to find reusable SQL/C#/config solutions, then generates the actual code fix (SQL script + TFS/C# change) for each matched bug. Use to check if an Altru bug has a known fix pattern, analyze sprint active bugs for matches, generate the SQL and TFS code fix for a bug, or produce a pattern-reuse report. Triggers on check patterns, reuse fixes, historical patterns, generate the fix, write the code fix, SKY API proxy 403, \"Error in proxy call\". Run proactively for recurring Altru bug types."
---

# Altru Pattern Reuse — Sprint Bug Analysis Skill

You are helping the Altru Cheetahs PTG Application Engineering team (Principal Engineer: Arun Khajuria)
find and apply proven fix patterns from historical **Altru product** bugs to current sprint issues,
and to **generate the concrete code fix (SQL + TFS/C#)** once a bug is matched.

**Scope: Altru product only.** Area path: `Products\RDO\Altru`. Repo: `$/Infinity/DEV/Phoenix`.
Do not include BBCRM, BBECRM, ResearchPoint standalone, or Certinia bugs unless they have
`Products\RDO\Altru` in their area path.

---

## Overview

This skill runs a structured analysis in five phases:

1. **Gather** — fetch current sprint active bugs from Azure DevOps, Altru area path only
2. **Match** — score each bug against the Pattern Catalogue (P-1 through P-8)
3. **Verify the code path executes** — *mandatory gate before proposing any fix*
4. **Generate Code Fix** — for each matched bug, produce the SQL and TFS/C# fix in stages
5. **Report** — produce a Word document (.docx) with confidence-rated matches, the generated
   fixes, and copy-paste templates; also write standalone `.sql` / `.cs` fix files per bug

Always complete all phases. Do not stop after listing matches — a matched bug must always be
followed by Phase 3 verification and then a generated code fix.

---

## Phase 1 — Gather Active Bugs

### Default sprint scope

- Project: `Products`
- Team: `Altru Cheetahs`
- Repo (for code fix context): `$/Infinity/DEV/Phoenix`
- **Area path filter: `Products\RDO\Altru` (REQUIRED — Altru only)**

Query the **current sprint** active bugs:

```wiql
SELECT [System.Id], [System.Title], [System.WorkItemType], [System.State],
       [System.AssignedTo], [Microsoft.VSTS.Common.Priority],
       [System.Tags], [System.IterationPath], [System.AreaPath]
FROM WorkItems
WHERE [System.TeamProject] = 'Products'
  AND [System.AreaPath] UNDER 'Products\RDO\Altru'
  AND [System.State] IN ('Active', 'Resolved')
  AND [System.WorkItemType] = 'Bug'
ORDER BY [System.ChangedDate] DESC
```

For a specific sprint, add an IterationPath filter:
```
AND [System.IterationPath] = 'Products\2026\Block 5\2 MonSun\B5S1 (Jun 22 - Jul 05)'
```

For broader historical analysis (common bug types), remove the State filter and add:
```
AND [System.State] IN ('Active', 'Resolved', 'Closed')
ORDER BY [System.ChangedDate] DESC
```

### Enriching bug details

For each active bug, fetch its comments (up to the most recent 5) to extract:
- Stated root cause
- Technology keywords (SQL, C#, BBMS, webform, OAuth, cache, etc.)
- Any fix attempts already in progress
- Affected table/column names, stored procedures, or source files named in the comments
- **Any error the customer quotes verbatim** — match it to source with ADO code search; the exact
  string usually pins the throwing method in one hop

---

## Phase 2 — Pattern Matching

Read the relevant pattern reference files in `patterns/` before scoring (P-1…P-4). P-5 through P-8
are documented inline below.

### Pattern Catalogue Summary

| ID  | Name                           | Trigger Keywords                                                         | Altru Source Bugs        |
|-----|--------------------------------|--------------------------------------------------------------------------|--------------------------|
| P-1 | SQL Dedup / Data Fix Script    | duplicate, dedup, "Data Fix", import, split designation, ClientUsers, dbo.ADDRESS | #3923498, #3920132 |
| P-2 | SQL Query Perf / Exec Plan     | slow, timeout, performance, "Sales Orders tab", constituent lookup, advanced sales, hours | #3926550, #3916184 |
| P-3 | OAuth / Cache / Auth Guard     | 403, Forbidden, scope, OAuth, SKY API, Workato, DataRail, cache, caching directive, login popup, "Please enter your previous credentials" | #4012957, #3617799 |
| P-4 | Batch Count Re-fetch           | batch count, count stale, count not updating, progress stuck, in-memory, Processing Uploaded File | #3944104 |
| P-5 | Auto-Renewal CCP Resume-State Collision | auto-renewal not committing, EFT batch stuck, resume information, CREDITCARDPROCESSINGSTATE, BATCHINSTANCESETFORRESUME, CreditCardProcessing.BusinessProcess | #3373288 |
| P-6 | **Marketing Effort Dynamic Selection Cost** | "is already running", Marketing Effort Segment Record Count Calculation, appeal mailing error, count calculation slow, BPappLock, dynamic selection, UFN_ADHOCQUERYIDSET | **#4095060** |
| P-7 | **CLR DataList N+1 (paging ignored)** | list page slow/not opening, datalist slow, WebShellDataListService.ashx, page hangs, "Appeal Mailings" page | **#4095060 (secondary)** |
| P-8 | **SKY API Translation Service Proxy Auth (401→403)** | "Error in proxy call", "You do not have permission to view this directory or page", SKY API third-party app stopped, 403 Forbidden, "Failed to extract a session key", SKYAPITranslation, infinity translation proxy, proxy user inactive, RemyPass/Workato/DataRail stopped returning data | **#4102982** (#4103391, #3983631 related) |

### Altru-Specific Pattern Indicators

| Signal | Category | Action |
|--------|----------|--------|
| Tag: `Data Fix` or `Data  Fix` | P-1 SQL data fix script | Check P-1 template; need DB snapshot + client sign-off |
| Tag: `Sample` or `5.3x` | Code fix in version branch | Check Phoenix repo changeset for that version |
| Tag: `Pentest_Finding` or `Checkmarx` | Security — not a catalogued pattern | Separate security workflow |
| auto-renewal batch not committing, EFT stuck, "resume information", CREDITCARDPROCESSINGSTATE | P-5 | Use `altru-auto-renewal-credit-card-processing` skill |
| "business process X is already running", appeal mailing count error, marketing effort takes hours | **P-6** | Profile the effort's selections **before** proposing any query fix |
| A list page will not open / spins; `WebShellDataListService.ashx` slow in devtools | **P-7** | Time the datalist's SP separately from the page |
| Third-party app (RemyPass/Workato/DataRail) via SKY API suddenly gets 403 / "Error in proxy call: You do not have permission to view this directory or page"; worked before | **P-8** | Check if platform-wide (multi-tenant) before touching tenant config |
| Bug title: `PCO workflow` / `PCO checkout` | PCO Workflow cluster | No pattern yet — log for P-9 candidacy |
| Bug title: `Unresolved Online Orders` | Unresolved orders cluster | Often data fix (P-1) or infra |
| Bug title contains `TOIL` tag | Infrastructure / DevOps | Escalate to DevOps INF L2 |
| Bug title: `Webform` + `Error` | Webform integration bug | Check PCO or OAuth (P-3) |

### Confidence Scoring Rules

**HIGH** — 2+ trigger keywords match AND the bug is in `Products\RDO\Altru`:
> Apply the pattern template directly. The fix is likely copy-paste-and-adapt.

**MEDIUM** — 1 trigger keyword matches OR the area overlaps but keywords are indirect:
> Use the pattern as the starting investigation point. Likely needs adapting.

**LOW** — Conceptual overlap only. **NO MATCH** — flag as fresh investigation required.

### Per-Bug Checks

#### Duplicate / Data Fix Check (P-1)
- Tag `Data Fix` / `Data  Fix`? → **P-1 HIGH immediately**
- Title contains: "duplicate", "dedup", "import", "UNIQUE KEY", "ClientUsers", "split designation"?
- ➜ Fetch `patterns/P1-sql-dedup.md`.

#### Query Performance Check (P-2)
- Title/comments contain "slow", "timeout", "performance", "taking too long", "hours"?
- Sales Orders tab, advanced sales order, constituent lookup, pledge/receivables report?
- ➜ Fetch `patterns/P2-query-perf.md`. **Then run Phase 3 before proposing anything.**

#### Auth / Cache / OAuth Check (P-3)
- Mentions "403", "Forbidden", "scope", "OAuth", "SKY API", "Workato", "DataRail"?
- Caching directive, login popup, ReportHost.aspx, PAT/impersonation/Run As User?
- ➜ Fetch `patterns/P3-auth-scope.md`. If it is a SKY API **third-party integration** call failing
  with "Error in proxy call" / "You do not have permission to view this directory or page", it is
  **P-8** (below), not the plain scope-mismatch P-3.

#### Batch Count / Stuck Process Check (P-4)
- Mentions "stuck", "Processing Uploaded File", count not updating, in-memory counter?
- ➜ Fetch `patterns/P4-batch-count.md`.

#### Auto-Renewal Credit-Card Resume-State Check (P-5)
- Membership Auto-Renewal / EFT batch **not auto-committing**, "error writing process resume
  information", `CREDITCARDPROCESSINGSTATE`, `BATCHINSTANCESETFORRESUME`, duplicate key on
  `UC_CREDITCARDPROCESSINGSTATE_BUSINESSPROCESSSTATUSIDENTIFIER`?
- Signature: commits fine **manually** but never auto-commits nightly; batches pile up `STATUSCODE = 0`.
- Root cause (#3373288): `USP_CREDITCARDPROCESSING_WRITERESUMEINFO` does a blind INSERT into
  `CREDITCARDPROCESSINGSTATE` (unique on `BUSINESSPROCESSSTATUSIDENTIFIER`) but resume is detected
  by `BATCHID`. Leftover state row → duplicate key → run dies; resume flag never cleared (no `Finally`).
- ➜ Use the **`altru-auto-renewal-credit-card-processing`** skill and
  `C:\Products\DEV\Phoenix\docs\bugs\Bug3373288-AutoRenewal-CCP\`.

#### SKY API Translation Service Proxy Auth Check (P-8)
- A third-party app (RemyPass, Workato, DataRail, custom) calling **SKY API** suddenly returns
  403 with `"Error in proxy call: You do not have permission to view this directory or page."`,
  and it **worked before** with no client-side change?
- Translation Service logs show `"Response from infinity translation proxy was not successful … 403 …"`
  and `"onServiceClientException status code: 403 - Error querying InfinityTranslationService service"`?
- AppFx/Web Shell (Splunk) shows `ServiceException: Failed to extract a session key from the
  RequestContext` with `Is authenticated: False`?
- ➜ Go to the **P-8** section below. Determine multi-tenant (platform incident) vs single-tenant
  (application/proxy-user) **before** proposing any change.

---

#### P-6 — Marketing Effort Dynamic Selection Cost *(from #4095060)*

**Customer-visible symptom:** *"An error occurred before executing the process 'Marketing Effort
Segment Record Count Calculation': The business process … is already running."*

**The error is a symptom, not the defect.** The locking is correct; the process is simply too slow.

##### Mechanism (verified in source)

`AppBusinessProcess.vb` (platform — **not** in Phoenix; `$/Infinity/DEV/Phoenix/Blackbaud.AppFx.Server`
contains only a `Deploy` folder):

```vb
Public Overrides Function GetAppLockNameForParameterSet() As String
    Return "BPappLock:" & Me.ProcessContext.ParameterSetID.ToString
End Function
...
Dim returnvalue As Integer = GetDbAppLock(_processConnection, lockName, 0)   ' 0 = fail fast
If returnvalue <> 0 Then
    Throw New Exception("The business process '" & _spec.Name & "' is already running.")
```

Three lock layers — do not confuse them:

| Layer | Resource | Timeout | Symptom |
|---|---|---|---|
| L1 | `<BusinessProcessStatusID>` | 0 | "status app lock not acquired" |
| L2 | `BPappLock:<ParameterSetID>` | **0** | **"…is already running"** ← the customer's error |
| L3 | `SegmentExclusionCache:<segid>`, then `MKTSEGMENT:<id>` + `IDSETREGISTER:<selid>` **per selection** | 3600000 (60 min) | hangs up to an hour, then errors |

##### Critical: blocked attempts leave NO database row

`CreateParameterSetAppLock()` throws inside `CreateAppLock()`, which runs **before**
`CreateProcessStatus()` → `USP_BUSINESSPROCESSSTATUS_ADDPROCESS` is never reached.

- **No row in `BUSINESSPROCESSSTATUS`.**
- **Nothing in `dbo.Error`** (verified: 0 of 1.36M rows match `'%is already running%'`).
- Only traces: Windows event log (**event 805300**) and IIS (a **~1 second HTTP 500**).
- Because it is fail-fast, victims are **fast** requests — they never appear in slow-request monitoring.
  This is why such bugs look like phantoms.

##### Root cause of the slowness

Efforts built from **many dynamic ad-hoc-query selections** over a large constituent set.
A dynamic selection (`IDSETREGISTER.STATIC = 0`, `OBJECTTYPE = 1`) is a TVF that re-executes its
whole ad-hoc query on every touch.

Evidence from #4095060:

| Effort | Segments | Selections | Dynamic | Records | Duration |
|---|---|---|---|---|---|
| Three appeal efforts | **1** | 30 | **30** | ~71,000 | **189–227 min** |
| Acknowledgements | 54 | 53 | 54 | — | **3 min** |
| Control (static) | 2 | 3 | **0** | — | **< 1 min** |

**Key discriminator: runtime is disproportionate to segment count.** A 54-segment effort finished
in 3 minutes while 1-segment efforts took 3+ hours. Historic worst: **954 min (15.9 h)**, recurring
since March 2026. The five worst runs were `STATUSCODE = 2` *"Process terminated unexpectedly"*
with end times clustered at ~02:09/02:23 — killed by a nightly window after 10–16 hours of work.

##### Diagnostic — run this first

```sql
-- Which efforts are at risk? Ranks by dynamic selection count.
SELECT TOP 40
    MS.NAME AS EffortName, MS.MAILINGTYPECODE,
    COUNT(DISTINCT SS.ID)                          AS Segments,
    COUNT(DISTINCT MSS.SELECTIONID)                AS Selections,
    SUM(CASE WHEN IR.STATIC = 0 THEN 1 ELSE 0 END) AS DynamicSelections,
    MAX(CI.RECORDCOUNT)                            AS MaxCachedRecords
FROM dbo.MKTSEGMENTATION MS
JOIN dbo.MKTSEGMENTATIONSEGMENT SS  ON SS.SEGMENTATIONID = MS.ID
LEFT JOIN dbo.MKTSEGMENTSELECTION MSS ON MSS.SEGMENTID = SS.SEGMENTID
LEFT JOIN dbo.IDSETREGISTER IR      ON IR.ID = MSS.SELECTIONID
LEFT JOIN dbo.MKTSEGMENTATIONSEGMENTCACHEINFO CI ON CI.SEGMENTID = SS.ID
GROUP BY MS.NAME, MS.MAILINGTYPECODE
HAVING SUM(CASE WHEN IR.STATIC = 0 THEN 1 ELSE 0 END) > 0
ORDER BY DynamicSelections DESC, MaxCachedRecords DESC;

-- Run history WITH the user (IIS cs_username is usually blank; APPUSER has the name)
SELECT BPS.STARTEDON, BPS.ENDEDON,
       DATEDIFF(MINUTE,BPS.STARTEDON,BPS.ENDEDON) AS Minutes,
       BPS.STATUSCODE, AU.USERNAME AS RunBy, MS.NAME AS EffortName,
       BPS.BUSINESSPROCESSPARAMETERSETID AS ParameterSetID
FROM dbo.BUSINESSPROCESSSTATUS BPS
LEFT JOIN dbo.APPUSER AU ON AU.ID = BPS.STARTEDBYUSERID
LEFT JOIN dbo.MKTSEGMENTATIONSEGMENTCALCULATEPROCESS CP ON CP.ID = BPS.BUSINESSPROCESSPARAMETERSETID
LEFT JOIN dbo.MKTSEGMENTATION MS ON MS.ID = CP.SEGMENTATIONID
WHERE BPS.BUSINESSPROCESSCATALOGID = '585D30CC-C822-4441-A161-F8A2306BDA8D'
  AND BPS.STARTEDON >= DATEADD(DAY,-60,GETDATE())
ORDER BY Minutes DESC;

-- Derive the blocked window (victims have no row — compute it from the holder)
-- Any launch of the same ParameterSetID between STARTEDON and ENDEDON failed in ~1s.
```

##### Fix order

1. **Config first, no release:** convert the shared dynamic selections to **static**. Tell the client
   static selections need a refresh schedule, or that becomes the next ticket.
2. Improve the error message (platform ask): it names no user/effort/time.
   `UFN_BUSINESS_PROCESS_CURRENTLYRUNBY` already exists and is used by the sibling check.
3. Guard `sp_releaseapplock` in `USP_MKTSEGMENTATIONSEGMENT_CACHEPREVIOUSSEGMENTEXCLUSIONS`
   (~end of proc, outside try/catch) with `if APPLOCK_MODE('public', @LOCKNAME, 'Session') <> 'NoLock'`
   — matches the sibling `USP_MKTSEGMENT_RELEASEAPPLOCK`. Otherwise error 1223 masks the real timeout.
4. Only then consider query tuning — **and run Phase 3 first**.

##### Do NOT

- Do not raise the applock timeout (makes users wait hours instead of failing fast).
- Do not add retry logic around the launch.
- Do not write stuck-process cleanup scripts without first proving the lock is actually orphaned
  (`STATUSCODE = 2` **and** `applock_test(...) = 0`). In #4095060 it was a live, completing process.

##### Useful GUIDs (Altru, stable across installs)

| GUID | Business Process |
|---|---|
| `585D30CC-C822-4441-A161-F8A2306BDA8D` | Marketing Effort Segment Record Count Calculation |
| `22C3D75C-A956-4BFC-A5FD-4B866BAEF509` | Marketing Effort Activate Process |
| `854A7703-89D1-4EF7-A7C6-B8836A887092` | Membership Renewal Effort Process |

Related config trap: `USP_MKTSEGMENTATIONSEGMENTCALCULATEPROCESS_GETDYNAMICSELECTIONS` (which drives
*"All selections used in the marketing effort must be static"*) checks **two** places —
`MKTSEGMENTSELECTION` **and** `MKTSEGMENTATIONFILTERSELECTION`. Clients commonly fix the segment
selections and miss the mailing-level filters, so the error persists. Always name the exact effort
and scope when advising Support.

---

#### P-7 — CLR DataList N+1, paging ignored *(from #4095060 secondary)*

**Symptom:** a list page will not open, or takes minutes. DevTools shows a slow
`WebShellDataListService.ashx?...&dataListId=...&start=0&limit=30`.

**Discriminator: the SP is fast, the page is slow.** Always time them separately.

##### Diagnostic

```sql
-- 1. Is the datalist SP-based or CLR-based?
SELECT ID, NAME, PROCEDURENAME, IMPLEMENTATIONTYPENAME, ASSEMBLYNAME, CLASSNAME
FROM dbo.DATALISTCATALOG WHERE ID = '<dataListId from the URL>';
-- IMPLEMENTATIONTYPENAME = 'CLR' and blank PROCEDURENAME => P-7 candidate.

-- 2. Time the underlying SP alone.
DECLARE @t0 datetime2 = SYSDATETIME();
-- EXEC the datalist's SP with the page's default params, INSERT INTO #tmp ...
SELECT DATEDIFF(MILLISECOND, @t0, SYSDATETIME()) AS SP_ElapsedMs;
```

If the SP returns in milliseconds but the page takes minutes, the cost is in the CLR loop.

##### Verified example

`ContextlessAppealMailingDatalist.vb` (Appeal Mailings page,
page `E61AABE7-DA92-453D-83D1-F452A2AE10AA`, datalist `1AD68942-ABA4-4FFA-ADA1-55773FDA91B9`):

```vb
For Each mailing In ...ExecuteSP(conn, ..., timeout:=300)
    If Not isBBEC Then                       ' Altru always takes this branch
        Try
            hasEmailJob = AppealMailingStatusHelper.HasEmailJob(mailing.ID, ..., 300)   ' per row
            If Not hasEmailJob Then
                emailJobStatusIds = CommunicationEmailHelper.GetEmailJobStatusIds(conn, mailing.ID)  ' per row
                hasEmailJob = CommunicationEmailHelper.EmailJobsActiveQueuedOrCompleted(...)          ' per row
            End If
        Catch ex As Exception
            hasEmailJob = True               ' swallows all failures — silent
        End Try
    End If
Next
```

Measured: `USP_DATALIST_CONTEXTLESSAPPEALMAILING` returned **1,399 rows in 50 ms**; the page took
minutes. The browser asked for **30** rows; the CLR built all **1,399** first, then paged —
~98% of the work discarded. 1,400–4,200 round trips to render 30 rows. Scales linearly with the
client's mailing count, so it degrades permanently as efforts accumulate.

##### Fix order

1. **Apply `start`/`limit` before the per-row loop** — smallest, safest, ~98% less work.
2. Batch the per-row lookup into one set-based call for the page's IDs.
3. Push the derived column into the SP as a JOIN (best shape, largest change).
4. Stop swallowing exceptions — a silent catch makes systemic failures invisible.

##### Generalisation

Any `IMPLEMENTATIONTYPENAME = 'CLR'` datalist that calls a helper **inside** a `For Each` over the
full result set is a P-7 candidate. Grep the class for a per-row `Helper.` call or a nested
`ExecuteSP` and check whether paging is applied before or after the loop.

---

#### P-8 — SKY API Translation Service Proxy Auth (401→403) *(from #4102982)*

**Customer-visible symptom:** a third-party app that reads Altru through **SKY API** (e.g. RemyPass
digital membership cards, Workato, DataRail, a custom integration) suddenly starts returning
*"Error in proxy call: You do not have permission to view this directory or page."* (HTTP 403),
after working for weeks/months with **no client-side change**. Often a sharp onset at a specific time.

**The 403 the caller sees is a re-labeled backend 401.** This is not a scope mismatch (that is plain
P-3) and usually not a code defect — it is a broken auth/session hop, most often a **platform SKY API
incident**, occasionally a **single-tenant application/proxy-user** problem.

##### Request flow (verified in `inf-apitr-svc` + Phoenix source)

```
RemyPass
  -> SKY API REST (api.sky.blackbaud.com, Blackbaud SKY OAuth 2.0)   <- OAuth enforced HERE, at the gateway
       gateway injects trusted headers: blackbaud-authentication-userid, blackbaud-environmentid
    -> Blackbaud.Infinity.Api.TranslationService   [repo: inf-apitr-svc, ClientAppName "SKYAPITranslation"]
         BuildTranslationContextAsync: obtains a service SAS token for audience {pod}\{entitlement}
         AdHocQueryRunService.RunAsync: builds SOAP <AdHocQueryProcessRequest>
         POSTs to  {instanceUrl}/appfxwebservice.asmx   via the "infinity translation proxy"
         SOAPAction: Blackbaud.AppFx.WebService.API.1/AdHocQueryProcess
      -> Altru Web Shell / AppFx (Blackbaud.AppFx.Server) appfxwebservice.asmx
           AdHocQueryProcess.vb runs the query
           -> QueryUsageEvent.CreateAndRaise -> AppFxWebServiceUsageEvent.GetSessionKey(context)
              reads context.RootRequest.ClientAppInfo.SessionKey; throws if no session established
```

Key code facts:
- **The exact error string is minted in the Translation Service, not AppFx.**
  `InfinityDataAdapter.SendAuthenticatedRequest` (repo `inf-apitr-svc`,
  `/src/…/DataAccess/InfinityDataAdapter.cs`): when the backend returns **401 Unauthorized**, it is
  caught and **re-thrown as 403 Forbidden** with `$"Error in proxy call: {responseContent}"`, where
  `{responseContent}` is the IIS body *"You do not have permission to view this directory or page."*
- The Translation Service authenticates the **service-to-service hop** with a SAS token
  (`Authorization` header, from `ServiceRouter.Route`, audience `{pod}\{entitlement}`), and forwards
  the end-user identity via headers. `ClientAppInfo` (`InfinityUtils.BuildClientAppInfo`) carries
  `REDatabaseToUse` + `ClientAppName="SKYAPITranslation"` — **no SessionKey attribute**; the AppFx
  session is established server-side (cookies / proxy), and its GUID is the SessionKey.
- **AppFx side** (`Blackbaud.AppFx.Server/Monitoring/AppFxWebServiceUsageEvent.vb`):
  `GetSessionKey` throws `ServiceException("Failed to extract a session key from the RequestContext")`
  when `context.RootRequest` is null or `ClientAppInfo.SessionKey` is not a parseable GUID — i.e.
  the request reached AppFx with **no established session**.

##### Where the application (proxy) user lives and how it authenticates

- OAuth authenticates **the third-party app** at the SKY API gateway (registered app id/secret +
  scopes + subscription key; a BBID owner consents once).
- The **application (proxy) user** is created in Altru **Administration → Application users** in the
  Web Shell, **linked to a BBID**, and granted a **system role** with the feature/record permissions
  the integration needs. Only the linked BBID owner can mint a **PAT** for it.
- At the backend, the SAS-authenticated proxy call + forwarded BBID is mapped to this application
  user; an AppFx session is established as that user and the SOAP request runs under it (its system
  role governs security — a missing feature grant surfaces as *"The current user does not have rights
  to use this search."*).

##### Log signatures (both ends)

Translation Service logs (Splunk export, filter by `EnvironmentId` = the tenant's `p-…` id):
```
Response from infinity translation proxy was not successful
{"title":"Forbidden","status":403,"detail":"Error in proxy call: You do not have permission to view this directory or page.", ...}
onServiceClientException status code: 403 - Error querying InfinityTranslationService service
```
Level = WARNING. In #4102982 these were **hourly and continuous on the incident day (all failing,
zero 200s)** for the affected tenant's env.

AppFx / Web Shell logs (Splunk, `s20aalt0x…web01a/b` nodes, ASP.NET event 1309):
```
ServiceException: Failed to extract a session key from the RequestContext
   at AppFxWebServiceUsageEvent.GetSessionKey
   at QueryUsageEvent.CreateAndRaise( … AdHocQuery … )
   Is authenticated: False   Request URL: (blank)
```
In #4102982 this fired **across many tenants' app pools simultaneously** → platform-wide.
Note: `"The current user does not have rights to use this search"` on `UIModelingService.ashx` with
`Is authenticated: True / Federation` = ordinary interactive **UI** users, **usually unrelated noise**;
do not conflate it with the API path.

##### Diagnostic — decide multi-tenant vs single-tenant first

1. **Multi-tenant?** If the AppFx `Failed to extract a session key` errors span several unrelated app
   pools at the same time, or a sibling bug reports the same error the same day (e.g. #4103391 Royal
   Oak), it is a **platform SKY API incident** → not a code/data fix; confirm resolution timing
   (Aug-26 incident "resolved later in the day") and close as duplicate of the platform event.
2. **Single-tenant only?** If only this env fails after the platform window, inspect the tenant's
   **application/proxy user**: is it **Active**? is its **linked app/BBID user still in the org**?
   does it retain the required **system role** permissions? (The #3983631 Mark Arts scenario: the
   linked app user left the org → proxy user went inactive → same error string.)

##### Fix / resolution

- **Platform incident:** no Phoenix change. Track the SKY API / hosting incident; verify the third-party
  app recovers after resolution. Set Root Cause to the platform SKY API auth incident.
- **Single-tenant proxy user:** create a **new proxy/application user + PAT with identical system
  roles/permissions** and re-point the integration (the #3983631 remediation). Not a code fix.

##### Source files (for reference, not necessarily to change)

| Component | Repo / Path |
|---|---|
| 401→403 "Error in proxy call" relabel | `inf-apitr-svc` `/src/…/DataAccess/InfinityDataAdapter.cs` |
| SAS auth context, env lookup | `inf-apitr-svc` `/src/…/BusinessLogic/ApiTranslationService.cs` |
| SOAP AdHocQuery build/submit/poll | `inf-apitr-svc` `/src/…/BusinessLogic/AdHocQueryRun/AdHocQueryRunService.cs` |
| ClientAppInfo / BuildAppFxRequest | `inf-apitr-svc` `/src/…/Utils/InfinityUtils.cs` |
| SessionKey extraction (throws) | Phoenix `Blackbaud.AppFx.Server/Monitoring/AppFxWebServiceUsageEvent.vb` |
| AdHoc query processor + usage event | Phoenix `Blackbaud.AppFx.Server/Requests/AdHocQuery/AdHocQueryProcess.vb` |

##### Do NOT

- Do not re-register scopes / touch the tenant's application user before confirming it is **not** a
  multi-tenant platform incident — you would be changing config that was never broken.
- Do not treat the caller-visible **403** as authoritative — the backend actually returned **401**;
  the Translation Service relabels it.
- Do not attribute it to the AppFx query code — `GetSessionKey` throws on the **telemetry** path
  after the query, and is a *marker* of the missing session, not the failing line.

---

## Phase 3 — Verify the Code Path Actually Executes *(MANDATORY GATE)*

**Do not propose a query fix until you have confirmed the code you want to change runs for this
customer's parameter values.**

In #4095060, four plausible, well-reasoned hypotheses were all eliminated by reading the
configuration — after a full code review had already recommended them:

| Hypothesis | Why it was dead |
|---|---|
| Literal `IN (...)` GUID list causes plan churn | `SEQUENCE = 1` → every `if @SEGMENTSEQUENCE > 1` block is skipped. Code never runs. |
| Hardcoded `option (hash join, merge join)` bans nested loops | Gated `if @NEEDCAST = 1`; `PRIMARYKEYTYPENAME = 'uniqueidentifier'` → never applied. |
| `cast(... as varchar(36))` makes joins non-sargable | Same gate → no casting occurs. |
| Missing covering index on the cache table | `..._CREATETABLE` already builds `INCLUDE ([CONSTITUENTID])`. A `sys.index_columns` query filtered to `is_included_column = 0` hides INCLUDE columns — check both. |

**Shipping any of them would have produced zero improvement.**

For P-8, the equivalent gate is: **confirm multi-tenant vs single-tenant** before touching any tenant
config — a platform incident needs no tenant change at all.

### The gate

For the SP or class you intend to change, pull the customer's **actual** parameter values and walk
the branches:

```sql
-- Marketing effort example — these four values decide which paths execute
SELECT SS.SEQUENCE,                 -- 1 => all "previous segment" logic SKIPPED
       QVC.PRIMARYKEYTYPENAME,      -- 'uniqueidentifier' => @NEEDCAST = 0: no cast, no join hint
       PKG.CHANNELCODE,             -- 0 => postal address processing runs
       MS.HOUSEHOLDINGTYPECODE      -- <> 0 => householding runs
FROM dbo.MKTSEGMENTATIONSEGMENTCALCULATEPROCESS CP
JOIN dbo.MKTSEGMENTATION MS ON MS.ID = CP.SEGMENTATIONID
JOIN dbo.MKTSEGMENTATIONSEGMENT SS ON SS.SEGMENTATIONID = CP.SEGMENTATIONID
JOIN dbo.MKTSEGMENT SEG ON SEG.ID = SS.SEGMENTID
LEFT JOIN dbo.MKTPACKAGE PKG ON PKG.ID = SS.PACKAGEID
LEFT JOIN dbo.QUERYVIEWCATALOG QVC ON QVC.ID = SEG.QUERYVIEWCATALOGID
WHERE CP.ID = '<parameter set id>';
```

Then state explicitly, per proposed fix: **"this code path executes / does not execute for this
customer, because \<value\>."** If it does not execute, withdraw the fix and say so.

### Measure before attributing

- Time the SP separately from the page/process (P-7).
- Prefer a control case: find a comparable object that is **fast** and diff the configuration.
  A fast 54-segment effort next to a slow 1-segment effort disproves "cost scales with segments"
  instantly.
- Distinguish **computing** from **waiting**: `sys.dm_exec_requests` (`wait_type`,
  `blocking_session_id`, `cpu_time` vs `total_elapsed_time`). They need different fixes.

---

## Phase 4 — Generate Code Fix (SQL + TFS)

For each bug scored HIGH or MEDIUM **and** passing the Phase 3 gate, produce a concrete fix — SQL
and/or TFS/C#. Work in stages so there is always a usable artifact.

### Stage A — Adapted template (always produced)

Fill the matched pattern's template with the bug's specifics:

- **SQL fixes (P-1, P-2, P-5 data unblock, P-6 diagnostics):** substitute real tables, columns,
  keys, date window, process/CHANGEAGENT filter, lookup IDs.
- **TFS/C# fixes (P-3, P-4, P-5 source, P-7, `5.3x` bugs):** name the target file(s), class/method,
  and write the adapted snippet.
- **P-8 (platform/config, usually no code fix):** produce the diagnostic (multi-tenant check + log
  filters + application-user inspection), not a source change, unless a single-tenant proxy-user
  remediation is confirmed.

Mark clearly as **DRAFT — TEMPLATE (not yet validated against live schema/source)**.

### Stage B — Live refinement

- **SQL:** use `search_objects_*` to confirm names/constraints and `execute_sql_*` for **SELECT/COUNT
  only**. Never run DELETE/UPDATE/DDL from this skill. Fold confirmed names and real row counts back in.
- **TFS/C#:** use `tfvc_list_items`, `tfvc_get_file_content`, `tfvc_list_changesets`,
  `tfvc_get_changeset`, `tfvc_get_file_changes` against `$/Infinity/DEV/Phoenix`; ADO code search to
  cross-reference. Produce an **actual before/after diff**, on the correct version branch for `5.3x`.
  For P-8, the Translation Service source lives in the **`inf-apitr-svc`** git repo (not Phoenix).

**Ownership check — know what is in Phoenix before promising a fix:**

| Component | In Phoenix? |
|---|---|
| `Blackbaud.AppFx.Marketing.Catalog` (SPs, datalists) | ✅ Yes — ours |
| `Blackbaud.AppFx.Server` (`AppBusinessProcess.vb`, applocks, BP framework, AppFxWebService) | ❌ **No** — `Deploy` folder only; platform ask |
| `Blackbaud.AppFx.Platform.SqlClr` (notifications) | ⚠️ Verify before assuming |
| `Blackbaud.Infinity.Api.TranslationService` (SKY API translation, P-8) | ❌ Separate repo `inf-apitr-svc`; platform/SKY-API team |

Promote to **CONCRETE FIX** once refined; otherwise note "live validation pending".

### Output per bug

1. **SQL fix** — ready-to-run script (SELECT preview → snapshot → change → verify), or "N/A".
2. **TFS/C# fix** — target path + before/after diff, or "N/A".
3. **Validation status** — DRAFT or CONCRETE, and what was checked.
4. **Phase 3 result** — which paths execute, which proposed fixes were withdrawn and why.
5. **Rollout notes** — lower env → prod, snapshot/backup, client sign-off for data fixes.

---

## Phase 5 — Report

Generate a Word document (`.docx`) using python-docx **and** standalone fix files.

```
Title: Altru — Pattern Reuse Analysis
Subtitle: Sprint <name>  ×  Historical Fix Patterns
Date, Team, Author

1. Executive Summary
   - Summary table: Bug | Pattern Match | Confidence | Fix Type (SQL/TFS/Config) | Est. Time Saved
2. High-Confidence Matches (one section per bug)
   - Root cause (from ADO comments + live verification)
   - Why this pattern applies
   - Hypotheses TESTED AND ELIMINATED  ← from Phase 3; often the most valuable section
   - Proposed Code Fix (SQL + TFS) with validation status + rollout notes
   - Key lessons / checklist
3. Medium-Confidence Matches
4. Low/No Match Bugs (PCO cluster, Unresolved Orders, Security)
5. Pattern Catalogue (P-1 through P-8)
6. Recommendations (immediate actions, new pattern candidates)
Appendix — Altru source bugs used to derive patterns
```

### Standalone fix files

Write into a `fixes/` subfolder next to the report:

- `fixes/Bug<ID>_fix.sql` — SELECT preview, snapshot, change, verify; commented and env-gated.
- `fixes/Bug<ID>_fix.cs` — before/after change, with the `$/Infinity/DEV/Phoenix` path and version
  branch in a header comment.
- `fixes/Bug<ID>_diagnostic.sql` — read-only profiling for bugs where the root cause is not yet
  pinned (P-6/P-7 especially).

Each header must state: bug ID, title, pattern matched, validation status, rollout order.

### Save locations

- Report: `C:\Users\Arun.Khajuria\Claude\Projects\Claude PM Pilot (1)\Altru_PatternReuse_<SprintName>_<Date>.docx`
- Fix files: `C:\Users\Arun.Khajuria\Claude\Projects\Claude PM Pilot (1)\fixes\`

If python-docx is unavailable (`pip install python-docx --break-system-packages` fails, or no shell),
produce the report as Markdown in the same folder and say so.

---

## Pattern Reference Files

| Pattern | Reference | When to Load |
|---------|-----------|--------------|
| P-1 | `patterns/P1-sql-dedup.md` | `Data Fix` tag, duplicate data, import error |
| P-2 | `patterns/P2-query-perf.md` | slow, timeout, performance, Sales Orders tab |
| P-3 | `patterns/P3-auth-scope.md` | 403, OAuth, cache directive, login popup, SKY API scope |
| P-4 | `patterns/P4-batch-count.md` | stuck process, count stale, Processing Uploaded File |
| P-5 | Skill `altru-auto-renewal-credit-card-processing` + `C:\Products\DEV\Phoenix\docs\bugs\Bug3373288-AutoRenewal-CCP\` | auto-renewal not committing, EFT stuck, CREDITCARDPROCESSINGSTATE |
| P-6 | Inline above | "already running", marketing effort count slow, appeal mailing error |
| P-7 | Inline above | list page slow/not opening, CLR datalist |
| P-8 | Inline above | SKY API third-party app 403, "Error in proxy call", "Failed to extract a session key", proxy user inactive |

---

## Working With Altru Databases — Gotchas

Learned the hard way; each of these produced a wrong conclusion before being caught.

- **`BUSINESSPROCESSSTATUS` timestamps are LOCAL (EDT), not UTC** — even when a tool renders them
  with a `Z` suffix. Splunk/IIS are UTC. Offset by 4 hours (EDT) before correlating, or you will
  conclude data is "missing" when it is present.
- **`applock_test` returns `1` = FREE, `0` = HELD.** The inversion is easy to get backwards.
- **`STATUSCODE`** (no lookup table): `0` Completed · `1` Running · `2` Did not complete
  (incl. "Process terminated unexpectedly") · `3` Completed with exceptions · `4` Enqueued.
- **`dbo.Error.ID` is the "Error reference number"** shown in the Altru UI. `dbo.WEBERRORLOG` is
  BBIS-only and typically empty. Framework `ServiceException`s (e.g. "already running") are **not**
  written to `dbo.Error` — they go to the Windows event log only.
- **Do not join `sys.partitions` across all per-effort cache tables** (`MKTSEGMENTATIONSEGMENTCACHE*`
  — thousands of them; 3,287 on one client). It times out. Scope to a single `OBJECT_ID`.
- **`APPUSER.USERNAME` gives the real person.** IIS `cs_username` is usually blank (anonymous at the
  edge, Federation downstream); join `BUSINESSPROCESSSTATUS.STARTEDBYUSERID` instead.
- Per-effort cache tables accumulating in the thousands is **normal**, not orphaned data — verify
  against `MKTSEGMENTATION` before proposing a P-1 cleanup.
- **SKY API failures (P-8): the caller's 403 hides a backend 401.** The Translation Service relabels
  a backend `401 Unauthorized` as `403 Forbidden "Error in proxy call: …"`. Correlate the Translation
  Service log (by `EnvironmentId`) with the AppFx `Failed to extract a session key` rows, and check
  whether the failure is multi-tenant (platform) before touching the tenant's application/proxy user.

---

## Tips

- **A match is not the deliverable — the code fix is.** Never end after Phase 2.
- **Phase 3 is not optional.** Verify the code path executes for the customer's parameter values
  before proposing a query fix. Four good hypotheses died on this check in #4095060.
- **Staged fix generation:** emit the Stage A template first so there is always an artifact.
- **Read-only against live SQL.** SELECT/COUNT only; DELETE/UPDATE/DDL go in the script for a human.
- **`Data Fix` tag = always P-1 first.**
- **Auto-renewal EFT not committing = P-5** → hand off to the dedicated skill.
- **"Already running" = P-6**, and the lock is almost never the defect — look at the runtime.
- **A slow page with a fast SP = P-7.**
- **SKY API third-party app suddenly 403 / "Error in proxy call" = P-8** — decide multi-tenant
  (platform incident, no fix) vs single-tenant (application/proxy-user) before changing anything.
- **PCO Workflow bugs are a cluster**, not a pattern — group and flag for P-9 candidacy.
- **Unresolved Online Orders** — usually data fix (P-1) or infra (check TOIL tag).
- Always check ADO comments before scoring — root cause is usually stated within 2–3 comments.
- `5.38`, `5.39` tags = Altru release version; target that branch in the generated diff.
- `Pentest_Finding` / `Checkmarx` — start with P-3 for cache/auth, but these have their own workflow.
- Multiple patterns can apply to one bug (#4095060 matched **P-6 and P-7**) — generate a fix for each.
- **Report what you eliminated, not just what you found.** A list of disproven hypotheses prevents
  the next engineer repeating the work, and prevents shipping no-op fixes.
