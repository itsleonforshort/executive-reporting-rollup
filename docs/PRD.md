# Product Requirements Document
## Executive Reporting Rollup

**Version 1.3 — 2026-09-13. Written by Leo. This is the contract.**

Change in 1.3: no step changed. Three open items moved. Leo is the approver at step 15.
HubSpot and Supabase are parked at Leo’s instruction, so sources 1 and 2 cannot be built
yet. The three Google Sheets now exist. All changes are listed in section 13.

Nothing gets built that this file does not ask for. Nothing in this file changes without
Leo agreeing to the change first. Sections 1 to 11 are Leo's words. Section 12 holds the
open items found while reading it, and is the only part Claude may add to.

---

### 1. Objective

Automate the weekly executive reporting process by consolidating sales, usage, finance,
and support data into a single leadership brief.

The system should replace manual weekly deck preparation and give leadership consistent
visibility into business performance every Monday.

### 2. Trigger

**Schedule:** Every Monday at 07:00.

### 3. Data Sources

- HubSpot — Deals / pipeline data
- Supabase — Product usage
- Google Sheets — Finance actuals
- Google Sheets — Support log
- Google Sheets — Previous weekly snapshot

### 4. Workflow

1. Trigger workflow every Monday at 07:00.
2. Retrieve current deal and pipeline data from HubSpot.
3. Query usage metrics from Supabase.
4. Retrieve finance actuals from Google Sheets.
5. Retrieve support metrics from Google Sheets.
6. Retrieve the previous weekly snapshot.
7. Merge all datasets.
8. Normalize metrics into a consistent reporting structure.
9. Calculate:
   - Variance against target
   - Week-over-week changes
   - Pipeline movement
   - Revenue trend
10. Send structured metrics to OpenAI. One call per brief, never one per row.
    Credential: "OpenAI account" on the self-hosted instance, type openAiApi.
11. Generate a concise executive narrative highlighting:
   - Major changes
   - Wins
   - Risks
   - Exceptions
   - Metrics requiring attention
12. Generate:
   - Pipeline bar chart
   - Revenue trend line chart
13. Create a Google Docs executive brief containing metrics, narrative, and charts.
14. Export the report as PDF.
15. Send the PDF for human approval, and wait for the answer.
    Nothing reaches leadership until a person approves it.
16. Email the PDF to leadership.
17. Post a short summary to Telegram.
18. Append the current metrics to the weekly snapshot sheet.

### 5. Historical Data Requirement

Seed approximately **12 weeks of synthetic pipeline data** before launch so trend analysis
and week-over-week reporting have sufficient history.

### 6. Output

Each weekly run should produce:

- Executive summary
- KPI table
- Target variance
- Week-over-week comparison
- Pipeline chart
- Revenue trend chart
- Key risks and observations
- PDF leadership brief
- Telegram summary
- Historical snapshot entry

### 7. Reliability Requirements

The workflow is primarily read-only.

Failures in reporting must not modify or corrupt source systems.

If an individual source fails, the workflow should flag the missing dataset rather than
silently generating misleading results.

### 8. Estimated Complexity

Approximately **20 automation nodes**.

### 9. Success Criteria

- Report generated automatically every Monday.
- Leadership receives reporting without analyst intervention.
- Numbers remain consistent across source systems and the final report.
- Weekly trends are visible from historical snapshots.
- Narrative accurately reflects meaningful KPI changes.
- Reporting cadence improves from monthly/manual visibility to weekly automated
  visibility.

### 10. Business Impact

The automation replaces repetitive analyst reporting work while increasing reporting
frequency.

The primary value is not simply saving report-building time. It gives leadership a
reliable weekly operating rhythm and faster visibility into revenue, pipeline, usage,
finance, and support performance.

---

### 11. Out of scope

Read from section 4. Listed here so nobody adds them later by accident.

- No live dashboard. This is a push.
- No per-rep or per-person breakdown.
- No forecasting. The brief reports what happened.
- No data warehouse. We read what already exists.
- No second brief for a second audience.
- No write-back to HubSpot or Supabase. The only write is step 18, the snapshot sheet.

### 12. Open items

Claude maintains this section. A build may not start on an OPEN item.

| # | Item | Status |
|---|---|---|
| 1 | **Narrative model.** Gemini is out of quota on this instance (Google 429 to a single call, 2026-09-07). Leo chose OpenAI on 2026-09-13. Step 10 now names OpenAI. The Integrator confirms the exact model id and request body against live OpenAI docs before the Coder builds. | **CLOSED 2026-09-13 — step 10 unblocked** |
| 2 | The three Google Sheets: their file ids, tab names, and column headers | **PART DONE 2026-09-13.** Created by Claude on Leo’s instruction. Ids and headers in `docs/knowledge/data-sources.md`. Still open: the metric columns, and which Google account n8n can actually see them from |
| 3 | Supabase: project URL, table or view, and which fields are the usage metrics | **OPEN — PARKED by Leo 2026-09-13.** No Supabase credential exists on the instance and Leo asked not to add one yet. Source 2 cannot be built |
| 4 | HubSpot: which pipeline, which deal stages count as "in pipeline" | **OPEN — PARKED by Leo 2026-09-13.** No HubSpot credential exists on the instance and Leo asked not to add one yet. Source 1 cannot be built |
| 5 | The metric list. Which numbers appear in the KPI table, and their definitions | **OPEN** |
| 6 | Targets. Where the target for each metric is stored | **OPEN** |
| 7 | Email recipients for step 16 | **OPEN** |
| 8 | Telegram chat for step 17 | **OPEN** |
| 9 | Time zone for the Monday 07:00 trigger, and the exact week boundary | **OPEN** |
| 10 | **Approval.** Leo decided on 2026-09-13 that a human approves the brief before it is sent. This is now step 15. | **CLOSED 2026-09-13** |
| 11 | Chart rendering method. n8n has no native chart node. QuickChart is the likely route, to be verified by the Integrator | **OPEN** |
| 12 | PDF export route for step 14, to be verified by the Integrator | **OPEN** |
| 13 | **Who approves at step 15.** **Leo. Settled 2026-09-13.** The channel is still open: no Telegram credential exists on the instance, and Gmail does. | **PART DONE — channel still open** |
| 14 | **The reject path.** What the run does when the brief is rejected, and when nobody answers in time. Does it still append the snapshot row at step 18. | **OPEN — blocks step 15** |

### 13. Version history

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-09-13 | Claude's placeholder draft. Superseded the same day. |
| 1.0 | 2026-09-13 | Leo's PRD, in full. Out-of-scope and open-items sections added. |
| 1.1 | 2026-09-13 | Step 10 moved from Gemini to OpenAI. Leo decided. Gemini is out of quota here. |
| 1.2 | 2026-09-13 | Human approval added as step 15. Leo decided. Old steps 15 to 17 shifted down by one. |
| 1.3 | 2026-09-13 | Leo is the approver. HubSpot and Supabase parked by Leo. The three Google Sheets created. |
