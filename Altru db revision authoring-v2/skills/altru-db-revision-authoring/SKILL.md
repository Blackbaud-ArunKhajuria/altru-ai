---
name: "altru-db-revision-authoring"
description: "Author a Blackbaud Altru/Infinity DB revision (DBREV) in the Phoenix repo — turn a changed catalog spec or raw SQL DDL (CREATE/DROP INDEX, new columns, altered stored procs, functions, data form templates) into a numbered DBRevision with its Specs snapshot and vbproj registration. Triggers on \"create revisions\", \"create the revision files\", \"add a DB revision\", \"DBREV\", \"I made changes in this spec file\", \"update the table spec\", \"add an index to the spec\", \"ServiceRevisions\", \"LoadSpec\", \"Specs1950\", or Infinity index naming questions."
---

# Authoring Altru / Infinity DB Revisions in Phoenix

Turns a schema or SQL change into a deployable Blackbaud Infinity DB revision: master spec edit → numbered snapshot → project registration → manifest entry → verification.

The most common request form is just **"I made changes in SomeSpec.xml, please create the revisions"** — everything below is what that actually entails.

## Repo layout

Work from the Phoenix working copy root (typically `C:\Products\DEV\Phoenix`).

| What | Where |
|---|---|
| Revision manifests | `Blackbaud.AppFx.Galileo.ServiceRevisions\DBREV<nnnn>.XML` — **note the uppercase `.XML`** |
| Manifest registration | `Blackbaud.AppFx.Galileo.ServiceRevisions\Blackbaud.AppFx.Galileo.ServiceRevisions.vbproj` |
| Per-revision spec snapshots | `Blackbaud.AppFx.Galileo.ServiceRevisions.Specs<nnnn>\` (one project per DBREV number) |
| Snapshot registration | `...Specs<nnnn>\Blackbaud.AppFx.Galileo.ServiceRevisions.Specs<nnnn>.vbproj` |
| ExecuteCode bodies | `Blackbaud.AppFx.Galileo.ServiceRevisions\CodeRevisions<nnnn>.vb` |
| Master catalog specs | either `Blackbaud\AppFx\<Area>\Catalog\` or `Blackbaud.AppFx.<Area>.Catalog\` (both conventions exist) |

The **highest-numbered `DBREV<nnnn>.XML` is the current dev revision** — that's the one to append to.

Master spec locations worth remembering (specs are scattered and slow to find):

- `dbo.EVENT` → `Blackbaud.AppFx.EventManagement.Catalog\Event.Table.xml` (`3EF5D061-5692-40F3-B8AD-1367A8C6FAD5`)
- `dbo.TICKET` → `Blackbaud\AppFx\Programming\Catalog\Ticket.Table.xml` (`2a9da10f-43f0-4ffc-b523-eba3527ff26a`)
- `dbo.SALESORDERITEMTICKET` → `Blackbaud\AppFx\Programming\Catalog\SalesOrderItemTicket.Table.xml` (`1532e937-8080-45d1-b33d-ce153730a75b`)
- Daily-sales function specs → `Blackbaud\AppFx\DailySales\Catalog\DailySaleItem\`
- Revenue/fundraising form specs → `Blackbaud.AppFx.Fundraising.Catalog\`

Table specs use namespace `bb_appfx_table` (not `bb_appfx_tablespec`) and the attribute is `Tablename=`, not `DBTableName=`.

## Searching this repo

Ripgrep and glob **time out** on the full tree. Always scope to a specific subfolder and use narrow patterns. To discover area folder names, glob `Blackbaud/AppFx/*/Catalog/*.vbproj` rather than listing directories. Expect to hunt folder by folder; say so rather than appearing stuck.

## Step 1 — Read the changed spec's root element

Everything downstream comes from it:

- `ID="..."` → the `CatalogItemID` for the LoadSpec. **Copy it; never invent it.**
- Root element name → the `CatalogItemType`.

Common pairs:

| Root element | CatalogItemType |
|---|---|
| TableSpec | `TableSpec` |
| SQLFunctionSpec | `SQLFunctionSpec` |
| SQLStoredProcedureSpec | `SQLStoredProcedureSpec` |
| ViewDataFormTemplateSpec | `ViewDataFormTemplateSpec` |
| AddDataFormTemplateSpec | `AddDataFormTemplateSpec` |
| EditDataFormTemplateSpec | `EditDataFormTemplateSpec` |

## Step 2 — Decide ReloadOnly

- **`ReloadOnly="true"`** — the spec is already registered and only its SQL body changed. Regenerates the procedure without rebuilding catalog metadata. Standard for data form templates.
- **`ReloadOnly="false"`** — output fields, form-level attributes, security, installed products, or table structure changed, so catalog metadata must be rebuilt. Standard for TableSpec and SQLFunctionSpec.

Both patterns exist in the history for the same spec types, so don't infer it from a single precedent. If the change isn't obviously one or the other, **ask the user**: "SQL body only, or did output fields / form metadata change too?" It's one question and it prevents a wrong deploy.

## Step 3 — Index rules (only for TableSpec changes)

An Index element takes **no attributes at all** in this codebase. Only:

```xml
<Index>
	<IndexFields>
		<IndexField Name="PROGRAMID"/>
		<IndexField Name="STARTDATETIMEWITHOFFSET"/>
	</IndexFields>
	<IncludeFields>
		<IncludeField Name="ID"/>
	</IncludeFields>
</Index>
```

- **Name is generated as `IX_<TABLE>_<keycols joined by _>`. Include columns never appear in the name.** So `(EVENTID) INCLUDE (STATUSCODE, ISREFUNDED)` becomes `IX_TICKET_EVENTID`, *not* `IX_TICKET_EVENTID_STATUSCODE_ISREFUNDED`. Verified against `IX_EVENT_PACKAGEID` (has INCLUDE `ID`) and `IX_EVENT_ENDDATETIMEWITHOFFSET` (nine includes).
- **Filegroup is not an index attribute.** Generated indexes land on `IDXGROUP` automatically. `FileGroup=` on the TableSpec root controls *data* placement (e.g. `TRANGROUP`).
- **`ONLINE = ON` cannot be expressed.**
- **A "DROP INDEX X; CREATE INDEX Y" pair cannot be reproduced declaratively** if both key the same column. Changing an include list makes the loader drop and recreate the index, but it returns under the same generated name. Say so rather than claiming the drop is done.
- **Remove prefix-redundant indexes.** Adding `(A, B)` makes a standalone `(A)` index dead weight — delete it in the same revision. Precedent: revision 225 replaced the single-column `SALESORDERITEMTICKETID` index with the covering `(SALESORDERITEMTICKETID, PROGRAMID)`. Safe for FK columns as long as the composite keeps that column leading.

When the user supplies raw DDL and asks for "spec files, not a script", translate it and **state plainly which parts don't survive** — generated names, `ONLINE`, filtered predicates.

**Use an ExecuteSql revision instead** when the spec genuinely can't express it: filtered indexes (`WHERE ...`), `ONLINE = ON`, a specific index name, or a real `DROP` with no replacement. Those revisions need no snapshot and no vbproj change. Precedent: `DBREV1940.XML` ID 190.

## Step 4 — Pick the revision ID

Open the current `DBREV<nnnn>.XML`. IDs start at 100 (a Comment anchor) and **increment by 5**. Take the highest existing ID, add 5.

**Re-check the highest ID immediately before writing.** Teammates land revisions continuously — an ID that was free at the start of a conversation may be taken an hour later. Grep for `DBRevision ID=` and read the tail; don't trust an earlier reading.

## Step 5 — Create the snapshot

Name it `<ID>.<ExactMasterFileName>` in `...Specs<nnnn>\`, e.g. `285.RevenueTransactionPageData.View.xml`.

The snapshot must be a **verbatim copy of the master as of this revision**. Have the user run a real file copy:

```
copy /Y "<master path>" "<...Specs nnnn>\<ID>.<name>"
```

Do **not** reproduce the file by writing it out. Retyping silently normalises trailing whitespace, which is harmless between XML elements but **significant inside CDATA sections** — and that's where trigger bodies and stored-procedure bodies live. A `copy /Y` snapshot verifies byte-identical; a retyped one shows dozens of whitespace-only line differences.

Multiple revisions may snapshot the same spec at different points (`225.Ticket.Table.xml` and `245.Ticket.Table.xml` coexist). Older snapshots intentionally differ from the master; only the **latest** snapshot of a given spec must match it.

## Step 6 — Register the snapshot

In `...Specs<nnnn>.vbproj`, each file gets its **own ItemGroup**, as `EmbeddedResource` (catalog projects use `Content`; Specs projects use `EmbeddedResource`):

```xml
  <ItemGroup>
    <EmbeddedResource Include="285.RevenueTransactionPageData.View.xml" />
  </ItemGroup>
```

Insert before the `Microsoft.VisualBasic.targets` Import line at the end of the project.

## Step 7 — Add the manifest entry

Append to `DBREV<nnnn>.XML`, before the closing DBRevisions tag. Comment format is `BLACKBAUD\<user> <date> <Bug#/US#> | <what and why>`:

```xml
	<!-- BLACKBAUD\First.Last 10/5/2026 Bug#1234567 | Revenue transaction page view form: stored procedure body updated (USP_DATAFORMTEMPLATE_VIEW_REVENUETRANSACTIONPAGEDATA) -->
	<DBRevision ID="285">
		<LoadSpec Spec="285.RevenueTransactionPageData.View.xml" ReloadOnly="true"  CatalogItemType="ViewDataFormTemplateSpec" CatalogItemID="FC4E46D0-A7EC-4120-A7CD-9D36B9E63E06" />
	</DBRevision>
```

Ask for the work-item number and a one-line description of the fix. Some entries omit the number, so it's not fatal, but a specific description is what makes the manifest readable a year later.

Step forms: a LoadSpec element, an ExecuteSql element wrapping CDATA, or an ExecuteCode element naming a `Rev_NNN_...` method.

## Step 8 — Verify

1. **ID** unique, highest, +5 from the previous.
2. **CatalogItemID** matches the spec root `ID`; **CatalogItemType** matches the root element.
3. **Snapshot registered exactly once** in the Specs vbproj.
4. **Latest snapshot matches master.** Don't eyeball it. Grep both files for the same landmark pattern with line numbers and confirm they align:
   - data form / function specs: `^\s*(SPName=|<SPDataForm|</SPDataForm>|ID=)`
   - table specs: `Trigger Name=|</Indexes>|</TableSpec>`
   - Also compare non-blank line counts (grep `.` in count mode). Equal counts plus aligned anchors means byte parity; a constant offset from some point onward pinpoints where they diverged.
5. **Check the DB** for the resulting index shape and name (table specs only):

```sql
select t.name as TABLENAME, i.name as INDEXNAME, i.type_desc,
  stuff((select ', ' + c2.name from sys.index_columns ic2
         join sys.columns c2 on c2.object_id=ic2.object_id and c2.column_id=ic2.column_id
         where ic2.object_id=i.object_id and ic2.index_id=i.index_id and ic2.is_included_column=0
         order by ic2.key_ordinal for xml path('')),1,2,'') as KEYCOLS,
  stuff((select ', ' + c3.name from sys.index_columns ic3
         join sys.columns c3 on c3.object_id=ic3.object_id and c3.column_id=ic3.column_id
         where ic3.object_id=i.object_id and ic3.index_id=i.index_id and ic3.is_included_column=1
         order by c3.name for xml path('')),1,2,'') as INCLUDECOLS,
  i.filter_definition, fg.name as FILEGROUP
from sys.indexes i
join sys.tables t on t.object_id=i.object_id
left join sys.filegroups fg on fg.data_space_id=i.data_space_id
where t.name in ('<TABLE1>','<TABLE2>') and i.type > 0
order by t.name, i.name;
```

6. **Warn about duplicates.** If the DDL was already run by hand on a dev/sandbox DB, the hand-made index sits there under the script's name and the revision adds the spec-named one alongside — two identical indexes. Tell the user to drop the hand-made one there. Clean DBs are unaffected.
7. **Remind about `tf add`.** The snapshot is a *new* file; checking out the project does not pend it. If it isn't in Pending Changes at check-in, the manifest ships pointing at a file that doesn't exist.

## TFVC gotchas

Phoenix is TFVC, and files not checked out are **read-only** — writes fail with `EPERM: operation not permitted, rename '<file>.tmp...'`. When that happens:

- Name the exact files needing checkout and stop. Don't clear the read-only bit to force it through.
- The uppercase `.XML` on `DBREV<nnnn>.XML` is easy to miss when checking out by extension — call it out explicitly.
- An open Visual Studio editor tab can hold a lock **even after checkout**, giving the identical `EPERM` on a file whose siblings write fine. Ask the user to close the tab or the IDE; don't conclude the checkout failed.
- Checkouts don't survive between sessions, and teammates' check-ins reset files to read-only. Expect to ask again on a later day.
- Retry once after the user says they're done; locks often clear a moment later.

## Worked examples

**Raw DDL into table specs.** Requester supplied:

```sql
CREATE NONCLUSTERED INDEX IX_EVENT_PROGRAMID_STARTDATETIMEWITHOFFSET_ID
  ON dbo.EVENT (PROGRAMID, STARTDATETIMEWITHOFFSET) INCLUDE (ID) WITH (ONLINE = ON) ON [IDXGROUP];
DROP INDEX IX_TICKET_EVENTID ON dbo.TICKET;
CREATE NONCLUSTERED INDEX IX_TICKET_EVENTID_STATUSCODE_ISREFUNDED
  ON dbo.TICKET (EVENTID) INCLUDE (STATUSCODE, ISREFUNDED) WITH (ONLINE = ON) ON [IDXGROUP];
```

Delivered as IDs 240 (Event.Table.xml, adding the composite and deleting the redundant single-column `PROGRAMID` index), 245 (Ticket.Table.xml, adding `ISREFUNDED` to the `EVENTID` index's includes), 250 (the function spec that motivated them). Reported caveats: generated names differ from the script's, `ONLINE = ON` is not carried over, `IX_TICKET_EVENTID` still exists afterward, and the dev DB where the script had been run by hand needed its two hand-made indexes dropped.

**Changed spec into one revision.** User changed `RevenueTransactionPageData.View.xml` (a ViewDataFormTemplateSpec, SP body only). Delivered as ID 285: `copy /Y` snapshot, one EmbeddedResource ItemGroup, one LoadSpec with `ReloadOnly="true"`. Verified byte parity via anchor alignment (root `ID` line 4, SPDataForm line 24, closing SPDataForm line 1339, close tag line 1489 in both) and equal non-blank line counts (1490).

