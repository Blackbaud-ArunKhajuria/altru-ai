# P-2: SQL Query Performance / Execution-Plan Re-derivation

## Source Bugs
**#3926550** — Pledge balance stale — re-derive from live transactions  
**#3999227** — Export_DO_NOT_CALL_MAIL_SMS failing (7+ hours) — custom view + UFN_ADHOCQUERYIDSET_* TVF join  
**#4002374** — Direct Debit Batch commit performance  
**#3636223** — Generate DD Files / Batch commit times SEPA

## Root Cause Pattern
A query or process that worked previously degrades to multi-hour runtime due to one of:
- SQL Server choosing a suboptimal execution plan (often after LCE / statistics changes)
- A custom view (`USR_V_*`) joined against an OOB inline TVF (`UFN_ADHOCQUERYIDSET_*`)
- A denormalized/cached column that diverges from live transaction state

## Trigger Keywords
`slow`, `performance`, `timeout`, `export failing`, `batch commit times`, `8 hours`, `7 hours`,
`LCE`, `cardinality estimation`, `Legacy Cardinality`, `UFN_ADHOCQUERYIDSET`, `USR_V_`,
`commit times`, `taking too long`, `execution plan`, `query hint`, `MAXDOP`

## Diagnosis — Find the Slow Statement

```sql
-- Run during the slow process to identify the bottleneck
SELECT
    r.session_id,
    r.status,
    r.command,
    r.wait_type,
    r.wait_time        / 1000.0 AS wait_sec,
    r.total_elapsed_time / 1000.0 AS elapsed_sec,
    r.cpu_time         / 1000.0 AS cpu_sec,
    r.reads,
    DB_NAME(r.database_id) AS db_name,
    t.text AS sql_text,
    qp.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
OUTER APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.database_id = DB_ID()
ORDER BY r.total_elapsed_time DESC;
```

## Fix A — Materialise Custom View into #Temp + Index (STRONGEST FIX)

Use this when: a custom view (`USR_V_*`) is joined against a TVF like `UFN_ADHOCQUERYIDSET_*`.

```sql
-- BEFORE (slow — SQL Server cannot accurately estimate TVF join cardinality):
INSERT INTO #TMP_EXPORT...(ColA)
  SELECT v.[CONSTITUENTID]
  FROM   [dbo].[USR_V_QUERY_VELOCITY_CONS_CHANGED_SINCELASTEXPORT] v
  INNER  JOIN dbo.UFN_ADHOCQUERYIDSET_041D0C8B_...() ROOTMAP
          ON  ROOTMAP.[ID] = v.[CONSTITUENTKEY];

-- AFTER (fast — optimizer has real statistics on #temp):
-- Step 1: Materialise the custom view
SELECT CONSTITUENTID, CONSTITUENTKEY
INTO   #VelocityChanged
FROM   [dbo].[USR_V_QUERY_VELOCITY_CONS_CHANGED_SINCELASTEXPORT];

CREATE INDEX IX_VC_Key ON #VelocityChanged (CONSTITUENTKEY);  -- match the join column

-- Step 2: Join #temp against TVF (much smaller rowset)
INSERT INTO #TMP_EXPORT...(ColA)
  SELECT v.CONSTITUENTID
  FROM   #VelocityChanged v
  INNER  JOIN dbo.UFN_ADHOCQUERYIDSET_041D0C8B_...() ROOTMAP
          ON  ROOTMAP.[ID] = v.CONSTITUENTKEY;

DROP TABLE #VelocityChanged;
```

## Fix B — OPTION (RECOMPILE, MAXDOP) Hint

Use this when: the plan is stale but re-running with a fresh plan should be fast enough.

```sql
-- Add to the slow INSERT/SELECT statement:
SELECT ...
FROM   ...
WHERE  ...
OPTION (RECOMPILE, MAXDOP 4);    -- RECOMPILE: fresh plan; MAXDOP: limit parallelism
```

## Fix C — Re-derive from Live Transactions (from #3926550)

Use this when: a cached/denormalized column (e.g. PledgeBalance, OutstandingBalance)
diverges from reality. Never read the cached column — always re-derive:

```sql
-- Re-derive balance from live transaction table (instead of reading cached column)
SELECT
    p.PledgeID,
    p.Amount                          AS PledgeAmount,
    ISNULL(SUM(t.Amount), 0)          AS TotalPaid,
    p.Amount - ISNULL(SUM(t.Amount), 0) AS OutstandingBalance
FROM dbo.Pledges p
LEFT JOIN dbo.PledgeTransactions t
       ON t.PledgeID = p.PledgeID AND t.Posted = 1
WHERE p.PledgeID = @PledgeID
GROUP BY p.PledgeID, p.Amount;
```

## Fix D — LCE (Legacy Cardinality Estimation) Reset

Use this when: DBA changed the database compatibility level or CE version and performance dropped.

```sql
-- Check current CE level
SELECT compatibility_level FROM sys.databases WHERE name = DB_NAME();

-- Temporarily enable LCE for this session (test fix):
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;
-- Then re-run the failing process. If it completes, the root cause is CE.

-- Reset (discuss with DBA before making permanent):
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = OFF;
```

## Checklist

- [ ] Capture the slow statement from `sys.dm_exec_requests` while it is running
- [ ] Check if a `USR_V_*` custom view is involved — if yes, use Fix A (materialise)
- [ ] Check if statistics are stale: `UPDATE STATISTICS dbo.<table> WITH FULLSCAN`
- [ ] Try `OPTION (RECOMPILE)` first — if it fixes the plan, the issue is plan cache pollution
- [ ] For export processes: check if a custom stored procedure / view was recently modified
  (ask the client's customization partner — e.g. Zuri)
- [ ] Always test the fix on a lower environment with a representative data volume before prod

## Known Problematic Patterns in Altru/BBCRM

| Anti-pattern | Why it's slow | Fix |
|---|---|---|
| `USR_V_*` joined against `UFN_ADHOCQUERYIDSET_*` | TVF returns unknown row count; optimizer picks nested loop | Fix A: materialise view |
| Cached balance column read instead of transaction SUM | Stale data causes wrong results AND possible table scan | Fix C: re-derive |
| `INSERT INTO #TMP SELECT ... FROM big_view WHERE ID IN (TVF())` | TVF cardinality = 1 assumed | Fix A or Fix B |
| Process that ran in 1 min now takes 8+ hrs after DB maintenance | Plan cache invalidated + bad new plan | Fix B: OPTION(RECOMPILE) |
