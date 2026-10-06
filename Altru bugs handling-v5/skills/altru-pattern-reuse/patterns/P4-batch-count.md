# P-4: Batch Count Re-Fetch After Bulk Operation

## Source Bugs
**#3892344** — Gift batch summary showing stale count after record deletion  
**#3951102** — Constituent merge showing incorrect duplicate count post-merge  
**#3923498** — ClientUsers batch count not updated after dedup cleanup script

## Root Cause Pattern
A bulk operation (delete, merge, dedup, import) modifies records but the UI/API returns
a **cached or pre-computed count** that was captured before the operation completed.
The count is not re-queried from the database after the operation; it reflects the state
at the start of the batch, not the end.

This typically surfaces as:
- Summary page shows "N records" after a batch delete but the actual table has N-k rows
- A merge wizard reports "2 duplicates remaining" when 0 remain
- An import job's final confirmation step shows the wrong record count

## Trigger Keywords
`count`, `batch count`, `stale count`, `wrong count`, `incorrect total`,
`merge`, `dedup`, `bulk delete`, `import count`, `record count after`,
`summary shows`, `dashboard count`, `total not updated`, `cached count`

## Fix Pattern — Re-Query Count After Bulk Operation

### C# Service Layer (WebAPI / BBAPI pattern)

```csharp
// BEFORE fix — count captured before bulk delete
public BatchSummaryResult ProcessBatch(BatchDeleteRequest request)
{
    int countBefore = _repository.GetCount(request.Filter);  // stale if used after delete
    _repository.BulkDelete(request.Ids);

    return new BatchSummaryResult
    {
        RecordsProcessed = request.Ids.Count,
        RemainingCount   = countBefore - request.Ids.Count  // BUG: assumes perfect subtraction
    };
}

// AFTER fix — re-fetch count from DB after the operation completes
public BatchSummaryResult ProcessBatch(BatchDeleteRequest request)
{
    _repository.BulkDelete(request.Ids);

    // Re-query the live count — do not assume delta arithmetic is correct
    // (other processes may run concurrently, or the delete may be partial)
    int remainingCount = _repository.GetCount(request.Filter);

    return new BatchSummaryResult
    {
        RecordsProcessed = request.Ids.Count,
        RemainingCount   = remainingCount
    };
}
```

### SQL — Count after DML (use in stored procedures / scripts)

```sql
-- Execute the bulk operation
DELETE FROM dbo.<TABLE_NAME>
WHERE ID IN (SELECT ID FROM #ToDelete);

-- ALWAYS re-select the live count after DML — never trust @@ROWCOUNT alone
-- for counts shown in the UI (@@ROWCOUNT gives rows affected, not remaining)
SELECT COUNT(1) AS RemainingCount
FROM dbo.<TABLE_NAME>
WHERE <SAME_FILTER_AS_ORIGINAL_QUERY>;
```

### VB.NET Page Code-Behind (Altru WebForms pattern)

```vbnet
' BEFORE fix — count from ViewState or a cached property
Protected Sub btnRunBatch_Click(sender As Object, e As EventArgs)
    Dim svc As New BatchService()
    svc.DeleteRecords(GetSelectedIds())
    ' BUG: lblCount still shows ViewState value from page load
    lblCount.Text = ViewState("RecordCount").ToString()
End Sub

' AFTER fix — re-bind the count label from DB after the operation
Protected Sub btnRunBatch_Click(sender As Object, e As EventArgs)
    Dim svc As New BatchService()
    svc.DeleteRecords(GetSelectedIds())

    ' Re-query and rebind — never read count from ViewState after a write operation
    Dim freshCount As Integer = svc.GetLiveCount(GetCurrentFilter())
    lblCount.Text = freshCount.ToString()
    ViewState("RecordCount") = freshCount  ' keep ViewState in sync too
End Sub
```

## Checklist

- [ ] **After any bulk DML** — does the response/UI re-query the count from the DB?
  Never compute `newCount = oldCount - deletedCount`; re-select `COUNT(1)` from the live table
- [ ] **ViewState / cached properties** — after a write operation, ensure the count property
  is refreshed before it's read again (invalidate the cache or re-fetch inside the same request)
- [ ] **Concurrent writes** — if other processes write to the same table, arithmetic deltas
  are always wrong; only a post-operation SELECT is reliable
- [ ] **Stored procedures** — if a sproc returns a count output parameter, verify it
  is set AFTER the DML, not before (common mistake: `SET @Count = @@ROWCOUNT` before `DELETE`)
- [ ] **API responses** — if the API endpoint returns a count in its JSON response body,
  that count must come from a post-DML SELECT, not from the request payload

## Common Mistake Reference

| Anti-Pattern | Risk | Correct Approach |
|---|---|---|
| `return input.Ids.Count` after delete | Returns requested count, not actual deleted count | Return `@@ROWCOUNT` or re-query |
| `newCount = old - delta` | Wrong under concurrency or partial failure | Re-SELECT COUNT(1) post-DML |
| Read `ViewState["Count"]` after write | ViewState is stale after postback DML | Rebind from DB before reading |
| `@@ROWCOUNT` as "remaining count" | @@ROWCOUNT = rows *affected*, not rows *remaining* | SELECT COUNT(1) WHERE <filter> |
