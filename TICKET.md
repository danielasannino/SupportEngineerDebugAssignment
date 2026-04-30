# Follow-up Ticket

**Title:** Add database index on `Tasks(UserId)` to prevent full-table scans as data grows  
**Priority:** P2 — performance regression risk; not actively breaking but will degrade predictably  
**Owner:** Engineering

---

## Description

During the incident on 2026-04-29, the `GET /api/tasks` endpoint was fixed to push filtering and pagination into the database query (replacing an in-memory full-table scan). This immediately reduced latency, but without an index on the `UserId` column the database still does a sequential scan of the `Tasks` table for each request — it just returns fewer rows.

With a small dataset this is fine. As the number of tasks grows (more users, more seeded data, longer retention), query time will degrade linearly again. Adding a B-tree index on `UserId` makes the per-user lookup O(log n) regardless of total table size.

This should be delivered as an EF Core migration so it applies automatically on startup and is tracked in source control.

---

## Acceptance criteria

- [ ] An EF Core migration exists that adds an index on `Tasks.UserId`
- [ ] The migration is applied automatically via `db.Database.EnsureCreated()` or `Migrate()` on startup (no manual step required)
- [ ] `GET /api/tasks?userId=X` query plan (verified via `EXPLAIN QUERY PLAN` on SQLite) shows an index scan, not a full-table scan, when the Tasks table has > 10,000 rows
- [ ] `dotnet test` continues to pass after the migration is added
- [ ] No change to API response shape or semantics

---

## Notes / context

- **Relevant code:** `AppDbContext.OnModelCreating` in `src/SupportEngineerChallenge.Api/Data/AppDbContext.cs` — add `.HasIndex(x => x.UserId)` to the entity configuration
- **Why not a composite index:** `UserId` alone is the right starting point. A composite index (`UserId`, `CreatedAt`) could further speed up the `ORDER BY CreatedAt DESC` step, but measure first — premature composite indexes add write overhead and may not be needed at current scale
- **Monitoring:** After the index is deployed, compare `ListTasks completed elapsedMs` log lines before and after for a high-volume user; expect sub-10ms on typical page sizes
- **Migration strategy:** If the app is ever moved from SQLite to Postgres or another engine, the index syntax is the same in EF Core — no migration changes needed
- **Related incident:** INCIDENT.md (2026-04-29) — slow list confirmed by `artifacts/sample_slow_list_log.txt`
