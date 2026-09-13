# QA Report — v2

**Date:** 2026-09-13
**Workflow:** `iIypy4KWtaqsowAQ`, "Executive Reporting Rollup - Weekly Leadership Brief"
**Instance:** self-hosted, `https://<your-n8n-instance>`
**Audited against:** the 34 acceptance criteria in `docs/reports/qa-v1.md`, with criterion 4 amended and MAJOR 1 withdrawn per the CORRECTION at the end of that file.
**Also read:** `CLAUDE.md`, `docs/state/handoffs/2026-09-13-23-20.md`, `docs/PRD.md` v1.3, `docs/architecture/v1-architecture.md`, `docs/reports/coder-v1.md` (§13 "Fixes after QA v1" is present), `docs/knowledge/n8n-gotchas.md`, `data-sources.md`, `money-facts.md`.
**Author:** QA agent. Written to disk by the main session — the agent has no write tool.

**Method.** Live `n8n_get_workflow` — `mode: structure`, then `mode: filtered` in six batches covering all 25 nodes, then `mode: details` for `settings` and execution stats. `n8n_validate_workflow` on the id, `profile: strict`. `get_node` schema reads against `nodes-base.hubspot` (operator, propertyName) and `nodes-base.telegram` (appendAttribution). n8n's own documentation re-read for the `alwaysOutputData` node setting. The `connections` object read by eye and compared line by line to architecture section 3. Nothing audited from memory. Nothing changed. Nothing run. Node 11 confirmed `disabled: true` and untouched.

**What changed since pass 1.** The Coder applied fixes in three batches via `n8n_update_partial_workflow`. `versionCounter` is 11, `updatedAt` 2026-09-13T15:16:09Z. Wiring is unchanged: 20 connection entries, identical to pass 1. `triggerCount: 0`, `totalExecutions: 0`, `active: false`, `pinData: null` — still never run.

**Validation.** `valid: true`, **0 errors**, 25 nodes, 24 enabled, 2 triggers, **26 valid connections, 0 invalid**, 26 expressions validated. Under the `strict` profile: 22 warnings, every one in a category already triaged in pass 1 — the `range` / A1-notation family on nodes 5, 6, 7, 20 (MINOR 4, inert, `dataLocationOnSheet` governs), "Code nodes can throw" on 9 and 12 (deliberate, O2), "Hardcoded nodeCredentialType" on 13 (correct usage), the disabled-node connection on 10→11 (required), the two known false-positive `$`-prefix warnings on 13 and 25, and three "no error handling" notices on 13, 15 and 20 that are all cases where `onError: stopWorkflow` is set on purpose. Under `runtime` the count is 3, as the main session reported. **No new warning class appeared.**

---

## Verdict

**PASS — fit to hand to the Tester for a limited first pass.**

**Still open: 1 blocker, 6 majors, 5 minors.**

The one blocker is **BLOCKER 4 — no Telegram credential**, which is Leo's and was always going to be. **Zero blockers remain that the Coder owns.** All three code blockers are fixed and verified live. One new MAJOR was introduced by the BLOCKER 3 fix (NEW-1 below); it does not block the Tester, and the correct sequence is to test first and fix after, for the reason given in that section.

**What the Tester can actually do today:** criteria 10, 11 and 22, plus the deliberate-breakage cases 20 and 21. That is a small list but it is the highest-value one available — it proves Lock 1 (the Merge behaviour that underpins the one-paid-call guarantee), it records the real empty-item and error-item shapes that three other criteria depend on, and it proves the paid call is never reached. It costs nothing. Everything from criterion 12 onward is gated on Finance Actuals holding metric rows, and everything from 16 onward is gated on the Telegram credential.

**Four fixes verified by the main session, re-confirmed by me against live:** node 16's `leftValue`, node 21's `approved` read, node 12's two `.first()` calls with zero `.item`, and `alwaysOutputData: true` on all five fetch nodes. All four are on the live workflow.

---

## The 34 criteria

| # | Criterion (short) | Result | Evidence |
|---|---|---|---|
| 1 | N16 + N21 read approval at the emitted path, strict `=== true` | **PASS** | Both read `.item.json.data?.approved ?? .item.json.approved`, N16 wrapped in `=== true` against `boolean/equals`, `typeValidation: strict` |
| 2 | N12 has zero `.item`; both cross-node reads use `.first()` | **PASS** | `$('Code - Normalise And Calculate Metrics').first().json` and `$('OpenAI - Write Executive Narrative').first().json`; whole `jsCode` read, no `.item` present |
| 3 | N3–N7 carry `alwaysOutputData: true` with `onError: continueRegularOutput` | **PASS** | All five carry both, plus `retryOnFail: true, maxTries: 3, waitBetweenTries: 5000` |
| 4 | *(amended)* N3 properties at `additionalFields.properties`; filter `type`/`operator` a schema-offered pair | **PASS** | `additionalFields.properties` holds six names; both filters are `type: "number"` with `GTE` / `LT`, and the schema gates those operators on `showWhen: { type: ["number"] }` |
| 5 | Node 11 still `disabled: true` | **PASS** | `"disabled": true` on `n11`, credential attached but node inert |
| 6 | N17 `sendTo` is Leo's own address | **FAIL** | Live value is `SET_ME_leadership_email_list` — see detail |
| 7 | N20 targets the snapshot doc/gid, `defineBelow`, six columns in order | **PASS** | doc `1TdTEZ…VElA`, gid `1425370562`, `mappingMode: defineBelow`, keys `week_label, week_start, week_end, run_at, synthetic, notes` |
| 8 | No `executionTimeout`; `errorWorkflow` is itself | **PASS** | `settings` holds 7 keys, none named `executionTimeout`; `errorWorkflow: "iIypy4KWtaqsowAQ"`; `timezone: Europe/London` |
| 9 | No Split In Batches / Split Out / Item Lists / Loop Over Items / Sub-workflow | **PASS** | All 25 `type` values read; none is a fan-out or sub-workflow node |
| 10 | Merge receives 5 inputs, N9 outputs exactly 1 item | **BLOCKED** | Zero executions. Structurally sound: N9 is `runOnceForAllItems` with a single `return [{json:{…}}]` |
| 11 | N9 classifies all five correctly; item shapes pasted into gotchas | **BLOCKED** | Needs a run. **Expected result changed by NEW-1 — read that section before testing** |
| 12 | With 2+2 metric rows, `readyToBuild` true, run reaches N12 | **BLOCKED** | Finance Actuals has no metric rows; PRD §12 item 5 OPEN |
| 13 | N12 produces `html`, `telegramSummary`, `weekLabel` and does not throw | **BLOCKED** | Gated on 12. All three keys are in N12's `return` |
| 14 | N13 returns a Doc id and both charts render inside it | **BLOCKED** | The design's riskiest unproven assumption; flagged on N13's notes |
| 15 | N14 returns binary `data` holding a PDF that opens | **BLOCKED** | `options.binaryPropertyName: "data"`, `docsToFormat: application/pdf` set |
| 16 | N15 sends exactly one approval, execution enters waiting | **BLOCKED** | No Telegram credential; `chatId: SET_ME_telegram_chat_id` |
| 17 | On Approve, N16 true branch, N17 attaches binary `data`, email arrives | **BLOCKED** | Gated on 16 and on criterion 6 |
| 18 | Sheet gains exactly one row with the six correct values | **BLOCKED** | Gated on 17 |
| 19 | Re-run same week gives `readyToBuild: false` naming the existing row | **BLOCKED** | Logic present in N9 `alreadyAppended` → `blockReason` |
| 20 | Broken sheet id: run continues, warning banner everywhere, no row | **BLOCKED** | Testable today without Telegram, up to node 13 |
| 21 | Header-only sheet classified `ok: true, rowCount: 0`, no warning | **BLOCKED — will fail as written** | NEW-1. **Criterion amended below** |
| 22 | Break Finance: `readyToBuild` false, N11 never reached, N22 fires | **BLOCKED** | This is the default state today; N22 send needs Telegram |
| 23 | Decline: nothing sent, N21 says **declined** | **BLOCKED** | Wording logic present: `approved === false` → "The brief was declined." |
| 24 | Expire: nothing sent, N21 says **expired**; expiry item recorded | **BLOCKED** | Falls to "Nobody responded within the 6-hour approval window." |
| 25 | Force N13 to fail: Error Trigger fires, N25 message **arrives** | **BLOCKED** | No Telegram credential — the message cannot arrive |
| 26 | No test row left with `synthetic` FALSE | **BLOCKED** | Tester protocol; N20 hardcodes `synthetic: false` (MINOR 2) |
| 27 | Telegram credential exists on N15, 18, 21, 22, 25; real `chatId` | **FAIL** | No credential on the instance; all five carry `SET_ME_telegram_chat_id` and no `credentials` block |
| 28 | N17 `sendTo` is the real leadership list | **BLOCKED** | Gated on 17; PRD §12 item 7 OPEN |
| 29 | N13 body names a real Drive folder parent, id recorded | **FAIL** | Metadata part is `{name, mimeType}` only — no `parents` key |
| 30 | `metric-definitions.md` complete, Finance Actuals populated | **FAIL** | PRD §12 item 5 OPEN; the file is empty on purpose |
| 31 | PRD §12 item 9 closed; node 2 matches | **FAIL** | Item 9 still OPEN. N2's notes now name all five `Europe/London` sites (MINOR 3 satisfied) |
| 32 | Real HubSpot business filter replaces `hs_lastmodifieddate` | **FAIL** | Still the placeholder; PRD §12 item 4 OPEN and PARKED |
| 33 | N11 enabled, one call, cost logged, exactly 1 in / 1 out | **BLOCKED** | Node disabled by rule; zero money spent on this project |
| 34 | N12 reads the narrative from the field proven in 33 | **BLOCKED** | Still four guessed names — correctly deferred until 33 runs |

**Group A: 8 of 9 PASS.** Groups B and C are entirely blocked on execution. Group D is blocked on Leo.

---

## Detail — FAIL rows

### Criterion 6 — node 17 `sendTo` is still the placeholder

`Gmail - Send Brief To Leadership`, `parameters.sendTo`:

```
"sendTo": "SET_ME_leadership_email_list"
```

**This is the correct and safe state today, and I am recording it as a FAIL only because the criterion is worded as a pre-run gate.** Gmail rejects a string with no `@`, so the node cannot send to anyone by accident. It must be changed to Leo's own address in the same sitting as the first run that can reach node 17, and not before — an address sitting there while the workflow is being broken on purpose is how a half-built brief reaches a real inbox. **Owner: Coder, on the Tester's instruction, immediately before criterion 17.** Non-blocking now.

### Criterion 27 — no Telegram credential (BLOCKER 4, unchanged)

Nodes 15, 18, 21, 22 and 25 all carry `chatId: "SET_ME_telegram_chat_id"` and no `credentials` block.

**MAJOR 3's fix raised the stakes on this.** Node 13 now has `retryOnFail: false, maxTries: 1` with `onError: stopWorkflow`. So a single transient Google Drive 503 on a Monday morning now kills the whole run, fires the Error Trigger, runs node 25 — which has `onError: continueRegularOutput`, fails to send, swallows the failure, and the execution finishes looking clean. **Nobody is told the brief did not happen.** That was true before; it is now reachable by a far more ordinary event than it used to be. Removing the retry was still the right call, and the alert is the correct safety net — but the safety net has no credential. **Owner: Leo. Blocking for deployment, not for the Tester's first pass.**

### Criterion 29 — no Drive folder parent

`HTTP Request - Create Google Doc From HTML`, the metadata part of `body`:

```
JSON.stringify({name: $('Code - Build Brief HTML And Chart URLs').item.json.docTitle, mimeType: "application/vnd.google-apps.document"})
```

No `parents` key. Every Doc lands in the credential's My Drive root. **Owner: Leo names the folder; Coder adds `parents: ["<id>"]` — one key in that object.** Non-blocking.

### Criteria 30, 31, 32 — three OPEN PRD items

All three are exactly where pass 1 left them, and all three are Leo's. Nothing regressed. Node 2's notes now carry the five-place `Europe/London` maintenance instruction, which is what MINOR 3 asked for.

---

## Status of every pass-1 finding

| Id | What it was | Now |
|---|---|---|
| BLOCKER 1 | N16 / N21 read approval at the wrong path | **FIXED** — both use `data?.approved ?? approved`, verified live |
| BLOCKER 2 | N12 used `.item` in `runOnceForAllItems` | **FIXED** — both reads are `.first()`, zero `.item` in the body |
| BLOCKER 3 | N3–N7 missing `alwaysOutputData: true` | **FIXED** — present on all five. **See NEW-1: the fix landed and broke something adjacent** |
| BLOCKER 4 | No Telegram credential | **STILL OPEN** — Leo's. The only blocker left |
| MAJOR 1 | HubSpot properties at the wrong path | **WITHDRAWN** — QA was wrong, the Coder was right. Not re-raised. Not re-applied |
| MAJOR 2 | `GTE` paired with `type: "string"` | **FIXED** — both filters now `type: "number"`; schema confirms the pairing |
| MAJOR 3 | N13 retried a non-idempotent create | **FIXED** — `retryOnFail: false, maxTries: 1` |
| MAJOR 4 | N20 retried a non-idempotent append | **FIXED** — `retryOnFail: false, maxTries: 1` |
| MAJOR 5 | Dedupe check and write up to 6 hours apart | **STILL OPEN** — Designer's ruling, correctly untouched |
| MAJOR 6 | PDF binary assumed to survive the approval pause | **STILL OPEN** — Tester proves it, criterion 17 |
| MAJOR 7 | Snapshot row carries no numbers, PRD §9 unreachable | **STILL OPEN** — gated on PRD §12 item 5, Leo |
| MAJOR 8 | Supabase and Support Log contribute nothing | **STILL OPEN** — gated on PRD §12 item 5, Designer / Leo |
| MAJOR 9 | N12 guesses four OpenAI field names | **PARTLY FIXED** — the false "node is disabled" sentence is gone; the four guesses remain, correctly deferred to criterion 34 |
| MINOR 1 | QuickChart long GET URL | **STILL OPEN** — Coder / Designer, handoff question 10 |
| MINOR 2 | `synthetic` hardcoded `false` on the only write | **STILL OPEN** — Tester protocol, criterion 26 |
| MINOR 3 | `Europe/London` hardcoded five times in N2 | **FIXED** — via the documented option: N2's notes name all five sites and require a single co-ordinated edit |
| MINOR 4 | `range` set and ignored on four Sheets nodes | **NOT APPLICABLE** — cosmetic, addressed by `dataLocationOnSheet`; the validator still flags the inert field |
| MINOR 5 | `appendAttribution` inconsistent | **FIXED** — `false` on 15, 17, 18, 21, 22, 25. Paths verified against the live Telegram schema: `options.` for `sendAndWait`, `additionalFields.` for `sendMessage` |
| O1 | Business numbers leave in a public QuickChart URL | **STILL OPEN** — Leo has not been told. Handoff question 11 |
| O2 | Neither Code node has `onError` | **UNCHANGED, and still correct** |
| O3 | Naming | **UNCHANGED** — all 25 nodes re-checked live, every one follows `Original Name - Action` |
| O4 | Connections | **UNCHANGED** — 20 entries, five Merge inputs at indexes 0–4 in the designed order, all three IFs have both outputs wired, Error Trigger island intact, no dangling output |

---

## New findings

### NEW-1 — MAJOR. The `alwaysOutputData` fix made node 9 unable to recognise an empty source.

**Nodes:** 3, 4, 5, 6, 7 (the setting) and 9 (the code that reads them).

The BLOCKER 3 fix is correct and was required by architecture section 10 step 1. But n8n's own documentation for that setting says the node "returns an **empty item** even if the node returns no data". Node 9's `classify()` was written before the setting existed and still tests for zero items:

```js
if (items.length === 0) {
  return { ok: true, rowCount: 0, note: 'Empty week: source returned no rows.', rows: [] };
}
```

**That branch is now unreachable for all five fetches.** A source that legitimately returns nothing now delivers `[{ json: {} }]`. The empty object has no `error` key, so it falls past the failure check into the final branch and is classified `ok: true, rowCount: 1, rows: [{}]` — a phantom row.

**What happens at runtime, traced source by source.**

- **HubSpot, once a credential exists and a week has no matching deals.** `dealRows = [{}]` → `stage = String(undefined || 'unknown')` → `stageTotals = { unknown: 0 }` and `opened = 1`. The brief then draws a pipeline bar chart with a single bar labelled "unknown" at zero, and reports one deal opened. **That is a number the workflow did not fetch, printed in a document going to leadership.** `CLAUDE.md` forbids it in plain words. This is the exact runtime harm withdrawn MAJOR 1 was worried about, arriving by a different road.
- **Snapshot.** One junk trend point with `weekLabel: ''` and `revenue: null`. Node 12 filters it out on `Number.isFinite(p.revenue)`. Harmless today.
- **Finance.** The phantom row has no `week_label`, so it is filtered out, `metrics` stays empty, `readyToBuild` goes false with the right `blockReason`. **Correct outcome, by luck rather than design.**
- **Support.** `sources.support.rowCount` reports 1 for an empty log. No numeric effect on the brief (MAJOR 8), but the classification object is wrong.
- **All five.** Architecture section 10 step 2 requires three outcomes to be told apart — OK, Empty week, Failed. Only two of the three can now occur. "A support log with no tickets is a good week. A support log that could not be read is an unknown." The design says those must never print the same, and today an empty week cannot be identified at all.

**Fix, named not applied.** In `classify()`, immediately before the error check, treat a lone key-less item as the empty-week case:

```js
if (items.length === 1 && (!items[0].json || Object.keys(items[0].json).length === 0)) {
  return { ok: true, rowCount: 0, note: 'Empty week: source returned no rows.', rows: [] };
}
```

**Owner: Coder. Non-blocking for the Tester's first pass — and it should deliberately not be fixed yet.** Criterion 11 requires the Tester to paste the real empty-sheet item shape and the real `continueRegularOutput` failure shape into `docs/knowledge/n8n-gotchas.md`. The guard above is my prediction of that shape. Writing it before the shape is measured would be building a node from a guess, which is the thing this project has a rule against. **Fix it after criterion 11 is recorded, and shape the guard to what the execution actually showed. Blocking before the workflow is switched on.**

**Criterion 21 is amended** to read: *Repeat against a sheet that exists but holds only a header row. That source is classified as an empty week — `ok: true` with no entry in `missingSources` and no warning raised — and the brief treats it as a real zero, not as one row of unknown data. The exact item that source emitted is recorded in `docs/knowledge/n8n-gotchas.md`.* Audit against this wording, not the original.

### NEW-2 — MINOR. A failed snapshot append is now unrecoverable and leaves a permanent gap.

`Google Sheets - Append Weekly Snapshot Row` now has `retryOnFail: false, maxTries: 1` with `onError: stopWorkflow`. That is the right answer to MAJOR 4 and I am not asking for it back. But the consequence has not been written down anywhere: a transient Sheets 503 now ends the run **after** the PDF has already reached leadership, and that week has no snapshot row, permanently. A re-run cannot recover it — the following Monday computes a new `weekLabel`. The only notice is the Error Trigger, which cannot send (BLOCKER 4).

Impact today is low, because the snapshot row carries no numbers anyway (MAJOR 7). It becomes real the moment MAJOR 7 is closed. **Owner: Designer to rule — accept and document, or add a bounded connection-level-only retry. Non-blocking.**

### NEW-3 — MINOR. Two inert leftovers from the retry removals.

Nodes 13 and 20 both carry `waitBetweenTries: 5000` beside `retryOnFail: false`. The value is never used. Harmless, but a reader will believe there is a retry. **Owner: Coder, cosmetic. Non-blocking.**

### NEW-4 — Observation, not a finding. Confirm `?.` and `??` on the first run that reaches node 16.

The BLOCKER 1 fix introduced optional chaining and nullish coalescing into an n8n expression. `n8n_validate_workflow` parsed all 26 expressions with zero errors, and both operators are supported by n8n's expression engine, so I expect this to be fine. I am naming it anyway because of where it sits: if the engine did reject it, node 16 would throw **after a human has already tapped Approve**, and the failure would look like a decline. **The Tester should confirm node 16 evaluated cleanly on the first execution that reaches it, as part of criterion 17.**

### NEW-5 — Observation. Node 11's `onError: continueRegularOutput` can produce a narrative-less brief after a paid failure.

If OpenAI fails both tries, the run continues and node 12 prints "Narrative unavailable this week…". The brief then goes to approval with no executive summary — PRD §4 step 11 and §6's first output line. This is exactly what architecture section 9's retry table specifies, the fallback sentence is honest, and the human gate at node 15 means Leo sees it before anything is sent. **Designed, correct, and now written down.**

---

## Money during testing — the seven nodes that touch something real

Unchanged from pass 1, and the Tester must know all seven before it executes anything.

| Node | What it does for real |
|---|---|
| 11 `OpenAI - Write Executive Narrative` | **The only node that spends money.** `disabled: true`. Keep it disabled for the entire first pass. Enable only on Leo's word, after criterion 22 has passed |
| 13 `HTTP Request - Create Google Doc From HTML` | Creates a real Google Doc in Leo's Drive on **every** run. Free, but one loose file per run in My Drive root, no folder, no cleanup. Delete them after testing |
| 15 `Telegram - Request Brief Approval` | Sends a real message and pauses the execution for up to six hours |
| 17 `Gmail - Send Brief To Leadership` | **Sends a real email.** Safe only while `sendTo` is the placeholder. Point it at Leo's own inbox before the first run that can reach it |
| 18, 21, 22, 25 | Real Telegram posts |
| 20 `Google Sheets - Append Weekly Snapshot Row` | **Writes a real row to the live snapshot sheet with `synthetic` FALSE.** Indistinguishable from a real row, and it blocks the real Monday run for that week. Delete it by hand |
| Node 12's chart URLs | Sends real pipeline and revenue numbers to `quickchart.io` in a public URL (O1) |

---

## What is still blocked on Leo

1. **A Telegram credential in n8n.** Nothing past node 14 can be tested, and no failure alert can arrive, until it exists. The only blocker left.
2. **A Drive folder for the briefs.** Every Doc lands loose in My Drive root. Criterion 29.
3. **The metric list.** `metric-definitions.md` is empty, so Finance Actuals has no metric rows, so a run today stops at node 10. This gates criteria 12 to 19 and criterion 30.
4. **The time zone and the exact week boundary.** PRD §12 item 9. Node 2 holds five copies of `Europe/London` awaiting one co-ordinated edit. Criterion 31.
5. **The leadership email list.** PRD §12 item 7. Criterion 28.
6. **Whether he accepts business numbers leaving in a public QuickChart URL** (O1). He has still not been told.
7. **A ruling on MAJOR 5** — the dedupe check and the write sit up to six hours apart. Designer's to draft, Leo's to accept.

---

## Bottom line

**Yes, hand it to the Tester — but only for criteria 10, 11, 20, 21 and 22, and stop there.** All three Coder blockers are fixed and verified on the live workflow; nothing the Coder owns blocks a run, and node 11 stays disabled, so the first pass costs nothing.

**What stands in the way of anything more:** the missing Telegram credential caps the run at node 14, and the missing metric list stops every run at node 10 before that.

**One thing the Tester must carry into the run:** NEW-1 — the `alwaysOutputData` fix means an empty source now arrives as one blank row, not zero rows. Record the real item shape under criterion 11, then route it back to the Coder. Do not fix it from my prediction.

---

## Main session verification note — 2026-09-13

Before QA v2 was dispatched, the main session pulled the live workflow and checked the Coder's four
headline rework claims by hand rather than taking the Coder's word or QA's:

| Checked | Live value | Verdict |
|---|---|---|
| Node 16 `IF - Was The Brief Approved?` leftValue | `={{ ($('Telegram - Request Brief Approval').item.json.data?.approved ?? $('Telegram - Request Brief Approval').item.json.approved) === true }}` | Landed |
| Node 21 `Telegram - Alert Brief Not Sent` text | same `data?.approved ?? …approved` pattern inside the IIFE | Landed |
| Node 12 `Code - Build Brief HTML And Chart URLs` body | `$('Code - Normalise And Calculate Metrics').first().json` and `$('OpenAI - Write Executive Narrative').first().json`; zero `.item` in the body | Landed |
| Nodes 3, 4, 5, 6, 7 | all five carry `alwaysOutputData: true` beside `onError: continueRegularOutput` | Landed |

`n8n_validate_workflow` at the same moment: `valid: true`, 0 errors, 3 warnings under `runtime`,
25 nodes, 24 enabled, 26 valid connections. Node 11 `disabled: true`. The workflow was not run,
not activated, and not modified.

**NEW-1 was independently re-checked by the main session** against node 9's live `jsCode`. See the
addendum below.

---

## Addendum — main session's independent check of NEW-1. 2026-09-13

**NEW-1 is CONFIRMED.** Node 9's live `jsCode` was pulled separately and read in full. QA quoted it
accurately and its runtime trace is correct.

The three branches of `classify()`, in the order they are evaluated:

1. `items.length === 0` → `{ ok: true, rowCount: 0, note: 'Empty week: source returned no rows.' }`
2. `items.length === 1 && items[0].json && typeof items[0].json.error !== 'undefined'` → `ok: false`
3. otherwise → `rows = items.map(i => i.json || {})`, `{ ok: true, rowCount: rows.length }`

`alwaysOutputData: true` is documented to make a node emit one empty item instead of nothing.
So branch 1 is now unreachable on nodes 3, 4, 5, 6 and 7. An empty source emits `[{ json: {} }]`,
which has no `error` key, so it falls past branch 2 into branch 3 and is classified
`ok: true, rowCount: 1, rows: [{}]`.

The HubSpot trace was re-walked by hand and is right:
`stage = String(d.dealstage || 'unknown')` → `'unknown'`; `amount = toNum(undefined) || 0` → `0`;
`stageTotals = { unknown: 0 }`; `'unknown'` matches neither closed-won nor closed-lost, so
`opened += 1`. An empty deal week would report **one deal opened** and draw a bar labelled
**"unknown"** at zero. Both are numbers the workflow never fetched.

**One thing QA did not say, and it matters for urgency.** The HubSpot harm is latent today, not
live. There is no HubSpot credential on this instance, so that node fails rather than returning
empty, and a failure is caught correctly by branch 2. NEW-1 becomes reachable on HubSpot the day
a credential is added. It is reachable on the three Google Sheets nodes now, because those do have
a working credential.

**The Finance branch is safe by accident, and that accident is load-bearing.** The phantom row has
no `week_label`, so it is filtered out, `metrics` stays empty, and `readyToBuild` correctly goes
false. Nobody should rely on that. It is the reason NEW-1 does not currently produce a bad brief,
and it would stop being true the moment the Finance filter changes.

**Agreed with QA on sequencing: do not fix this yet.** The guard must be shaped to the item that a
real execution produces, recorded under criterion 11, not to a prediction. Fixing it from a guess
is the exact thing `CLAUDE.md` forbids. It is blocking before the workflow is switched on.

---

## Superseded after this report was written — 2026-09-14

**Criterion 27 is now CLOSED. BLOCKER 4 is closed.** This report was written before the Telegram
credential existed, so its FAIL on criterion 27 was correct when written and is now out of date.
It is recorded here rather than edited above, so the original findings stay honest.

What changed, verified live by the main session node by node:

- Leo created the Telegram credential `Executive Rollup Bot Telegram account` (`xkNXmRcwoz0Htiqy`,
  type `telegramApi`), owned by `<owner-google-account>`, the correct account.
- Leo gave his chat id, `SET_ME_telegram_chat_id`, his own Telegram user id.
- The Coder wired both into all five Telegram nodes — 15, 18, 21, 22 and 25. Every one now carries
  the real chat id and the credential. `SET_ME_telegram_chat_id` no longer appears anywhere.
- The stale `notes` on nodes 15 and 18, which still claimed no credential existed, were corrected.
- Validation after both edits was unchanged: `valid: true`, 0 errors, the same 3 warnings, 25 nodes,
  24 enabled, 26 valid connections, 20 connection entries. Node 11 still `disabled: true`.

**Three of the four `SET_ME_` placeholders remain** — `SET_ME_supabase_table_name`,
`SET_ME_supabase_date_column` and `SET_ME_leadership_email_list`. All three are still correct and
still safe.

**Still open, unchanged by this:** criteria 6, 29, 30, 31 and 32, and NEW-1.

## First run attempt — blocked, nothing executed. 2026-09-14

The Tester was dispatched for the limited first pass (criteria 10, 11, 20, 21, 22). **The workflow
did not run. It still has zero executions.** Confirmed with `n8n_executions`: `returned: 0`.

Leo typed `unlock spending`, which is good for one call or one hour. The Tester's first call was
`n8n_test_workflow` with `method: prepare` — a read-only call that lists which nodes would need
pinned data and cannot execute anything. **The guard counted that as the paid call and spent the
unlock on it.** The real run seven seconds later was denied.

From `C:\Users\AMD\.claude\spend-log.txt`, verbatim:

```
2026-09-13T16:17:43.226Z UNLOCKED SPENDING by user phrase
2026-09-13T16:18:12.668Z ALLOWED n8n run: {"workflowId":"iIypy4KWtaqsowAQ","method":"prepare"}
2026-09-13T16:18:19.503Z DENIED n8n run: {"workflowId":"iIypy4KWtaqsowAQ","method":"direct",...}
```

**Nothing real was created or sent.** No Telegram message, no Doc, no PDF, no email, no snapshot
row, no paid call. Node 11 untouched and still disabled. The workflow was not modified.

**The guard behaved correctly by its own rules and failed closed. It was not worked around.** The
observation worth keeping is that `method: prepare` is read-only and burning a one-call unlock on
it costs a whole attempt. Changing that needs Leo's `unlock guard` phrase and is his decision, not
a fault to fix quietly. The guard file cannot even be read from here — a `grep` against it was
denied, which is the guard working as designed.

**On the retry the Tester goes straight to `method: direct` and skips `prepare` entirely**, so the
single unlock lands on the real run.

**NEW-1 remains neither confirmed nor refuted.** Criteria 10, 11, 20, 21 and 22 are NOT REACHED,
not failed. No execution data exists yet to read.
