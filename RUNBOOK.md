# Runbook — SupportEngineerChallenge

## Service overview
- **Service:** SupportEngineerChallenge.Api
- **Purpose:** Minimal task tracker (create + list tasks)
- **Data store:** SQLite (`app.db` in the API working directory)

## Common commands

**Run locally**
```bash
cd src/SupportEngineerChallenge.Api
dotnet run
```

**Run tests**
```bash
dotnet test
```

## Key endpoints
- `GET /api/tasks?userId={id}&limit={n}`
- `POST /api/tasks`

---

## Diagnosing issues in production

### Issue 1 — Create task fails with 500

**What to look at:**
Search logs for `CreateTask request` lines. The structured fields tell you everything:

```
CreateTask request UserId=user-001 Title=Buy groceries X-Client-Timestamp present=False length=0
System.FormatException: String '' was not recognized as a valid DateTime.
```

Key fields:
- `X-Client-Timestamp present=False` — the client did not send the header
- `length=0` — confirms empty string
- The `FormatException` stack trace pointing to `DateTime.Parse` confirms the crash site

**Root cause confirmed:** The `X-Client-Timestamp` header was optional in practice but the server called `DateTime.Parse("")` unconditionally, throwing before the request could be processed.

**Fix applied:** Replaced `DateTime.Parse` with `DateTime.TryParse` and a `DateTime.UtcNow` fallback. The header is now treated as optional.

**How to verify in production:**
1. Check that `POST /api/tasks` without the `X-Client-Timestamp` header returns `201 Created` (not 500).
2. Confirm no new `FormatException` lines in logs after deploy.
3. Run: `curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:<PORT>/api/tasks -H "Content-Type: application/json" -d '{"userId":"test","title":"probe"}'` — expect `201`.

---

### Issue 2 — Task list is slow for some users

**What to look at:**
Search logs for `ListTasks completed` lines with high `elapsedMs`:

```
ListTasks completed userId=user-001 limit=50  count=50  elapsedMs=312
ListTasks completed userId=user-002 limit=50  count=50  elapsedMs=289
ListTasks completed userId=user-015 limit=200 count=200 elapsedMs=1847
```

Key signals:
- Latency scales with the **total number of tasks in the database**, not just the user's tasks — because the previous code loaded every row before filtering.
- High `elapsedMs` across different `userId` values on the same instance points to a full-table scan, not a per-user data problem.

**Root cause confirmed:** The GET handler called `db.Tasks.ToListAsync()` (all rows) then filtered in application memory. With a large dataset (e.g. many seeded users), every request paid the full table-scan cost regardless of `userId` or `limit`.

**Fix applied:** Moved `Where`, `OrderByDescending`, `ThenByDescending`, and `Take` into the EF Core query so the database does the filtering and the result set is bounded before any data crosses the wire.

**How to verify in production:**
1. After deploy, `ListTasks completed` log lines should show `elapsedMs` in single or low double digits for typical user loads.
2. `elapsedMs` should no longer grow linearly with total task count — it should be stable across users.
3. Run the same user query twice; confirm response time is consistent.

---

### Issue 3 — Ordering weirdness / apparent duplicates after refresh

**What to look at:**
If tasks appear in a different order between refreshes, check whether multiple tasks share the same `CreatedAt` value. When the primary sort key is not unique, the database is free to return ties in any order — this manifests as items "jumping" positions.

**Root cause confirmed:** The list query sorted only by `CreatedAt DESC`. Tasks created in rapid succession (or with a client timestamp rounded to the same second) produced non-deterministic ordering.

**Fix applied:** Added `.ThenByDescending(t => t.Id)` as a stable tiebreaker — `Id` is an auto-incrementing integer so it uniquely orders rows with identical timestamps.

**Note on "duplicates":** No server-side duplication was found. If a user sees the same task twice, it is most likely a client-side rendering issue (stale state merged with a fresh fetch). This warrants follow-up investigation in the frontend.

---

## Verification steps (post-deploy)

1. **Create task without header** — `curl -X POST /api/tasks` with no `X-Client-Timestamp`. Expect `201`, not `500`.
2. **Create task with header** — Send with `X-Client-Timestamp: <ISO8601>`. Expect `201` and `CreatedAt` matching the supplied timestamp.
3. **List tasks** — `GET /api/tasks?userId=user-001&limit=50`. Expect `200` with only `user-001` tasks, ordered newest-first, stable across repeated requests.
4. **Run test suite** — `dotnet test` should pass with no failures.
5. **Check logs** — No `FormatException` or `System.Exception` lines after a round of manual smoke testing.

---

## Rollback / mitigation plan

| Scenario | Action |
|---|---|
| Fix introduces regression on task creation | Redeploy previous build; the old code path is preserved in git history |
| Slow queries persist after fix | Check whether a DB index on `UserId` is present (see follow-up ticket); run `EXPLAIN QUERY PLAN SELECT ...` against `app.db` |
| 500s continue after fix | Confirm the new binary is actually deployed; check for a different unhandled path (e.g. `req` body missing entirely) |
| Need to disable seeding temporarily | Set `Seed:Enabled=false` in `appsettings.json` and restart |
