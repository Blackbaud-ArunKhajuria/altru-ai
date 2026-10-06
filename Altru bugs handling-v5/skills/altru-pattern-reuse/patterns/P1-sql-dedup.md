# P-1: SQL Dedup via ROW_NUMBER + Date-Ranged DELETE

## Source Bug
**#3923498** — dbo.ClientUsers duplicate users after data migration  
**#3559978** — dbo.ADDRESS AddressFinder duplicate addresses (DoNotMail flag wrong)

## Root Cause Pattern
A bulk process (AddressFinder, data migration, batch import) inserts duplicate rows into a
constituent-linked table (dbo.ADDRESS, dbo.ClientUsers, dbo.Constituents, etc.).
The duplicate rows differ by a flag or secondary attribute (DONOTMAIL, ISPRIMARY, UserTypeID).
The defect window is scoped by a known date range (when the batch process ran).

## Trigger Keywords
`duplicate`, `dedup`, `AddressFinder`, `bulk import`, `migration`, `DoNotMail`,
`SCRIPT FIX`, `Client Specific`, duplicate records, duplicate address, duplicate users

## Reusable SQL Template

```sql
-- ============================================================
-- P-1 SQL DEDUP TEMPLATE
-- Replace: TABLE_NAME, KEY_COLUMN, PARTITION_COLUMNS, DATE_COLUMN, PROCESS_FILTER
-- ============================================================

-- STEP 1: Identify duplicates (ALWAYS run SELECT first — share with client before DELETE)
SELECT
    ID,
    CONSTITUENTID,
    -- include the key columns that define "duplicate" for this table
    <PARTITION_COLUMNS>,
    <DATE_COLUMN>,
    ROW_NUMBER() OVER (
        PARTITION BY <PARTITION_COLUMNS>
        ORDER BY <DATE_COLUMN> DESC    -- keep the LATEST record, delete earlier ones
    ) AS rn
INTO #DupRows
FROM dbo.<TABLE_NAME>
WHERE <DATE_COLUMN> BETWEEN '<start_date>' AND '<end_date>'   -- scope to defect window
  AND CHANGEDBYID IN (
      SELECT ID FROM dbo.CHANGEAGENT
      WHERE NAME LIKE '%<PROCESS_NAME>%'    -- e.g. '%AddressFinder%', '%BatchImport%'
  );

-- STEP 2: Preview — share this result with the client / support before proceeding
SELECT * FROM #DupRows WHERE rn > 1;
-- Expected: only the rows you intend to delete. Verify count with client.

-- STEP 3: Snapshot (run on PROD before deleting)
-- SELECT * INTO dbo.<TABLE_NAME>_Backup_<YYYYMMDD> FROM dbo.<TABLE_NAME>
-- WHERE ID IN (SELECT ID FROM #DupRows WHERE rn > 1);

-- STEP 4: Delete duplicates
-- Run on LOWER ENV first (dev / test / staging). Confirm results. Then run on PROD.
DELETE FROM dbo.<TABLE_NAME>
WHERE ID IN (SELECT ID FROM #DupRows WHERE rn > 1);

-- STEP 5: Verify
SELECT COUNT(*) AS RemainingDuplicates
FROM (
    SELECT <PARTITION_COLUMNS>,
           COUNT(*) AS cnt
    FROM dbo.<TABLE_NAME>
    WHERE <DATE_COLUMN> BETWEEN '<start_date>' AND '<end_date>'
    GROUP BY <PARTITION_COLUMNS>
    HAVING COUNT(*) > 1
) x;
-- Expected: 0
```

## dbo.ADDRESS Specific Template (from #3559978)

```sql
-- AddressFinder duplicate address fix
SELECT
    ID, CONSTITUENTID, ADDRESSBLOCK, CITY, POSTCODE, STATEID, COUNTRYID,
    DONOTMAIL, ISPRIMARY, DATEADDED, DATECHANGED,
    ROW_NUMBER() OVER (
        PARTITION BY CONSTITUENTID, ADDRESSBLOCK, POSTCODE
        ORDER BY DATEADDED DESC
    ) AS rn
INTO #DupAddresses
FROM dbo.ADDRESS
WHERE DATEADDED BETWEEN '<defect_start>' AND '<defect_end>'
  AND CHANGEDBYID IN (
      SELECT ID FROM dbo.CHANGEAGENT WHERE NAME LIKE '%AddressFinder%'
  );

SELECT * FROM #DupAddresses WHERE rn > 1;  -- share with client

DELETE dbo.ADDRESS WHERE ID IN (SELECT ID FROM #DupAddresses WHERE rn > 1);
```

## dbo.ClientUsers Specific Template (from #3923498)

```sql
-- ClientUsers dedup — partition by SiteID + Email + UserTypeID (matches IX_ClientUsers constraint)
SELECT ClientUserID,
       ROW_NUMBER() OVER (
           PARTITION BY SiteID, Email, UserTypeID
           ORDER BY ClientUserID DESC
       ) AS rn
INTO #DupUsers
FROM dbo.ClientUsers
WHERE ConstituentID IN (
    SELECT ConstituentID FROM dbo.Constituents
    WHERE LookupID IN (<comma_separated_lookupids>)
);

DELETE FROM dbo.ClientUsers
WHERE ClientUserID IN (SELECT ClientUserID FROM #DupUsers WHERE rn > 1);
```

## Critical Checklist

- [ ] Always **snapshot** the table before running DELETE in production
- [ ] Use a **temp table staging** approach when a CHECK constraint blocks direct DELETE/UPDATE
  (e.g. `CK_ADDRESS_FORMERADDRESSCANNOTBEPRIMARY` on dbo.ADDRESS)
- [ ] Scope `DATEADDED` / `DATECHANGED` **tightly** — over-broad ranges risk deleting legitimate records
- [ ] Share the SELECT preview (Step 2) with client/support for sign-off before deleting
- [ ] Run on lower environment first (dev → test/staging → prod)
- [ ] Check for **related batches** — if AddressFinder ran as batch 198, 203, 207, 209,
  scope to all batch numbers, not just the first one you identified

## Common Pitfalls

- Constraint errors like `CK_ADDRESS_FORMERADDRESSCANNOTBEPRIMARY`: stage to `#temp` first,
  modify the flag, then delete — avoids constraint validation on the live table.
- Client says "still seeing duplicates after script": check if the defect date range is too narrow.
  Widen by 1–2 days and re-run the SELECT preview.
- Script deletes too many rows: ensure PARTITION BY columns match the business definition of
  "duplicate" for that specific table (not just any two rows with the same constituent).
