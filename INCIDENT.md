# Incident Summary

**Title:** Create-task endpoint throws 500 when `X-Client-Timestamp` header is absent  
**Date:** 2026-04-29  
**Severity:** P1 — task creation broken for any client not sending the optional header

---

## Impact

- **Users affected:** Any client that omits the `X-Client-Timestamp` request header when calling `POST /api/tasks` — this includes mobile clients, third-party integrations, and some browser versions that strip custom headers.
- **Symptoms:** `POST /api/tasks` returned HTTP 500. Customers reported "sometimes clicking Add just errors" — intermittent because some clients sent the header and some did not.
- **Secondary issue:** `GET /api/tasks` was also degraded at scale — requests for users with large datasets took 1–2 seconds due to a full-table scan (see follow-up ticket).

---

## Detection

- Customer report: *"I click Add and sometimes it just errors. If I try again a few seconds later it works."* (intermittent 500)
- Engineering provided `artifacts/sample_api_log.txt` — a log snippet from a failed production request that could not be reproduced locally.
- Log analysis confirmed the issue (see Root cause below).

---

## Timeline (UTC)

| Time | Event |
|---|---|
| ~T+0 | Customer report received: intermittent 500 on task creation |
| ~T+10m | Log artifact (`sample_api_log.txt`) reviewed; `FormatException` identified at `DateTime.Parse` |
| ~T+15m | Root cause confirmed: `X-Client-Timestamp present=False` in structured log precedes the exception |
| ~T+30m | Fix implemented: `DateTime.TryParse` with `UtcNow` fallback; validation reordered |
| ~T+45m | Secondary issue identified from `sample_slow_list_log.txt`: full-table scan in list endpoint |
| ~T+60m | Both fixes applied, build clean, tests passing |

---

## Root cause

**Primary (500 on create):**
`TaskEndpoints.cs` called `DateTime.Parse(clientTimestamp)` unconditionally before any input validation. When the `X-Client-Timestamp` header was absent, `StringValues.ToString()` returned `""`, and `DateTime.Parse("")` threw `System.FormatException`. The exception was unhandled and propagated as a 500.

The intermittent nature of the bug (works on retry) was because some clients sent the header and some did not — not a transient infrastructure issue.

**Diagnosed from logs:**
```
CreateTask request UserId=user-001 Title=Buy groceries X-Client-Timestamp present=False length=0
System.FormatException: String '' was not recognized as a valid DateTime.
   at System.DateTime.Parse(String s)
   at TaskEndpoints.cs:line 44
```
The `present=False` field in the structured log directly identified the missing header as the trigger.

**Secondary (slow list):**
`GET /api/tasks` loaded all rows from the `Tasks` table into memory (`ToListAsync()`) before filtering by `userId` in application code. With a large seeded dataset (many users × many tasks), every request paid the full table-scan cost. Confirmed by `sample_slow_list_log.txt`: user-015 with `limit=200` took 1847ms vs ~300ms for earlier users — latency scaling with total data volume, not the requested user's row count.

---

## Mitigation / resolution

1. **Create-task 500:** Replaced `DateTime.Parse(clientTimestamp)` with `DateTime.TryParse(..., out var parsed) ? parsed : DateTime.UtcNow`. Moved input validation (`req`, `UserId`, `Title` null/blank checks) to run **before** timestamp parsing. Added `req is null` guard to address compiler nullable warning.

2. **Slow list:** Replaced the in-memory filter pattern with a single EF Core query: `.Where(t => t.UserId == userId).OrderByDescending(t => t.CreatedAt).ThenByDescending(t => t.Id).Take(...)`. The database now does all filtering; only the requested rows are returned.

3. **Ordering stability:** Added `.ThenByDescending(t => t.Id)` to ensure a stable, deterministic sort when tasks share the same `CreatedAt` value.

4. **Test infrastructure:** Fixed `FluentAssertions` package reference (version `8.9.0` does not exist → `6.12.0`) and added the missing `using Xunit;` directive in `TaskApiTests.cs` so the test suite can compile and execute.

---

## Verification

- `dotnet build` — 0 errors, 0 warnings.
- `dotnet test` — both tests pass (create returns 201, list returns only requested user's tasks).
- Manual smoke test: `POST /api/tasks` without `X-Client-Timestamp` returns `201 Created`.
- Manual smoke test: `GET /api/tasks?userId=user-001&limit=50` returns correct, stably-ordered results.

---

## Follow-ups / action items

- [ ] Add a database index on `Tasks(UserId)` — the query-level fix prevents the full-table scan but a proper index will keep performance stable as data grows (see `TICKET.md`)
- [ ] Investigate client-side "duplicate" rendering — no server-side duplication found; the stable sort may resolve it, but frontend state management should be reviewed
- [ ] Add monitoring/alerting on 5xx rate for `POST /api/tasks` so future regressions are caught before customer reports
- [ ] Consider whether `X-Client-Timestamp` should be formally documented as optional, or validated and rejected (400) if present but malformed
