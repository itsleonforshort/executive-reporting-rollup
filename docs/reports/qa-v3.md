# QA Report — v3

**Date:** 2026-09-14
**Workflow:** `iIypy4KWtaqsowAQ` — "Executive Reporting Rollup - Weekly Leadership Brief"
**Instance:** self-hosted, `https://<your-n8n-instance>`
**Audited against:** the 34 criteria in `docs/reports/qa-v1.md`, criterion 4 amended by its CORRECTION, criterion 21 amended by `qa-v2.md`, criterion 3 amended by `docs/architecture/v1-source-classification-ruling.md`.
**Author:** QA agent. Written to disk by the main session — the agent has no write tool.

**Method:** live `n8n_get_workflow` — `structure`, `filtered` in four batches covering all 25 nodes, `details` for `settings` and stats. `n8n_validate_workflow` profile `runtime`. `n8n_executions` list. `get_node` schema reads against `nodes-base.scheduleTrigger` (full), `nodes-base.gmail` (attachment, emailType), `nodes-base.telegram` (resume), `nodes-base.merge` (chooseBranch), `nodes-base.googleSheets` (operation). `validate_node` on scheduleTrigger and gmail. The `connections` object read by eye. Nothing from memory. Nothing changed. Nothing run. Node 11 confirmed `disabled: true`.

**What changed since pass 2.** Telegram credential wired to all five Telegram nodes and delivery proven live (execution 537). Executions 536 and 537 ran. All five fetch nodes moved to `onError: continueErrorOutput`; five error connections added; node 9's `classify()` rewritten around `$(node).all(1, 0)`; node 12 gained an `emptySources` line; node 11's prompt gained an `emptySources` rule; `mode`/`language` restored on both Code nodes.

**Live state, verified:** `valid: true`, 0 errors, 3 warnings (all three pre-triaged), 25 nodes, 24 enabled, **31 valid connections, 0 invalid**, 26 expressions. `active: false`. `settings` holds 10 keys, **none named `executionTimeout`**; `errorWorkflow: "iIypy4KWtaqsowAQ"`; `timezone: Europe/London`; `pinData: {}`. `versionCounter: 21`, `updatedAt: 2026-09-13T17:08:21Z`.

**The single most important fact in this report: executions 536 and 537 both started at 16:23 and 16:33; the rework landed at 17:08. Every piece of execution evidence in `tester-v1.md` was measured on a graph that no longer exists. The current build has zero executions against it.**

---

## Verdict

**PASS for one more free Tester run. FAIL for deployment.**

**Open: 0 blockers that stop a test run. 3 blockers that stop activation. 5 new majors, 5 new minors, plus 6 majors and 4 minors carried from earlier passes.**

Nothing found prevents the next run — and that run is the only thing that can resolve the two largest risks, both of which are structural consequences of the rework rather than defects in it. The run costs nothing: node 11 stays disabled, Finance Actuals is still empty so the run stops at node 10, and no write is reachable.

The rework is good work. Node 9's classifier is materially safer than what it replaced, the phantom row is gone unconditionally, and the Coder's deviation from the ruling is correct. But this pass found three faults that passes 1 and 2 both missed, one of which has been sitting in the build since day one.

---

## The 34 criteria

| # | Criterion (short) | Result | Evidence |
|---|---|---|---|
| 1 | N16 + N21 read approval at the emitted path, strict `=== true` | **PASS** | Both live: `.item.json.data?.approved ?? .item.json.approved`, N16 wrapped `=== true`, `typeValidation: strict` |
| 2 | N12 zero `.item`; both cross-node reads `.first()` | **PASS** | Whole `jsCode` read; `$('Code - Normalise…').first()` and `$('OpenAI…').first()`; no `.item` present |
| 3 | *(amended by ruling)* N3–N7 `alwaysOutputData: true` + `onError: continueErrorOutput` | **PASS** | All five carry both, plus `retryOnFail: true / maxTries: 3 / waitBetweenTries: 5000` |
| 4 | *(amended)* N3 properties at `additionalFields.properties`; filter type/operator a schema pair | **PASS** | Six names at `additionalFields.properties`; both filters `type: "number"` with `GTE` / `LT` |
| 5 | N11 still `disabled: true` | **PASS** | `"disabled": true` on n11; credential attached, node inert; never reached on 536 or 537 |
| 6 | N17 `sendTo` is Leo's own address | **FAIL** | Live value `SET_ME_leadership_email_list`. Correct and safe today — see detail |
| 7 | N20 targets snapshot doc/gid, `defineBelow`, six columns in order | **PASS** | doc `1TdTEZ…VElA`, gid `1425370562`, keys `week_label, week_start, week_end, run_at, synthetic, notes` |
| 8 | No `executionTimeout`; `errorWorkflow` is itself | **PASS** | `settings` read live: 10 keys, no `executionTimeout`; `errorWorkflow: "iIypy4KWtaqsowAQ"` |
| 9 | No Split In Batches / Split Out / Item Lists / Loop Over Items / sub-workflow | **PASS** | All 25 `type` values read; none is a fan-out or sub-workflow node |
| 10 | Merge receives 5 inputs, N9 outputs exactly 1 item | **BLOCKED** | Was PASS on 536/537; the graph changed under it — 10 connections now feed the same 5 inputs. Re-prove |
| 11 | N9 classifies all five correctly; item shapes recorded | **BLOCKED** | Was FAIL (Supabase `ok: true`). Rewrite is live and correct on paper; zero executions against it |
| 12 | With 2+2 metric rows, `readyToBuild` true, run reaches N12 | **BLOCKED** | Finance Actuals has no metric rows; PRD §12 item 5 OPEN |
| 13 | N12 produces `html`, `telegramSummary`, `weekLabel`, does not throw | **BLOCKED** | Gated on 12. All three keys present in N12's single `return` |
| 14 | N13 returns a Doc id and both charts render inside it | **BLOCKED** | Gated on 12. Still the design's riskiest unproven assumption |
| 15 | N14 returns binary `data` holding a PDF that opens | **BLOCKED** | `options.binaryPropertyName: "data"`, `docsToFormat: application/pdf` set |
| 16 | N15 sends exactly one approval, execution enters waiting | **BLOCKED** | Credential and chat now work (537). Gated on 12. `retryOnFail: false` so only one send |
| 17 | On Approve, N16 true, N17 attaches binary `data`, email arrives | **BLOCKED** | Gated on 12 and 6 — **and now at risk from NEW-2: the attachment field name is not set** |
| 18 | Sheet gains exactly one row with the six correct values | **BLOCKED** | Gated on 17 |
| 19 | Re-run same week gives `readyToBuild: false` naming the existing row | **BLOCKED** | `alreadyAppended` → `blockReason` logic present and now gated on `snapshot.state` |
| 20 | Broken sheet id: run continues, warning everywhere, no row | **BLOCKED** | Was PARTIAL PASS on 536. Detection mechanism entirely replaced; re-prove |
| 21 | *(amended)* Header-only sheet classified as an empty week, not one unknown row | **BLOCKED** | Was FAIL. Code now returns `state: 'empty', ok: true, rowCount: 0`; unproven |
| 22 | Break Finance: `readyToBuild` false, N11 never reached, N22 fires | **BLOCKED** | Was full PASS on 537 including delivery. Node 9 rewritten beneath it; re-prove |
| 23 | Decline: nothing sent, N21 says **declined** | **BLOCKED** | Wording logic live: `approved === false` → "The brief was declined." `approvalType: double` set |
| 24 | Expire: nothing sent, N21 says **expired**; expiry item recorded | **BLOCKED** | Falls to "Nobody responded within the 6-hour approval window." |
| 25 | Force N13 to fail: Error Trigger fires, N25 message **arrives** | **BLOCKED** | Now reachable — bot conversation open. Not yet exercised |
| 26 | No test row left with `synthetic` FALSE | **BLOCKED** | No row has ever been written. N20 still hardcodes `synthetic: false` |
| 27 | Telegram credential on N15/18/21/22/25; real `chatId` | **PASS** | All five carry `xkNXmRcwoz0Htiqy` + `SET_ME_telegram_chat_id`; delivery proven on execution 537 |
| 28 | N17 `sendTo` is the real leadership list | **BLOCKED** | Gated on 17; PRD §12 item 7 OPEN |
| 29 | N13 body names a real Drive folder parent, id recorded | **FAIL** | Metadata object is `{name, mimeType}` only — no `parents` key |
| 30 | `metric-definitions.md` complete, Finance Actuals populated | **FAIL** | PRD §12 item 5 OPEN; the file is empty on purpose |
| 31 | PRD §12 item 9 closed; node 2 matches | **FAIL** | Item 9 still OPEN. N2's notes carry the five-site maintenance instruction |
| 32 | Real HubSpot business filter replaces `hs_lastmodifieddate` | **FAIL** | Still the placeholder; PRD §12 item 4 OPEN and PARKED |
| 33 | N11 enabled, one call, cost logged, exactly 1 in / 1 out | **BLOCKED** | Node disabled by rule; zero money spent on this project |
| 34 | N12 reads the narrative from the field proven in 33 | **BLOCKED** | Still four guessed names — correctly deferred |

**Totals: 9 PASS, 5 FAIL, 20 BLOCKED, 0 N/A.** Group A is 8 of 9. Group D gained its first PASS (27).

---

## Detail on the FAIL and re-opened rows

**Criterion 6.** `SET_ME_leadership_email_list` is still live. This remains the right state. Gmail rejects a string with no `@`, so the node cannot reach anyone by accident while the workflow is being broken on purpose. Change it to Leo's own address in the same sitting as the first run that can reach node 17, and not before. **Owner: Coder, on the Tester's instruction. Non-blocking.**

**Criteria 10, 20, 22 — re-opened, and this is the whole reason this section exists.** All three were satisfied or partly satisfied on executions 536 and 537. The rework then changed the five fetch nodes' error routing, added five connections into the Merge, and rewrote node 9's classifier. Evidence measured on the old graph cannot be carried forward to the new one. I am not saying they will fail. I am saying nobody knows, and King must not be shown a PASS that was earned by a build that no longer exists.

**Criteria 11 and 21 — the fix is live and looks right; that is not the same as working.** Traced by hand against the live `jsCode`, the classifier is correct for every path *provided* `$(node).all(1, 0)` behaves as documented. See NEW-4.

**Criterion 29.** Node 13's metadata part is `JSON.stringify({name, mimeType})`. No `parents`. Every Doc lands in My Drive root. **Owner: Leo names the folder, Coder adds one key.**

**Criteria 30, 31, 32.** Three OPEN PRD items, all Leo's, all exactly where pass 2 left them.

---

## Status of every prior finding

| Id | What it was | Now |
|---|---|---|
| BLOCKER 1 | N16/N21 wrong approval path | **FIXED** — verified live again this pass |
| BLOCKER 2 | N12 used `.item` in `runOnceForAllItems` | **FIXED** — zero `.item` in the body |
| BLOCKER 3 | N3–N7 missing `alwaysOutputData` | **FIXED, then superseded** — setting stays, but the ruling stripped it of meaning |
| BLOCKER 4 | No Telegram credential | **CLOSED** — credential + chat id on all five; delivery proven on 537 |
| MAJOR 1 | HubSpot properties path | **WITHDRAWN** — not re-raised, not re-applied |
| MAJOR 2 | `GTE` with `type: "string"` | **FIXED** — both filters `type: "number"` |
| MAJOR 3 | N13 retried a non-idempotent create | **FIXED** — `retryOnFail: false, maxTries: 1` |
| MAJOR 4 | N20 retried a non-idempotent append | **FIXED** — `retryOnFail: false, maxTries: 1` |
| MAJOR 5 | Dedupe check and write up to 6h apart | **STILL OPEN** — Designer. Worsened by NEW-4, see below |
| MAJOR 6 | PDF binary assumed to survive the pause | **STILL OPEN** — Tester, criterion 17. Now compounded by NEW-2 |
| MAJOR 7 | Snapshot row carries no numbers | **STILL OPEN** — gated on PRD §12 item 5 |
| MAJOR 8 | Supabase and Support Log contribute nothing | **STILL OPEN** — but `emptySources` now at least names them on the brief |
| MAJOR 9 | N12 guesses four OpenAI field names | **STILL OPEN** — correctly deferred to criterion 34 |
| MINOR 1 | QuickChart long GET URL | **STILL OPEN** — handoff question 10 |
| MINOR 2 | `synthetic` hardcoded `false` | **STILL OPEN** — Tester protocol, criterion 26 |
| MINOR 3 | `Europe/London` five times in N2 | **FIXED** — via documented maintenance note |
| MINOR 4 | `range` set and ignored on Sheets nodes | **FIXED** — `range` is gone from nodes 5, 6, 7 and 20 |
| MINOR 5 | `appendAttribution` inconsistent | **FIXED** — `false` on 15, 17, 18, 21, 22, 25 |
| O1 | Numbers leave in a public QuickChart URL | **STILL OPEN** — Leo has still not been told |
| O2–O6 | Code node `onError`, naming, connections, settings, never-run | **UPDATED** — naming re-checked, all 25 conform. Connections re-read, 31 valid. Two executions now exist |
| **NEW-1** (v2) | `alwaysOutputData` broke empty-source detection | **FIXED** — confirmed on 536, then fixed at source by the ruling. `rowCount` counts keyed items only |
| **NEW-2** (v2) | Failed append now unrecoverable | **STILL OPEN** — Designer to accept and document |
| **NEW-3** (v2) | Inert `waitBetweenTries` on 13 and 20 | **STILL OPEN** — cosmetic |
| **NEW-4** (v2) | Confirm `?.` and `??` on the first run reaching N16 | **STILL OPEN** — carry into criterion 17 |
| **NEW-5** (v2) | N11 `continueRegularOutput` → narrative-less brief | **UNCHANGED, designed, correct** |
| **T-F1** (tester) | Dead Supabase reported healthy | **FIXED IN CODE, UNPROVEN** — criterion 11 |
| **T-F2** (tester) | Alert failed, run reported success | **CLOSED** — Leo opened the bot chat; proven on 537 |
| **T-F3** (tester) | NEW-1 confirmed on real data | **FIXED** — see NEW-1 above |

---

## New findings

### NEW-1 — MAJOR, BLOCKING BEFORE ACTIVATION. The Schedule Trigger has no schedule.

**Node 1, `Schedule Trigger - Weekly Monday 7am`. `"parameters": {}` — empty. It has been empty since the workflow was created, and the local copy confirms it was never set.** Both prior QA passes missed it.

Read from the live `nodes-base.scheduleTrigger` v1.4 schema this pass: `rule` defaults to `{ "interval": [ { "field": "days" } ] }`, `daysInterval` defaults to `1`, `triggerAtHour` defaults to `0` (Midnight), `triggerAtMinute` defaults to `0`.

**What happens the moment this workflow is switched on: it runs every day at 00:00, not weekly on Monday at 07:00.** PRD §4 step 1 says Monday 07:00. The node's own name says Monday 7am. Neither is true.

Traced consequence of a full week live: Monday 00:00 produces the brief seven hours early; Tuesday through Sunday each run to node 10, hit `alreadyAppended`, and fire node 22 — **six "Run blocked" Telegram messages to Leo every week, forever.** Each of those days also creates nothing and costs nothing, so the fault is loud rather than dangerous. It is still wrong.

`misfirePolicy` is also unset. Its default is `skip`, which is the safe value and the one QA v1's MAJOR 5 hypothetical assumed away from — but it is a default, not a decision.

**Fix, named not applied.** `rule.interval[0]` must be `field: "weeks"`, `weeksInterval: 1`, `triggerAtDay: [1]`, `triggerAtHour: 7`, `triggerAtMinute: 0`. Note `triggerAtDay` defaults to `[0]` — Sunday — so it must be set explicitly. Set `misfirePolicy` explicitly at the same time.

**Owner: Coder. Blocking before the workflow is activated. Not blocking the next Tester run, which is manual.**

### NEW-2 — MAJOR. Node 17 names no attachment field, so the PDF may never attach.

**Node 17, `Gmail - Send Brief To Leadership`:**

```json
"attachmentsUi": { "attachmentsBinary": [ {} ] }
```

The array holds an **empty object**. There is no `property` key. `qa-v1.md` states this was "verified correct against the Gmail v2.2 schema" at `options.attachmentsUi.attachmentsBinary[0].property = "data"`. **That value is not there and, per the local copy, never was.** Pass 1 verified that the path exists. It did not check that anything was written to it. Pass 2 repeated the claim.

A `search_properties` read against `nodes-base.gmail` this pass returns **two** entries for `options.attachmentsUi.attachmentsBinary.property`: one labelled "Attachment Field Name (in Input)" with `default: ""`, one labelled "Attachment Field Name" with `default: "data"`. The flattened output hides the operation scoping — the exact trap that produced the withdrawn MAJOR 1. **I am not going to guess which applies to `message:send` on v2.2, and neither should anyone else.**

**What happens at runtime, both ways.** If the applicable default is `"data"`, the PDF attaches by luck. If it is `""`, node 17 either sends leadership an email with no PDF on it, or throws "binary property not found" — and node 17 has no `onError`, so the default stops the run **after a human has already approved**. PRD §4 step 16 fails either way, silently in the first case.

**Fix.** Set `property: "data"` explicitly. That is correct under either default, so the ambiguity stops mattering the moment it is written down.

**Owner: Coder. Blocking criterion 17. Not blocking the next run — node 17 is unreachable while Finance Actuals is empty.**

### NEW-3 — MAJOR. Node 17 retries a real email send three times.

`retryOnFail: true, maxTries: 3, waitBetweenTries: 5000` on a Gmail `send`. **Sending an email is not idempotent.** A Gmail call that timed out after the message was accepted, retried twice, puts the same brief in every leadership inbox three times. It also retries a 4xx — a malformed recipient list can never succeed, so two of the three calls are guaranteed waste.

This is the same fault as MAJOR 3 and MAJOR 4, on the one node that reaches people outside the company, and all three earlier passes walked past it. MAJOR 3 and 4 were fixed by setting `retryOnFail: false, maxTries: 1`. The same fix applies here, for the same reason.

**Owner: Coder. Non-blocking now, blocking before criterion 17.**

### NEW-4 — MAJOR. The fail-closed guard covers a throw. It does not cover a silent empty array, and that gap lands on the only write.

This is my ruling on task item 4, and it is the most important thing in this report.

`$('<node>').all(1, 0)` has never been measured on this instance. Three things can happen. I traced all three against the live `jsCode`.

**(a) It throws for a healthy node.** Every source returns `state: 'failed'` from the guard. `missingSources.length === 5` → `blockReason: "All five sources failed. Nothing to report."` → run stops at node 10 → node 22 alerts. Each source's `note` says *"Could not read the error output of … Treated as failed -- fail-closed"* and `sourceDiagnostics.errorBranchThrew` is `true`. **Loud, honest, diagnosable, no brief, no row, no paid call. The guard covers this completely.**

**(b) It throws for one node only.** That source is falsely `failed` → warning banner on the brief, `dataComplete: false`, node 19 routes false, no snapshot row. **A false alarm. Recoverable. The guard covers this.**

**(c) It returns `[]` for a node that genuinely failed.** The guard never fires. Execution falls to rule 2, `$(node).all(0, 0)`, which for a node that routed everything to output 1 returns zero items or one `alwaysOutputData` blank. Rule 3 finds no `error` key — because under `continueErrorOutput` the error item is on branch 1, where nothing is looking. Rule 4 drops the blank. Rule 5 returns **`state: 'empty'`, `ok: true`**.

**A dead source is reported as a healthy empty week.** That is the Supabase bug from execution 536, wearing a new coat, and the fail-closed guard does not touch it.

**It is worse than the original, in one specific place.** With `state: 'empty'` the source is absent from `missingSources`, so `dataComplete` stays `true`, so **node 19 routes true and node 20 appends a snapshot row**. If the source that silently failed is the snapshot sheet itself, `alreadyAppended` was computed from an empty `rows` array and is `false` — so the run appends a row for a week that may already have one. **Case (c) puts a duplicate row on the workflow's only write, and it does it quietly.** That interacts with MAJOR 5, which is still open.

**Is the fail-closed guard sufficient cover? No — it is sufficient for (a) and (b), and it is no cover at all for (c).** The Coder's report and the ruling both read as though the throw case was the danger. It was one of two.

**What resolves it: one free run, already possible.** `sourceDiagnostics.supabase.errorBranchCount` is the answer. Supabase has no credential, so it fails on every run. If `errorBranchCount >= 1` and `supabase.state === 'failed'`, case (c) is dead and the ruling is proven. If `errorBranchCount === 0` while `errorBranchThrew` is `false`, case (c) is real and the Designer's Plan B (ruling §4.6, five Set nodes reading `.isExecuted`) must be built.

**Owner: Tester to measure on the next run. Designer to rule if it measures badly. Non-blocking for the run — the run is the fix. Blocking before activation.**

### NEW-5 — MAJOR. Two outputs of one node into one Merge input is unmeasured, and a stall here is silent.

Read from the live `connections` object, node by node. Every one of the five fetch nodes now has `main[0]` and `main[1]`, each holding exactly one target, both pointing at the same Merge input: HubSpot → 0, Supabase → 1, Finance → 2, Support → 3, Snapshot → 4. **The Merge keeps five inputs at indexes 0 to 4 in the designed order. No existing connection was removed or re-pointed. This is built exactly as the ruling specifies.** `validConnections` went 26 → 31, exactly +5.

The Merge is `mode: chooseBranch`, `numberInputs: 5`, `output: "empty"`. `chooseBranchMode` is absent; I read the live schema and it defaults to `waitForAll`, which is also its only option, so that one cannot go wrong.

**The unmeasured part.** On every run, exactly one of the two connections feeding each Merge input stays silent — the error branch on a good run, the main branch on a failure. If `waitForAll` waits on *connections* rather than on *inputs*, the Merge never fires.

**What that looks like on an unattended Monday at 07:00.** The execution sits at node 8. Nothing downstream runs — including node 22 and node 25. There is no error, so `settings.errorWorkflow` never fires. **No brief, no alert, nobody told, and the execution does not even go red.** That is the worst failure shape this workflow has, and it would arrive on the first live Monday.

This is not a defect — the Coder built the approved ruling, and the ruling explains at §4.2 exactly why both outputs feed one input. It is a risk the design consciously took, and the Designer listed it at §5 item 2. I am raising it to the top of the Tester's list because its failure mode is silent and its cost is a whole reporting week, and because one free run settles it.

**Owner: Tester to prove on the next run, before anything else. Blocking before activation.**

### NEW-6 — MINOR. Node 8's notes now describe a workflow that does not exist.

`Merge - Wait For All Five Sources` still carries: *"UNVERIFIED … whether this node still emits exactly one empty item when one of the five inputs carries an error-shaped item from an upstream **continueRegularOutput** node."*

Two things wrong. No fetch node uses `continueRegularOutput` any more. And the behaviour was verified — twice, on 536 and 537. The note also calls it "the single most safety-critical unverified claim in the design", which is no longer true; NEW-5 is. This is the same class of stale-note fault the Orchestrator caught on nodes 15 and 18. **Owner: Coder, cosmetic but misleading.**

### NEW-7 — MINOR. Nine behaviour-bearing parameters are left at n8n defaults.

`CLAUDE.md` build rule 1: "Set every parameter yourself. Never rely on an n8n default." My own remit: "`onError` settings are deliberate, not left at default." Read live:

- **`onError` absent on nodes 13, 14, 17 and 20.** All four rely on the `stopWorkflow` default. `qa-v1.md` states these were "set on purpose" — they are not set at all. The behaviour is right; the record is not.
- **`resource` and `operation` absent on nodes 5, 6, 7** (Sheets read), **14** (Drive) and **17** (Gmail). Confirmed from the live Sheets schema that `operation` defaults to `read` under `resource: sheet`, and from `validate_node` that node 17 resolves to `message` / `send`.
- **`emailType` absent on node 17.** The schema marks it **`required: true`** and defaults it to `html` for `message:send`. The body is HTML, so the default is the value wanted — but a required field is unset.
- **`limitType` and `resumeUnit` absent on node 15.** `resumeAmount: 6` alone. Read from the live Telegram schema: `limitType` defaults to `afterTimeInterval` and `resumeUnit` to `hours`, so the window really is six hours. Nothing in the build says so.
- **`chooseBranchMode` absent on node 8.** Locked to its only value; harmless.
- **`misfirePolicy` absent on node 1.** Defaults to `skip`. See NEW-1.

None of these is wrong today. All of them are a value nobody chose, and three of them (`onError`, `emailType`, `resumeUnit`) carry real behaviour. This is the same fault the Orchestrator already forced a fix for on the two Code nodes' `mode` and `language`. **Owner: Coder. Non-blocking.**

### NEW-8 — MINOR. `sourceDiagnostics` is now sent to the paid model, undescribed.

Node 11's user message is `JSON.stringify($('Code - Normalise And Calculate Metrics').item.json)` — the whole object, which now includes `sourceDiagnostics` with `errorBranchThrew`, `errorBranchMessage` and the item counts. The system prompt describes `missingSources` and `emptySources` and says nothing about diagnostics. Token cost is trivial. The honest risk is a leadership narrative that mentions an error branch. The ruling says nothing downstream reads `sourceDiagnostics`; node 11 now does. **Fix: strip it from the prompt payload, or name it in the system prompt and forbid mentioning it. Owner: Coder, before node 11 is ever enabled.**

### NEW-9 — MINOR. `emptySources` reaches the Doc only.

Node 12 adds `emptySourcesHtml` to `html` and nowhere else. `telegramSummary` carries `dataWarning` only, and node 17's email body carries `dataWarning` only. Leo approved naming empty sources on the brief, and the Doc is the brief, so this meets what was approved. Recording it so nobody later reads the Telegram summary as the full picture. **Owner: Designer, if consistency is wanted. Non-blocking.**

### NEW-10 — MINOR. NEW-3 from pass 2 is still open.

Nodes 13 and 20 still carry `waitBetweenTries: 5000` beside `retryOnFail: false`. Inert. A reader will believe there is a retry.

---

## Rulings on the four questions I was asked

**1. Node 9's rewritten `classify()` — correct, with one hole that is not the Coder's.** Rule order read from the live source is exactly the ruling's: error branch, main branch, `error`-key belt-and-braces, drop key-less items, empty, ok. The five node names in `classify()` calls match the `connections` object character for character; `$('Set - Define Reporting Week').first()` matches too. `rowCount` is `rows.length` **after** key-less items are dropped, so it can only exceed zero when `state === 'ok'` with a genuinely keyed row. All three call sites the ruling named now gate on `state`, and the Coder correctly extended that to `finance` rather than leave one `.ok` behind. `dealRows` gates on `hubspot.state === 'ok'`, `revenueTrend` on `snapshot.state === 'ok'`, `alreadyAppended` on `'ok' || 'empty'` — and for `'empty'`, `rows` is `[]`, so `.some()` returns false safely. **Can a dead source still be reported as healthy? Only through case (c) of NEW-4, and no amount of rewriting node 9 can close that — it needs a measurement or Plan B.**

**One item, still.** The only top-level `return` is `return [{ json: {…} }];`. Every other `return` is inside `classify()` and hands back a plain object, not items. `mode: "runOnceForAllItems"` and `language: "javaScript"` are both explicit. No fan-out node exists anywhere. **The one-paid-call guarantee is intact and was not put at risk by this rework.**

**2. The five error connections — correct, exactly as ruled.** Verified by reading `connections` myself. See NEW-5 for the one thing about them nobody has measured.

**3. The Coder's deviation — UPHELD. I agree with it, and I would have required it.** The ruling's literal wording sends a throw on `.all(1, 0)` through rule 2 (no throw on main), rule 3 (nothing to inspect), into rule 5 — `state: 'empty'`, `ok: true`. A genuinely dead source reported as a healthy empty week, in the one code path nobody has run. That is the precise fault the whole ruling exists to remove, relocated rather than fixed. The deviation's cost is a false FAILED, which produces a blocked run, an honest alert, no brief, no snapshot row and no paid call. **A false alarm is recoverable in ten minutes. A false brief in front of leadership is not.** The Coder also flagged it rather than absorbing it into the ruling, which is the right way to disagree. Keep it. The Designer should fold it into the ruling text so the next reader does not have to reconstruct the argument. **And nobody should read "fail-closed" as meaning the error-branch read is now safe — see NEW-4.**

**4. Node 12's `emptySources` line — PASS, it reads as a plain line.** Live: `'<p>' + … + '</p>'`, no `style` attribute, placed directly after `dataWarningHtml`, which by contrast carries `background:#fff3cd; border:1px solid #ffe69c; font-weight:bold`. The two are visually unmistakable. Text renders as *"Support log: no data this week."* — generic across all five sources rather than the ruling's ticket-specific example, which is better. **This is what Leo approved.**

**5. The phantom row — genuinely gone, on every path, unconditionally.** Even under NEW-4 case (c), rule 4 drops key-less items before any counting, so `rowCount` is `0` and `rows` is `[]`. `dealRows` cannot produce a bar labelled "unknown" at zero or one phantom deal opened. `revenueTrend` cannot produce `{ weekLabel: '', revenue: null }`. Node 12's `Number.isFinite(p.revenue)` filter is still in place as defence in depth. **This is the cleanest win of the rework and it does not depend on any unmeasured behaviour.**

---

## Money and real-world effects during testing — unchanged, eight items

| Node | What it does for real |
|---|---|
| 11 `OpenAI - Write Executive Narrative` | **The only node that spends money.** `disabled: true`. Keep it disabled for the whole next pass |
| 13 `HTTP Request - Create Google Doc From HTML` | Creates a real Google Doc in Leo's Drive on every run that reaches it. Free, loose in My Drive root, no cleanup |
| 15 `Telegram - Request Brief Approval` | Sends a real message and pauses the execution for six hours |
| 17 `Gmail - Send Brief To Leadership` | **Sends a real email, and retries it up to three times (NEW-3).** Safe only while `sendTo` is the placeholder |
| 18, 21, 22, 25 | Real Telegram posts. Delivery now works — proven on 537 |
| 20 `Google Sheets - Append Weekly Snapshot Row` | **Writes a real row with `synthetic` FALSE.** Indistinguishable from a real row; blocks the real Monday run for that week |
| Node 12's chart URLs | Sends real pipeline and revenue numbers to `quickchart.io` in a public URL (O1) |
| **Node 1, if activated** | **NEW-1: would run the whole workflow every day at midnight, not weekly** |

None of the first seven is reachable on the next run while Finance Actuals holds no metric rows. The run stops at node 10.

---

## Still blocked on Leo

1. **The metric list.** `metric-definitions.md` is empty, so every run stops at node 10. This gates criteria 12 to 19, 23, 24, 26, 28 and 30 — twelve of the thirty-four.
2. **A Drive folder for the briefs.** Criterion 29.
3. **The leadership email list.** Criterion 28.
4. **The time zone and the exact week boundary.** PRD §12 item 9. Criterion 31.
5. **Whether he accepts business numbers leaving in a public QuickChart URL** (O1). He has still not been told.
6. **A ruling on MAJOR 5**, the dedupe check and the write sitting six hours apart — Designer to draft, Leo to accept. NEW-4 case (c) makes this sharper.
7. **Confirming the workflow may stay "Available in MCP".** `settings.availableInMCP` is `true` on the live workflow. It had to be, for executions 536 and 537. The handoff says to ask him before enabling it on any workflow.

---

## New acceptance criteria — 35 to 41

The 34 stand. These are added because this pass found faults no existing criterion would have caught. All must be true before the workflow is switched on.

35. Node 1's `rule.interval[0]` is `field: "weeks"`, `weeksInterval: 1`, `triggerAtDay: [1]`, `triggerAtHour: 7`, `triggerAtMinute: 0`, and `misfirePolicy` is set explicitly — read back off the live workflow, not the local copy.
36. On one execution of the current graph, node 8 fires and node 9 outputs exactly one item, with both outputs of all five fetch nodes wired into their five inputs. The Merge's own output item count is read off the execution and recorded in `n8n-gotchas.md`. **If node 8 does not fire, the execution is stopped and NEW-5 is escalated to the Designer before anything else is attempted.**
37. On that same execution, `sourceDiagnostics.supabase.errorBranchCount` is at least 1, `errorBranchThrew` is `false`, `sources.supabase.state` is `"failed"`, and `"Supabase product usage"` appears in `missingSources`. The exact error-branch item shape is pasted into `n8n-gotchas.md`.
38. On that same execution, each of the three Google Sheets sources shows `state: "empty"`, `rowCount: 0`, `blankItemCount: 1`, absent from `missingSources`, present in `emptySources`, and `dataWarning` names only the sources that actually failed.
39. Node 17 carries `options.attachmentsUi.attachmentsBinary[0].property = "data"`, `emailType: "html"`, `resource: "message"`, `operation: "send"`, and `retryOnFail: false` with `maxTries: 1`.
40. Every node whose behaviour depends on it carries an explicit `onError` — 13, 14, 17 and 20 included — and no node in the workflow relies on an unstated default for `resource`, `operation`, `emailType`, `resumeUnit` or `limitType`.
41. Node 11's prompt payload either excludes `sourceDiagnostics` or the system prompt names it and forbids referring to it, and node 8's `notes` describe the graph as it is now.

---

## Bottom line

**Yes, hand it to the Tester — one free run, node 11 disabled, and criteria 36, 37 and 38 are the entire scope. Stop at node 10 as the workflow already does.** That run settles both of the two risks that matter, and it costs nothing.

**What stands in the way of more:** NEW-5 could stall the Merge silently, NEW-4 case (c) could still report a dead source as a healthy week and then duplicate the only write, and the missing metric list stops every run at node 10 regardless.

**Do not activate this workflow under any circumstances.** Node 1 has no schedule — switched on today it runs every day at midnight, not Monday at seven.

---

## Main session verification — 2026-09-14

Before this report was saved, the main session checked QA's two headline new findings against the live workflow rather than taking them on trust. **Both are confirmed.**

**NEW-1 confirmed.** `n8n_get_workflow` mode `filtered` on `Schedule Trigger - Weekly Monday 7am` returns:

```json
{ "parameters": {}, "id": "n1", "name": "Schedule Trigger - Weekly Monday 7am",
  "type": "n8n-nodes-base.scheduleTrigger", "typeVersion": 1.4, "position": [0, 0] }
```

`parameters` is empty. There is no `rule`, no `triggerAtHour`, no `triggerAtDay`. The node name is the only place in the build where "Monday 7am" appears. Two QA passes and a Coder build went past it, and it would have surfaced as a wrong-time run on the first live Monday.

**NEW-2 confirmed.** Node 17's `options.attachmentsUi.attachmentsBinary` is `[ {} ]` — a single empty object with no `property` key.

**NEW-3 confirmed.** Node 17 carries `retryOnFail: true, maxTries: 3, waitBetweenTries: 5000` on a Gmail send, and has no `onError` key at all.

QA v3 is the strongest pass so far. It found three real faults that two earlier passes and the Coder all walked past, and it was right to re-open criteria 10, 20 and 22 on the grounds that the graph changed underneath the evidence.
