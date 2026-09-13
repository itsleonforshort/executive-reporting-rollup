# Tester Report — v1. First execution ever.

**Date:** 2026-09-13 (execution clock), reported 2026-09-14
**Workflow:** `iIypy4KWtaqsowAQ` — "Executive Reporting Rollup - Weekly Leadership Brief"
**Instance:** self-hosted, `https://<your-n8n-instance>`
**Execution id:** `536` — the first execution this workflow has ever had
**Run by:** the main session, after two blocked attempts by the `tester` agent
**Scope:** the limited first pass QA v2 authorised — criteria 10, 11, 20, 21, 22

**Result:** `status: success`, `finished: true`, 11 of 25 nodes executed, 19.147 seconds.
Started 16:23:59.360Z, stopped 16:24:18.507Z.

**Cost: zero.** Node 11 `OpenAI - Write Executive Narrative` was never reached and stayed
`disabled: true` throughout.

---

## 1. How the run was finally made

Two earlier attempts failed before anything executed. Both are worth recording.

**Attempt 1 — the unlock was spent on a read-only call.** The `tester` agent called
`n8n_test_workflow` with `method: prepare`, which lists nodes needing pinned data and cannot
execute anything. The spending guard counted it as the paid call. The real run 7 seconds later was
denied. From `spend-log.txt`:

```
2026-09-13T16:17:43.226Z UNLOCKED SPENDING by user phrase
2026-09-13T16:18:12.668Z ALLOWED n8n run: {"workflowId":"iIypy4KWtaqsowAQ","method":"prepare"}
2026-09-13T16:18:19.503Z DENIED n8n run: {"workflowId":"iIypy4KWtaqsowAQ","method":"direct",...}
```

**Attempt 2 — the guard passed it, a session permission layer did not.** The agent went straight
to `method: direct`. The spending guard ALLOWED it, then the call was stopped inside the session
before it reached n8n, and the agent lost access to its other tools.

```
2026-09-13T16:19:21.541Z UNLOCKED SPENDING by user phrase
2026-09-13T16:19:28.587Z ALLOWED n8n run: {"workflowId":"iIypy4KWtaqsowAQ","method":"direct",...}
```

**Attempt 3 — run from the main session, straight to `method: direct`.** Succeeded.

**Lesson worth keeping: never call `method: prepare` when the unlock is a one-shot.** Go straight
to `method: direct`. Two of Leo's three unlocks were spent producing nothing.

---

## 2. What executed

| # | Node | Status | Out | Note |
|---|---|---|---|---|
| 1 | Schedule Trigger - Weekly Monday 7am | success | 1 | manual start |
| 2 | Set - Define Reporting Week | success | 1 | `2026-W36`, Europe/London |
| 3 | HubSpot - Fetch Deals And Pipeline | success | 1 | failed, named itself, 10,022 ms |
| 4 | Supabase - Fetch Product Usage | success | 1 | **failed silently, 11 ms** |
| 5 | Google Sheets - Fetch Finance Actuals | success | 1 | blank item, 1,406 ms |
| 6 | Google Sheets - Fetch Support Log | success | 1 | blank item |
| 7 | Google Sheets - Fetch Last Week Snapshot | success | 1 | blank item |
| 8 | Merge - Wait For All Five Sources | success | 1 | did not stall |
| 9 | Code - Normalise And Calculate Metrics | success | **1** | criterion 10 passes |
| 10 | IF - Enough Data To Build Brief? | success | 1 | routed false, correctly |
| 22 | Telegram - Alert Run Blocked | success | 1 | **failed, message never arrived** |

Nodes 11 through 21, 23, 24 and 25 did not execute. That is the designed path for a blocked run.

**The reporting week was `2026-W36`** — `weekStart 2026-08-31T00:00:00.000+01:00`,
`weekEnd 2026-09-07T00:00:00.000+01:00`. The run happened on a Sunday, so "the seven days ending
the Sunday before the run" gives the week before last. On the intended Monday 07:00 schedule it
would give the week just ended. Not a fault, but PRD §12 item 9 is still open and this is worth
re-checking once the week boundary is settled.

---

## 3. FINDING 1 — CRITICAL. A source with no credential was reported as healthy.

**This is the most serious thing the run found, and it is new. QA predicted half of it.**

HubSpot and Supabase both have **no credential at all**. Both carry identical error settings —
`onError: continueRegularOutput`, `alwaysOutputData: true`, `retryOnFail: true, maxTries: 3`.
They behaved differently.

```json
"HubSpot - Fetch Deals And Pipeline":  { "json": { "error": "Node does not have any credentials set" } }
"Supabase - Fetch Product Usage":      { "json": {} }
```

Node 9 detects failure with `typeof items[0].json.error !== 'undefined'`. So it got one right and
one wrong:

```json
"hubspot":  { "ok": false, "rowCount": 0, "note": "Node does not have any credentials set" },
"supabase": { "ok": true,  "rowCount": 1, "note": "" },
"finance":  { "ok": true,  "rowCount": 1, "note": "" },
"support":  { "ok": true,  "rowCount": 1, "note": "" },
"snapshot": { "ok": true,  "rowCount": 1, "note": "" }
```

```json
"missingSources": ["HubSpot deals and pipeline"],
"dataComplete": false,
"dataWarning": "Data warning: HubSpot deals and pipeline could not be read this week. Figures from that source is missing, not zero."
```

**Supabase is absent from `missingSources`.** A source that could not be read at all was counted as
present, with one row. `CLAUDE.md` states: *"Never let one failed source produce a complete-looking
brief. Name the missing dataset on the brief itself."* This is the mechanism that breaks that rule.

The 11 ms execution time is the giveaway — no real network call completes that fast. HubSpot took
10,022 ms because it genuinely retried three times.

**Severity: CRITICAL. It does not block today**, because Supabase contributes nothing to the brief
yet (QA MAJOR 8) and the run stops at node 10 regardless. It becomes live harm the moment either
MAJOR 8 is closed or a Supabase credential is added.

**Owner: Coder, with a Designer ruling on the detection rule.** Do not detect failure by the
presence of an `error` key alone — it is not consistent across node types.

---

## 4. FINDING 2 — CRITICAL. The alert failed and the run still reported success.

`Telegram - Alert Run Blocked` was fully wired — real credential `xkNXmRcwoz0Htiqy`, real numeric
chat id `SET_ME_telegram_chat_id`. It still failed:

```json
{ "json": { "error": "Bad Request: chat not found" } }
```

**Leo received nothing.** The node carries `onError: continueRegularOutput`, so the failure was
swallowed, and the execution finished `status: success, finished: true`.

**Cause: a Telegram bot cannot start a conversation.** The person must message the bot first, from
their own account. The chat id is correct — it is Leo's user id — but the Executive Rollup Bot has
never received a message from him. Reading the id from `@userinfobot` does not create that
conversation, because `@userinfobot` is a different bot.

**Fix: Leo opens his own Executive Rollup Bot in Telegram and sends it any message.** No workflow
change is needed. This should be re-tested before anything else.

**This is exactly the silent failure QA described under BLOCKER 4**, and the first execution
reproduced it on the first try. The credential existing was necessary and not sufficient.

---

## 5. FINDING 3 — CONFIRMED. QA v2's NEW-1 is real.

`alwaysOutputData: true` emits one blank item, not zero:

```json
{ "json": {}, "pairedItem": { "item": 0 } }
```

Measured on all three Google Sheets nodes, each reading a real sheet with headers and no data rows
for `2026-W36`. Node 9's `items.length === 0` branch is now unreachable, exactly as QA predicted.
Each empty sheet was classified `ok: true, rowCount: 1` — a phantom row.

The predicted junk trend point also appeared, verbatim:

```json
"revenueTrend": [ { "weekLabel": "", "revenue": null, "synthetic": false } ]
```

Node 12 filters it on `Number.isFinite(p.revenue)`, so it is harmless today. Node 12 was not
reached this run.

**QA was right to name NEW-1 and right to say do not fix it from a prediction.** The measured shape
matches the prediction, but Finding 1 shows the guard QA drafted would still be wrong — it would
have classified Supabase as an empty week rather than as a failure. **The fix must handle both
cases, and it could not have been written correctly before this run.**

---

## 6. What went right, and is now proven rather than assumed

1. **The five-input Merge did not stall** with a failure item on input 0 and blank items on three
   others. This was explicitly unproven — QA v1 called Merge v3.2's runtime behaviour UNPROVEN.
2. **Node 9 emitted exactly one item.** `itemsOutput: 1`. The one-paid-call guarantee now has
   evidence behind it, not just structure.
3. **The run stopped exactly where it should**, with an honest reason:
   `"No metrics were computed: Finance Actuals had no rows for 2026-W36."`
4. **`alreadyAppended: false`** — the dedupe read worked against the real snapshot sheet.
5. **No number was invented.** `metrics: []`, `pipeline.byStage: []`, movement all zeros. Nothing
   was filled in.
6. **Nothing was written anywhere.** No snapshot row, no Google Doc, no PDF, no email.
7. **No money was spent.** Node 11 never ran.

---

## 7. Verdict on the five criteria in scope

| # | Criterion | Verdict |
|---|---|---|
| 10 | Merge receives 5 inputs, node 9 outputs exactly 1 item | **PASS** — measured, `itemsOutput: 1` |
| 11 | Node 9 classifies all five sources correctly; item shapes recorded | **FAIL** — Supabase misclassified `ok: true`. Shapes now recorded in `n8n-gotchas.md` |
| 20 | A broken source: run continues, warning raised, no row written | **PARTIAL PASS** — HubSpot behaved exactly right. Supabase did not. No row written |
| 21 | Header-only sheet classified `ok: true, rowCount: 0`, no warning | **FAIL** — classified `rowCount: 1` with a phantom row. This is NEW-1, as amended by QA v2 |
| 22 | Break Finance: `readyToBuild` false, node 11 never reached, node 22 fires | **PARTIAL PASS** — routing and blocking are perfect, node 11 never reached. **The alert did not arrive** |

**Two clean passes, two fails, two partials. Nothing dangerous happened, and every fault is a
reporting-honesty fault rather than a data-loss one.**

---

## 8. What has to happen next, in order

1. **Leo messages his own Executive Rollup Bot in Telegram.** One message, any text. Then re-run
   and confirm the alert arrives. Nothing else should be tested until alerting works, because
   every other failure mode reports itself through Telegram.
2. **Designer rules on how a failed source is detected**, given that `error` keys are not reliable
   across node types. An execution-time or a per-node expected-shape check are both candidates.
3. **Coder fixes node 9's `classify()`** to the Designer's ruling — handling both the blank-item
   empty case and the blank-item failure case, which look identical.
4. Then re-run criteria 11, 20 and 21.

**Not in scope and not touched:** criteria 12 to 19 and 23 to 26 are gated on the metric list;
33 and 34 are gated on node 11 being enabled, which needs Leo's explicit word.

---

## 9. Hard limits observed

- Nothing was repaired. Every finding is routed, not fixed.
- No node was modified. No connection changed. Node 11 stayed `disabled: true`.
- The workflow is still `active: false`.
- No write to HubSpot or Supabase. No snapshot row.
- The spending guard was never worked around. Every attempt is in `spend-log.txt`.

---

## 10. Re-test — execution 537. FINDING 2 is closed.

**Run at 2026-09-13T16:33:04Z, after Leo pressed Start on his own bot. `status: success`, 13.877s.**

`Telegram - Alert Run Blocked` now succeeds. The real Telegram API response, verbatim:

```json
{
  "ok": true,
  "result": {
    "message_id": 3,
    "from": { "id": 8751301493, "is_bot": true, "first_name": "Executive Rollup", "username": "ExecutiveRollupBot" },
    "chat": { "id": SET_ME_telegram_chat_id, "first_name": "Dream", "username": "onmydreammmss", "type": "private" },
    "date": 1789317198,
    "text": "Run blocked for 2026-W36:\nNo metrics were computed: Finance Actuals had no rows for 2026-W36."
  }
}
```

**The alert arrived on Leo's phone.** Chat id `SET_ME_telegram_chat_id` is confirmed correct — it was never the
problem. The bot is `@ExecutiveRollupBot`, id `8751301493`.

**The only change between execution 536 and 537 was Leo pressing Start on the bot.** No workflow
edit, no credential change, no parameter change. That isolates the cause completely: the chat had
to be opened from the human side first.

**Criterion 22 now passes in full** — routing, blocking, node 11 never reached, and the alert
actually delivered. It was a PARTIAL PASS on execution 536 for the delivery failure alone.

Every other observation from execution 536 reproduced identically on 537: HubSpot named its own
failure, Supabase returned a blank item and was still misclassified `ok: true`, and the three
Google Sheets nodes each emitted one blank item.

**FINDING 1 stands, unchanged and still CRITICAL.** It is now the only open fault from this pass.

---

## 11. Execution 538 — the first run of the reworked graph. 2026-09-14.

Run from the main session, `method: direct`, manual, no unlock needed — the spending guard was
removed earlier the same day. `status: success`, `finished: true`, 15.992 seconds, 11 nodes.
**Zero cost. Node 11 never reached and still `disabled: true`.**

This run existed to settle QA v3's NEW-4 and NEW-5. **It settled both.**

### QA v3 NEW-5 — RESOLVED. The Merge does NOT stall.

This was the worst-shaped risk in the build: a silent stall on an unattended Monday, with no
error, no alert and no red execution.

`Merge - Wait For All Five Sources`: `status: success`, `itemsOutput: 1`, **executionTime 2 ms**.

```json
{ "json": {}, "pairedItem": [ { "item": 0 }, { "item": 0, "input": 1 } ] }
```

Every input has two connections feeding it and only one carries data on any given run. **The Merge
fired anyway.** `chooseBranch` / `waitForAll` waits on **inputs**, not on individual connections.
Node 9 then emitted exactly one item. **Criterion 10 passes again on the current graph, and the
design decision at ruling section 4.2 is proven safe.**

### QA v3 NEW-4 — SETTLED, and case (c) is REAL.

`$('<node>').all(1, 0)` **works exactly as documented.** Proven by HubSpot:

```json
"hubspot": { "mainItemCount": 0, "blankItemCount": 0, "errorBranchCount": 1,
             "errorBranchThrew": false, "errorBranchMessage": "" }
```

`errorBranchThrew` is `false` on all five sources. The read never threw. **Cases (a) and (b) are
dead — the fail-closed guard will never fire spuriously.** HubSpot was correctly classified
`state: "failed"` and is the only entry in `missingSources`.

**But Supabase proves case (c) is real:**

```json
"supabase": { "mainItemCount": 1, "blankItemCount": 1, "errorBranchCount": 0,
              "errorBranchThrew": false, "errorBranchMessage": "" }
```

**Supabase has no credential. It genuinely cannot be read. It did not route to the error output at
all** — it put one blank item on the *main* output and reported success. Node 9 therefore
classified it `state: "empty", ok: true, rowCount: 0` and listed it under `emptySources`, not
`missingSources`.

**A dead source is still being reported as a healthy empty week.** It is better than execution
536, where it was reported as a healthy week *with one row of data* — the phantom row is gone —
but the classification is still wrong.

**Why this is dangerous, and why it is not dangerous today.** `dataComplete` came out `false` only
because HubSpot failed loudly. **If HubSpot had a working credential and Supabase were the only
dead source, `dataComplete` would be `true`, node 19 would route true, and node 20 would append a
snapshot row on data that was never read.** That is exactly the harm QA NEW-4 predicted.

**The important nuance nobody should skip.** The cause is not `.all(1, 0)`, which works. The cause
is that **the Supabase node with no credential at all does not raise an error** — it returns
nothing and `alwaysOutputData` pads it with a blank item. HubSpot, in the identical situation,
raises "Node does not have any credentials set" properly. **Two nodes, same missing-credential
state, same error settings, different behaviour** — the same inconsistency recorded after
execution 536, now proven to survive the move to `continueErrorOutput`.

**This may be an artefact of having no credential at all**, which is a temporary state Leo chose.
A credentialed Supabase node hitting a real network failure would probably route to the error
output correctly. **Nobody has proven that, and it must not be assumed.**

### The three Google Sheets — correctly classified

```json
"finance":  { "mainItemCount": 1, "blankItemCount": 1, "errorBranchCount": 0 }
"support":  { "mainItemCount": 1, "blankItemCount": 1, "errorBranchCount": 0 }
"snapshot": { "mainItemCount": 1, "blankItemCount": 1, "errorBranchCount": 0 }
```

All three have a working credential, read successfully, and hold no rows for `2026-W36`. All three
came out `state: "empty"`, `rowCount: 0`, and appear in `emptySources`. **That is correct.** Note
the diagnostics are byte-identical to Supabase's — which is precisely why shape alone can never
separate the two cases, exactly as the Designer ruled.

### Everything else this run proved

- **The phantom row is gone, on every path.** Every `rowCount` is `0`. `metrics: []`,
  `pipeline.byStage: []`, and **`revenueTrend: []`** — the junk `{ weekLabel: "", revenue: null }`
  point from execution 536 no longer exists.
- **`emptySources` works** and carries all four readable-but-empty sources.
- **`dataWarning` names only the source that actually failed** — HubSpot, and nothing else.
- **The run stopped honestly:** `readyToBuild: false`,
  `"No metrics were computed: Finance Actuals had no rows for 2026-W36."`
- `alreadyAppended: false`. No snapshot row written. No Doc, no PDF, no email, no paid call.

### Verdict on QA v3's new criteria

| # | Criterion | Verdict |
|---|---|---|
| 36 | Node 8 fires, node 9 outputs exactly one item on the current graph | **PASS** |
| 37 | `supabase.errorBranchCount >= 1`, `state: "failed"`, in `missingSources` | **FAIL** — `errorBranchCount: 0`, `state: "empty"`. See above |
| 38 | Three Sheets sources `state: "empty"`, `rowCount: 0`, `blankItemCount: 1`, in `emptySources` | **PASS** — exactly as specified |

### What must happen next

1. **Designer rules on Supabase.** The error-branch approach works for nodes that raise errors and
   does nothing for nodes that do not. Options: the Plan B marker Set nodes with `isExecuted`; a
   per-source expected-shape check for Supabase only; or accepting it and revisiting once Supabase
   has a credential, since the no-credential case may be artificial.
2. **Do not activate until item 1 is settled.** The failure mode is a snapshot row written on data
   that was never read.
3. Criterion 37 stays FAILED until re-measured.

---

## 12. Execution 539 — all five sources classify correctly. QA v3 NEW-4 is CLOSED.

Run 2026-09-14, `method: direct`, manual. `status: success`, 17.423 seconds, 11 nodes.
**Zero cost. Node 11 never reached, still `disabled: true`.**

**The only change since execution 538: Leo created the HubSpot and Supabase credentials and the
Coder attached them.** No code changed. No connection changed. That isolates the cause completely.

### The finding — the Supabase fault was an artefact of having NO credential

Execution 538 showed Supabase, with no credential, reporting success and emitting a blank item on
its **main** output, so node 9 called it an empty week. QA v3 NEW-4 case (c). The worry was that
`onError: continueErrorOutput` might be unreliable in general.

**It is not. With a credential attached, Supabase routes its failure to the error output exactly
as designed.** Raw node output on execution 539, both branches:

```json
// main output (branch 0) - one blank item from alwaysOutputData
[ { "json": {}, "pairedItem": [ { "item": 0, "input": 0 } ] } ]

// error output (branch 1) - the real failure, correctly routed
[ { "json": { "timezone": "Europe/London", "weekStart": "2026-08-31T00:00:00.000+01:00",
              "weekEnd": "2026-09-07T00:00:00.000+01:00", "weekLabel": "2026-W36",
              "runId": "2026-W36", "triggeredAt": "2026-09-13T19:39:44.917+01:00",
              "error": "Could not find the table 'public.SET_ME_supabase_table_name' in the schema cache" },
    "pairedItem": { "item": 0 } } ]
```

**A node with no credential at all is a special, artificial case.** It is not how a real failure
behaves. Do not generalise from it — and note that nobody would have known this without attaching a
credential and running it.

### Node 9's classification — correct on all five, for the first time

```json
"hubspot":  { "state": "empty",  "ok": true,  "rowCount": 0, "note": "Empty week: the source was read successfully and had no rows." },
"supabase": { "state": "failed", "ok": false, "rowCount": 0, "note": "Could not find the table 'public.SET_ME_supabase_table_name' in the schema cache" },
"finance":  { "state": "empty",  "ok": true,  "rowCount": 0, "note": "Empty week: the source was read successfully and had no rows." },
"support":  { "state": "empty",  "ok": true,  "rowCount": 0, "note": "Empty week: the source was read successfully and had no rows." },
"snapshot": { "state": "empty",  "ok": true,  "rowCount": 0, "note": "Empty week: the source was read successfully and had no rows." }
```

```json
"missingSources": [ "Supabase product usage" ],
"emptySources":   [ "HubSpot deals and pipeline", "Finance actuals", "Support log", "Previous weekly snapshot" ],
"dataComplete": false,
"dataWarning": "Data warning: Supabase product usage could not be read this week. Figures from that source is missing, not zero."
```

**Every source is in the right bucket.** The one source that genuinely could not be read is the
only one named as missing. The four that were read and held nothing are named as empty. The
warning names Supabase and nothing else.

### HubSpot connected and read real data

`executionTime: 489 ms`, error branch empty, main output one blank item. **HubSpot authenticated
successfully with the Service Key credential and returned zero deals** for the week
2026-08-31 to 2026-09-07. That is a real empty read, not a failure, and `state: "empty"` is right.

**Criterion 4 is still not fully settled.** The withdrawn-MAJOR-1 question was whether deals arrive
carrying `amount` and `dealstage` rather than as bare ids. **Zero deals came back, so that still
has not been proven either way.** It needs a week that actually contains deals.

### The `SET_ME_` placeholder did exactly its job

`SET_ME_supabase_table_name` failed loudly and **named itself in the error message**. That is the
behaviour QA v1 predicted when it ruled the four placeholders safe. It is now proven rather than
assumed.

### Verdict on QA v3's added criteria

| # | Criterion | Verdict |
|---|---|---|
| 36 | Node 8 fires, node 9 outputs exactly one item | **PASS** — `itemsOutput: 1` again |
| 37 | `supabase.errorBranchCount >= 1`, `state: "failed"`, in `missingSources` | **PASS** — was FAIL on 538. All three conditions met |
| 38 | Readable-but-empty sources are `state: "empty"`, `rowCount: 0`, in `emptySources` | **PASS** — now on four sources, not three |

### One diagnostic quirk, harmless

`sourceDiagnostics.supabase` reports `mainItemCount: 0, blankItemCount: 0` even though the main
branch did carry one blank item. That is because `classify()` returns at rule 1 the moment the
error branch has an item, before it ever reads the main branch. The counters keep their initial
zeros. **The classification is right; only the diagnostic is incomplete.** Not worth a change —
but do not read `mainItemCount: 0` as proof the main branch was empty.

### Still true

No snapshot row. No Doc, no PDF, no email. No paid call. `readyToBuild: false` with the honest
reason: *"No metrics were computed: Finance Actuals had no rows for 2026-W36."*

---

## 13. Executions 540 and 541 — the first end-to-end run. 2026-09-14.

Leo pasted twelve sample metric rows into Finance Actuals, all marked `synthetic = TRUE`.
Definitions and expected results are in `docs/knowledge/metric-definitions.md`. **PRD §12 item 5
is still OPEN** — these are samples chosen by Claude, not Leo's real metrics.

### Execution 540 — FAILED, and found a real bug no review had caught

`status: error`, 33.1 s. First run where `readyToBuild` was ever `true`.

**Node 13 `HTTP Request - Create Google Doc From HTML` returned the raw Node.js HTTP response
object instead of the parsed JSON body.** Its item's top-level keys were `_events`,
`_readableState`, `socket`, `statusCode` and so on — an unconsumed `IncomingMessage`. The real
body was sitting inside `_readableState.buffer[0].data` as raw bytes.

The Doc itself was created correctly — `statusCode: 200`, and the buried body read:

```json
{ "kind": "drive#file", "id": "1ttIZhVihIPvvLl6411AFAG78rEHXO2A07Sw1q3Q-g_o",
  "name": "Executive Brief - 2026-W36", "mimeType": "application/vnd.google-apps.document" }
```

But `$('HTTP Request - Create Google Doc From HTML').item.json.id` was `undefined`, so node 14
failed:

```
NodeApiError: The resource you are requesting could not be found
nodeName: "Google Drive - Export Brief As PDF"
```

Node 14 has `onError: stopWorkflow`, so the run ended. Nodes 15 to 20 never executed.

**Three QA passes read node 13 and passed it.** The node's `options.response.response.responseFormat`
was already `"json"`, which looks correct and is a schema-valid path. **Only a run exposed that it
did not take effect.**

**The fix, and the cause.** The Coder read the live httpRequest 4.5 schema and changed exactly one
leaf: `responseFormat` `"json"` → `"autodetect"`. The reasoning: the request uses
`contentType: "raw"` for the multipart body, which turns off JSON handling at the request layer,
and the explicit `json` branch assumes that layer already parsed. `autodetect` is the branch that
buffers the stream and parses it by content type.

The Coder also corrected an assumption in the brief it was given: the apparently missing
`content-type` header was a red herring. On a Node `IncomingMessage`, `headers` is a prototype
getter and only `rawHeaders` is an own property, so `headers` never survives serialisation.

### Execution 541 — SUCCESS. The workflow reached the approval step for the first time.

`status: waiting`, 23.4 s, **15 nodes executed**. The only change from 540 was that one leaf.

**Node 13 now emits the parsed body:**

```json
{ "kind": "drive#file", "id": "1LnygXP_y_UfENBU35HKdswaey8TCO8ff3EkFf__8JqI",
  "name": "Executive Brief - 2026-W36", "mimeType": "application/vnd.google-apps.document" }
```

**Node 14 produced a real PDF:**

```json
"binary": { "data": { "mimeType": "application/pdf", "fileType": "pdf",
  "fileName": "Executive Brief - 2026-W36", "fileSize": "61.2 kB", "bytes": 61201,
  "id": "filesystem-v2:workflows/iIypy4KWtaqsowAQ/executions/541/binary_data/43a5c370-..." } }
```

**Node 15 sent the Telegram approval and the execution went to `status: waiting`.**

### CRITERION 14 — the design's riskiest assumption is PROVEN

Integrator priority item 3 asked whether Google Drive's HTML-to-Doc conversion **fetches remote
`<img src>` URLs** during conversion. The whole of Route A depends on it. If it did not, node 13
and node 14 would have to be replaced by a different design (Route B).

**It does. Two independent confirmations:**

1. **Leo opened the Doc and confirmed he can see the charts.**
2. **The exported PDF is 61,201 bytes.** A brief of this length with a four-row table and no images
   is 3 to 8 kB. 61 kB is consistent with two embedded 600×350 raster images and nothing else.

**Route A is proven. Route B is not needed and that contingency can be closed.**

### MAJOR 6 — partially advanced, not closed

The PDF binary **is present on node 15's output**, carried as a `filesystem-v2` reference rather
than inline base64. So it survives *into* the approval node. **Whether it survives the resume after
the pause is still unproven** — that needs the approval actually tapped.

### One thing worth knowing about disabled nodes

Node 11 `OpenAI - Write Executive Narrative` reports `status: success` in the execution data with
an output identical to node 10's. **A disabled node passes its input straight through** — it does
not vanish from the run. It did not call OpenAI and nothing was billed. Do not read
`status: success` on node 11 as evidence a paid call happened; check `disabled` on the node.

Node 12 correctly fell back to its "Narrative unavailable this week" text, which is the designed
behaviour when node 11 produces no narrative.

### Metrics — computed exactly as predicted

Week `2026-W36`, `readyToBuild: true`, `blockReason: ""`.

| Metric | Actual | Target | vs target | Week over week |
|---|---|---|---|---|
| Revenue | 52400 | 50000 | +4.8% | +8.7% |
| New Customers | 38 | 35 | +8.6% | +22.6% |
| Churned Customers | 9 | 5 | +80.0% | +28.6% |
| Weekly Active Users | 4380 | 4500 | −2.7% | +6.3% |

**Every figure matches the prediction written into `metric-definitions.md` before the run.** The
calculation chain is correct.

`missingSources: ["Supabase product usage"]` — correct, its table name is still a placeholder.
`emptySources: ["HubSpot deals and pipeline", "Support log", "Previous weekly snapshot"]` —
correct, all three read successfully and held nothing for that week.

### Housekeeping

**Two stray Google Docs now exist in Leo's My Drive root**, one per run:
- `1ttIZhVihIPvvLl6411AFAG78rEHXO2A07Sw1q3Q-g_o` (execution 540, the failed run)
- `1LnygXP_y_UfENBU35HKdswaey8TCO8ff3EkFf__8JqI` (execution 541)

Every run creates one and nothing deletes them. That is QA criterion 29 — no Drive folder parent is
set. They should be cleaned up after testing.
