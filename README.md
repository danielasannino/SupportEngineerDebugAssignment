
Issues confirmed and fixed:

Create-task 500 (confirmed) — POST /api/tasks threw System.FormatException when the X-Client-Timestamp header was absent. Fixed by replacing DateTime.Parse with DateTime.TryParse and a DateTime.UtcNow fallback, and moving input validation before timestamp parsing.

Slow task list (confirmed) — The list endpoint loaded every row in the Tasks table into memory before filtering by userId. Fixed by pushing Where, OrderBy, and Take into the EF Core query so the database does the work.

Ordering instability (confirmed) — Tasks with identical CreatedAt values sorted non-deterministically. Fixed by adding .ThenByDescending(t => t.Id) as a stable tiebreaker.

Test suite not compiling (fixed) — Missing using Xunit; directive in TaskApiTests.cs prevented the test file from compiling.

How I used logs to diagnose:
artifacts/sample_api_log.txt showed a structured log line immediately before the exception: X-Client-Timestamp present=False length=0. That single field told me the header was absent, not malformed — which meant the fix was a fallback, not stricter validation. The FormatException stack trace then confirmed the exact crash site (TaskEndpoints.cs line 41). I did not need to reproduce the bug locally; the log was sufficient to identify the root cause and write the fix.

artifacts/sample_slow_list_log.txt showed elapsedMs scaling with limit and implicitly with total data volume (1847ms for user-015 vs ~300ms for earlier users). That pointed to a full-table scan rather than a per-user problem.

Tradeoffs:

The timestamp fallback to UtcNow silently accepts requests without the header rather than returning a 400. This preserves compatibility with clients that don't send it. If the timestamp is semantically meaningful (e.g. for deduplication), a 400 would be safer — noted in the incident follow-up.
The ThenByDescending(t.Id) tiebreaker stabilises ordering but doesn't investigate the client-side "duplicate" report. I flagged it as a likely frontend state issue in the incident doc.
What I'd do next with more time:

Add the Tasks(UserId) database index (tracked in TICKET.md) — the query fix prevents the full-scan but without an index performance will degrade again as data grows.
Add a test case for POST /api/tasks without the X-Client-Timestamp header to lock in the fix.
Investigate the frontend rendering for the "duplicates" report — no server-side duplication was found but the symptom hasn't been fully ruled out.


# Support Engineer Challenge — Debug & Stabilize

This repo contains a small prebuilt app with a few realistic "production" issues. Your goal is to **triage**, **diagnose**, and **ship safe fixes** with clear communication.

## Scenario

Assume this app is running in production and you're on-call. We’ve received customer reports:

1) “Sometimes creating a task fails with a 500.”
2) “The tasks list is slow for some users.”
3) “We’ve seen tasks appear duplicated or out of order after refresh.”
4) "We're seeing 500 errors on task creation in production but can't reproduce locally. A log snippet is in **artifacts/sample_api_log.txt** — use the logs to identify the cause and fix it."

Not all reports may be accurate — part of the exercise is determining what’s real, what’s reproducible, and what you’d do next.

## What’s here

- `src/SupportEngineerChallenge.Api` — .NET 8 minimal API + SQLite + static UI
- `tests/SupportEngineerChallenge.Tests` — xUnit tests (some may fail)
- `RUNBOOK.md` — **you will update**
- `INCIDENT.md` — **you will fill in**
- `TICKET.md` — **you will write a follow-up ticket**
- `artifacts/` — sample logs + incident report context

## Timebox

Aim for ~2–4 hours. If you go beyond that, please note it.

## Getting started

Requirements:
- .NET SDK 8.x

Run the API + UI:

```bash
cd src/SupportEngineerChallenge.Api
dotnet restore
dotnet run
```

Then open:
- API Swagger: http://localhost:5088/swagger
- UI:         http://localhost:5088/

Run tests:

```bash
dotnet test
```

## Deliverables

Please complete:

1. **Triage & reproduction**  
   - Use the provided log artifacts (e.g. `artifacts/sample_api_log.txt`) to diagnose at least one issue
   - Clear repro steps for each confirmed issue
   - Capture evidence (logs, stack traces, etc.)

2. **Root cause analysis**  
   - Explain what’s happening and why (briefly)

3. **Fixes**  
   - Safe, minimal fixes
   - Add/update tests where appropriate

4. **Operational thinking**  
   - Update `RUNBOOK.md` with diagnosis + verification + rollback/mitigation steps

5. **Incident summary**  
   - Fill in `INCIDENT.md` with impact/timeline/root cause/fix/follow-ups

6. **Follow-up ticket**  
   - Write `TICKET.md` with a high-quality ticket (acceptance criteria, priority, etc.)

## Notes

- You can change anything in this repo (including UI) as long as you explain your choices.
- If you get blocked by setup, write down what you tried and where you got stuck.
