# Metric definitions

Every number that appears in the brief has a block here. No block, no number.
`CLAUDE.md`: *"Never change a metric definition to make a number look better. Change it here
first, with the reason."*

---

## STATUS — these four are SAMPLES, not Leo's real metrics. 2026-09-14.

**Read this before you trust anything below.**

This file was empty on purpose until 2026-09-14. PRD §12 item 5 asked Leo which numbers leadership
actually wants each week. He answered **"just do sample"** — meaning: pick something sensible so
the workflow can be proven end to end, rather than stay blocked.

**So these four metrics are placeholders chosen by Claude, not a business decision by Leo.**

- **PRD §12 item 5 stays OPEN.** These do not close it.
- **Every seeded row in the Finance Actuals sheet is marked `synthetic = TRUE`** and carries a note
  saying it is sample data. `CLAUDE.md` forbids seeding a sheet the real reporting reads without
  marking it.
- **Before this goes live, Leo replaces these four with the real ones**, and the sample rows are
  deleted from the sheet.
- The numbers below are invented. They are shaped to exercise the brief — one metric comfortably
  ahead of target, one behind, and one badly off so the "risks" section has something to show.

---

## Why these four

They are the smallest set that exercises every part of the brief:

1. **Revenue** — required. Node 9 looks for `/revenue/i` in the metric name to build the revenue
   trend chart. **If no metric contains the word "revenue", PRD §9's trend chart cannot render.**
   Whatever Leo's real list turns out to be, one metric must have "revenue" in its name, or the
   chart logic needs changing.
2. **New Customers** — a growth number, counted not summed.
3. **Churned Customers** — a number where *higher is worse*. See the warning below.
4. **Weekly Active Users** — a usage number, the natural place for Supabase data to land later.

---

## Revenue

- **What it is:** recognised revenue for the seven days of the reporting week.
- **Unit:** currency, whole units.
- **Source:** Finance Actuals sheet, rows where `metric` is `Revenue`.
- **Target:** the `target` column on the same row.
- **Direction:** higher is better.
- **Feeds:** the KPI table, and the revenue trend chart (PRD §9).

## New Customers

- **What it is:** count of customers who first paid during the reporting week.
- **Unit:** whole number.
- **Source:** Finance Actuals sheet, rows where `metric` is `New Customers`.
- **Direction:** higher is better.

## Churned Customers

- **What it is:** count of paying customers lost during the reporting week.
- **Unit:** whole number.
- **Direction:** **LOWER IS BETTER.**

**A known limitation, recorded rather than hidden.** Node 9 computes variance as
`(actual - target) / target`, and node 12 flags anything more than 10% off target as a risk. It has
no idea which direction is good. So Churned Customers coming in *above* target shows as a positive
variance — `+80.0%` — which reads like good news and is the opposite of the truth.

**This is not a fault the sample introduced. It is a gap in the design that the sample exposed.**
Any real metric where lower is better has the same problem. Fixing it means adding a direction to
each metric definition and teaching node 9 and node 12 about it. **Owner: Designer. It must be
settled before real metrics go in.**

## Weekly Active Users

- **What it is:** distinct users active at least once during the reporting week.
- **Unit:** whole number.
- **Source:** Finance Actuals sheet today. Should move to Supabase once PRD §12 item 3 is answered.
- **Direction:** higher is better.

---

## The seeded rows

Three weeks, so that week-over-week has something to compare against whichever week a run computes.
`2026-W36` is the week the run of 2026-09-13 used; `2026-W37` is the week a Monday run computes.

All twelve rows carry `synthetic = TRUE`.

| week_label | metric | actual | target |
|---|---|---|---|
| 2026-W35 | Revenue | 48200 | 50000 |
| 2026-W35 | New Customers | 31 | 35 |
| 2026-W35 | Churned Customers | 7 | 5 |
| 2026-W35 | Weekly Active Users | 4120 | 4500 |
| 2026-W36 | Revenue | 52400 | 50000 |
| 2026-W36 | New Customers | 38 | 35 |
| 2026-W36 | Churned Customers | 9 | 5 |
| 2026-W36 | Weekly Active Users | 4380 | 4500 |
| 2026-W37 | Revenue | 51100 | 50000 |
| 2026-W37 | New Customers | 34 | 35 |
| 2026-W37 | Churned Customers | 12 | 5 |
| 2026-W37 | Weekly Active Users | 4510 | 4500 |

**What a W37 brief should say if the workflow is working correctly:**

- Revenue `51,100` against target `50,000` — `+2.2%` vs target, `-2.5%` week over week.
- New Customers `34` against `35` — `-2.9%` vs target, `-10.5%` week over week.
- Churned Customers `12` against `5` — `+140%` vs target. **Should appear in the risks list**, and
  is the case the direction gap above gets wrong.
- Weekly Active Users `4,510` against `4,500` — `+0.2%` vs target, `+3.0%` week over week.

**Use this table to check the brief.** If a number in the Doc does not match, the calculation is
wrong, not the sheet.

---

## Removing the samples later

1. Delete every row in Finance Actuals where `synthetic` is `TRUE`.
2. Delete any snapshot row written by a test run — node 20 hardcodes `synthetic: false`, so a test
   row is indistinguishable from a real one and will block the real run for that week (QA MINOR 2).
3. Replace the four blocks above with Leo's real metrics.
4. Keep one metric with "revenue" in its name, or change node 9's trend logic.
