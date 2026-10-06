# P-3: OAuth Scope / Authorization Boundary Guard

## Source Bugs
**#3632680** — PAT token operations blocked in Run-As-User (impersonation) sessions  
**#3617799** — ReportHost.aspx serving cached sensitive data (missing cache-control headers)  
**#4012957** — SKY API 403 Forbidden: DataRail via Workato (wrong OAuth scope — mrch.r vs altr.r)

## Root Cause Pattern
An API call, page render, or token operation fails or behaves incorrectly because:
1. The OAuth access token does not carry the scope required by the target endpoint, OR
2. A sensitive page/endpoint is served from browser/proxy cache instead of fresh from server, OR
3. A privileged operation (PAT token, admin action) executes inside an impersonation session
   where it should be blocked

## Trigger Keywords
`403`, `Forbidden`, `scope`, `OAuth`, `SKY API`, `Workato`, `DataRail`, `PAT token`,
`impersonation`, `Run As User`, `Run-As`, `access denied`, `cache`, `no-cache`,
`altr.r`, `mrch.r`, `insufficient scope`, `authorization`, `entitlement`

## Scenario A — SKY API 403 Wrong OAuth Scope

### Diagnosis Signal
```
HTTP 403 Forbidden
"does not contain the scope altr.r"
Token scope present: mrch.r (Merchant Services read)
Required scope: altr.r (Altru read)
```

### Fix — Re-register the app with correct scope
1. Go to `https://developer.blackbaud.com/apps/`
2. Select the integration app (e.g. DataRail / Workato app)
3. Edit → Scopes → **ADD**: Altru (altr) → Read (`altr.r`)
4. Re-authorize the integration (new OAuth consent flow required to get updated token)

### Scope Reference for Altru/BBCRM integrations
| Scope | Product | Access |
|---|---|---|
| `altr.r` | Altru | Read |
| `altr.w` | Altru | Write |
| `mrch.r` | Merchant Services | Read |
| `sky.r` | SKY API General | Read |

### Server-Side Scope Guard (C# — add to integration middleware)
```csharp
// Guard added to Altru API middleware — prevents future scope mismatches
// from reaching downstream processing and provides actionable error messages
if (!token.Scopes.Contains("altr.r"))
{
    return new HttpResponseMessage(HttpStatusCode.Forbidden)
    {
        ReasonPhrase =
            "Token missing required scope: altr.r. " +
            "Re-register the application with Altru read scope at " +
            "https://developer.blackbaud.com/apps/ and re-authorize."
    };
}
```

## Scenario B — PAT Token / Privileged Op in Impersonation Session

### Root Cause
`SessionContext.IsImpersonating` is true (Run As User mode) but PAT token creation/revocation
UI and API are still visible/callable. Security rule: privileged user-identity operations
must not be available during impersonation.

### Fix — Check impersonation state before showing controls / executing operation

```csharp
// In the code-behind (.aspx.cs) — hide controls when impersonating
protected void Page_Load(object sender, EventArgs e)
{
    bool isImpersonating = SessionContext.IsImpersonating;
    pnlCreatePAT.Visible = !isImpersonating;
    pnlRevokePAT.Visible  = !isImpersonating;
}

// In the API/service layer — enforce server-side (never trust client-side hide alone)
public IHttpActionResult CreatePATToken(CreatePATRequest request)
{
    if (CurrentSession.IsImpersonating)
    {
        return new StatusCodeResult(HttpStatusCode.Forbidden)
        {
            ReasonPhrase = "PAT token operations are not permitted during Run As User sessions."
        };
    }
    // ... proceed with PAT creation
}
```

## Scenario C — Sensitive Page Served from Cache (ReportHost.aspx pattern)

### Root Cause
A page serving sensitive/user-specific data (reports, financial data) is cached by the
browser or a proxy. Subsequent visits serve stale data from the previous user's session.

### Fix — Apply no-cache response headers in Page_Load or a base class

```csharp
// ReportHost.aspx.cs — add to Page_Load (or a shared SecurePage base class)
protected override void OnInit(EventArgs e)
{
    base.OnInit(e);
    SetNoCacheHeaders();
}

private void SetNoCacheHeaders()
{
    Response.Cache.SetNoStore();
    Response.Cache.SetCacheability(HttpCacheability.NoCache);
    Response.Cache.SetNoServerCaching();
    Response.Cache.SetExpires(DateTime.UtcNow.AddDays(-1));
    Response.AppendHeader("Pragma", "no-cache");
    Response.AppendHeader("Expires", "-1");
    Response.AppendHeader("Cache-Control", "no-cache, no-store, must-revalidate");
}
```

## Checklist

- [ ] **403 on SKY API?** — Check the token scope via `developer.blackbaud.com/apps/`
  Verify: required scope vs presented scope from the 403 message body
- [ ] **After scope change** — the integration MUST re-authorize (new consent flow)
  to receive an updated token containing the new scope
- [ ] **PAT / admin op accessible during impersonation?** — add both UI hide AND server-side check
  Never rely only on hiding the UI control — the API endpoint must also guard against impersonation
- [ ] **Sensitive page caching?** — Apply `SetNoStore()` + all five headers in `SetNoCacheHeaders()`
  Test by: log in, view page, log out, log in as different user — ensure no stale data shown
- [ ] **Document the scope matrix** — add to team wiki per integration:
  which scopes are needed for which integrations (avoids repeat mrch.r vs altr.r mistakes)
