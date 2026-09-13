# QA Report — v1

**Date:** 2026-09-13
**Workflow:** `iIypy4KWtaqsowAQ`, "Executive Reporting Rollup - Weekly Leadership Brief"
**Instance:** self-hosted, `https://<your-n8n-instance>`
**Audited against:** `docs/PRD.md` v1.3, `docs/architecture/v1-architecture.md` v1, `docs/reports/integrator-v1.md`, `docs/reports/coder-v1.md`
**Author:** QA agent. Written to disk by the main session, because the agent has no write tool.

**Method:** live `n8n_get_workflow` (structure, filtered x6, details), `n8n_validate_workflow` strict
profile, `get_node` schema reads for telegram, supabase, hubspot, gmail, googleSheets, merge,
scheduleTrigger and openAi, and official n8n documentation for the `sendAndWait` output shape.
Nothing audited from memory. Nothing was changed. Nothing was run.

`n8n_validate_workflow` returns `valid: true, 0 errors, 20 warnings`. **That is a statement about
JSON shape. It is not a verdict.** All four blockers below pass validation cleanly.

---

## Findings

### BLOCKER 1 — Node 16 reads the approval result at the wrong path. An approved brief can never be sent.

**Node:** `IF - Was The Brief Approved?`
**Parameter:** `conditions.conditions[0].leftValue`

```
={{ $('Telegram - Request Brief Approval').item.json.approved === true }}
```

n8n's official approvals documentation shows the `sendAndWait` result **nested under `data`**:

```json
{ "data": { "approved": true, "respondedAt": "2025-07-13T12:34:56.000Z", "responder": { } } }
```

`$json.approved` is therefore `undefined`. `undefined === true` is `false`.

**What happens at runtime.** Leo taps Approve. Node 16 takes the **false** branch anyway. No email
to leadership. No Telegram summary. No snapshot row. Node 21 posts "Nobody responded within the
6-hour approval window" — after he just responded. PRD steps 16, 17 and 18 never execute, on every
run, forever. The failure is completely silent: the execution ends green.

This came from `integrator-v1.md` item 6, which states "n8n's shared sendAndWait mechanism writes a
top-level `approved` boolean". The only official documentation that exists contradicts that.

**Fix, named not applied.** Read `data.approved`, keeping the strict form. A shape that is correct
either way costs nothing:

```
={{ ($('Telegram - Request Brief Approval').item.json.data?.approved ?? $('Telegram - Request Brief Approval').item.json.approved) === true }}
```

**Owner:** Coder for the fix. Integrator owns the wrong claim. **Blocking.**

---

### BLOCKER 2 — Node 12 uses `.item` inside a Code node running "Run Once for All Items"

**Node:** `Code - Build Brief HTML And Chart URLs`, `mode: runOnceForAllItems`

```js
const metricsData = $('Code - Normalise And Calculate Metrics').item.json;
const ai = $('OpenAI - Write Executive Narrative').item.json;
```

`docs/knowledge/n8n-gotchas.md` records, proven by running on this instance: **"`$json` does not
exist in a Code node set to 'Run Once for All Items'."** `.item` resolves against the same
current-item context that `$json` comes from. Node 9 — same Coder, same job, adjacent node —
correctly uses `.first()`, which is the form architecture section 5 prescribes. The inconsistency is
the tell.

**What happens at runtime.** Either node 12 throws on its first line on every run and the whole run
dies before the Doc is built (no `onError` is set, so the default stop applies), or it resolves and
the risk is latent. Neither state is proven.

**The second use is worse.** It sits inside a `try/catch`. If it throws, the catch swallows it and
the brief prints "Narrative pending: the OpenAI node is disabled..." — *after OpenAI has already
been billed*. A paid call whose result is silently discarded is exactly what `money-facts.md` exists
to prevent.

**Fix.** Replace both with `.first()`. There is no loop node anywhere in this workflow, so `.first()`
is provably safe here and architecture section 5 already justifies it.

**Owner:** Coder. **Blocking.**

---

### BLOCKER 3 — The five source fetches are missing `alwaysOutputData: true`

**Nodes:** 3, 4, 5, 6, 7.

Architecture section 10, step 1 — the mechanism the whole missing-source design rests on:

> Each of nodes 3 to 7 is set to `onError: continueRegularOutput` with **`alwaysOutputData: true`**.
> So each branch always delivers something to the Merge. The Merge therefore never stalls waiting on
> a dead branch.

Live, all five carry `onError`, `retryOnFail`, `maxTries` and `waitBetweenTries`. **None carries
`alwaysOutputData`.** Every node object returned by `mode: details` was checked.

**What happens at runtime.** A source that legitimately returns zero rows — which is the state of
all three sheets today — emits zero items. Whether a five-input `chooseBranch` / `waitForAll` Merge
still fires on a zero-item input is unproven (Integrator item 11, COULD NOT VERIFY). If it does not,
the run hangs at node 8 and nothing downstream executes: no brief, no alert, no error.

**Second effect.** Node 9's `classify()` returns `{ ok: true, rowCount: 0 }` for a zero-length
array. It cannot tell "the sheet was empty this week" from "that branch produced nothing at all".
That is a dead source quietly becoming a good week, which `CLAUDE.md` forbids in plain words.

**Owner:** Coder. **Blocking.**

---

### BLOCKER 4 — Every alert path in the workflow is Telegram, and no Telegram credential exists

**Nodes:** 15, 18, 21, 22, 25. All five carry `chatId: "SET_ME_telegram_chat_id"` and **no
`credentials` block at all.** `data-sources.md` confirms no Telegram credential exists on this
instance.

**What happens at runtime.** Node 25 (`Telegram - Alert Workflow Failed`) has
`onError: continueRegularOutput`. So an unattended Monday failure fires the Error Trigger, runs node
25, fails to send, swallows the failure, and the execution finishes looking clean. **Nobody is
told.** The same is true of nodes 21 and 22.

It also means the approval gate — PRD step 15, the one human-review point in the whole design —
cannot be exercised at all.

**Owner:** Leo (PRD §12 item 8, OPEN). **Blocking for deployment. Blocks Tester criteria 16, 17, 23,
24 and 25. Does not block testing nodes 1 to 14.**

---

### MAJOR 1 — HubSpot deal properties are set on a parameter that does not exist

**Node:** `HubSpot - Fetch Deals And Pipeline`

```
"additionalFields": { "properties": ["dealname","amount","dealstage","pipeline","closedate","hs_lastmodifieddate"] }
```

All 87 properties on the v2.2 schema were searched. **There is no `additionalFields.properties`.**
The only "Deal Properties to Include" multiOptions field lives at **`filters.properties`**.
`additionalFields` for `deal:search` holds `sortBy`, `direction` and `query`.

**What happens at runtime.** The bare-Deal-ID trap the Integrator flagged as blocking item 7 is
**not** avoided. Every deal returns as an id with no `amount` and no `dealstage`. Node 9 then buckets
every deal into stage `'unknown'` at value 0, and reports `opened = <deal count>`, `closedWon = 0`,
`closedLost = 0`. The pipeline chart draws a single bar labelled "unknown" at zero. **That is a
number printed that the workflow did not fetch.**

`coder-v1.md` §2 states this trap was avoided. It was not.

**Owner:** Coder, with the Integrator to re-confirm the correct path for `deal:search` specifically.
Non-blocking today only because HubSpot has no credential.

---

### MAJOR 2 — The HubSpot date filter pairs `GTE` with `type: "string"`, a combination the schema does not offer

```
{ "propertyName": "hs_lastmodifieddate", "type": "string", "operator": "GTE", "value": "={{ Date.parse(...) }}" }
```

Read from the schema: `operator` offers `GT`, `GTE`, `LT`, `LTE` **only when `type` is `number`**
(`showWhen: { type: ["number"] }`). Under `type: "string"` the options are `CONTAINS_TOKEN`, `EQ`,
`HAS_PROPERTY`, `NOT_HAS_PROPERTY`, `NEQ`.

**What happens at runtime.** Unproven. At best HubSpot accepts the range. At worst the filter is
dropped and the search returns the entire deal set. Separately, the n8n editor will not display
`GTE` for a string filter, so anyone who opens this node may silently reset it.

**Owner:** Coder. Non-blocking today.

---

### MAJOR 3 — Node 13 retries a non-idempotent create

**Node:** `HTTP Request - Create Google Doc From HTML`, `retryOnFail: true, maxTries: 3,
options.timeout: 60000`

A POST that creates a Google Doc is not idempotent. A 60-second timeout on a request Google actually
completed produces a second, then a third Doc for the same week — identical names, My Drive root, no
folder. The approval link points at whichever id came back last. `retryOnFail` also retries a 400 or
403, which can never succeed: two wasted calls.

**Owner:** Coder. Non-blocking.

---

### MAJOR 4 — Node 20 retries a non-idempotent append

**Node:** `Google Sheets - Append Weekly Snapshot Row`, `retryOnFail: true, maxTries: 3`

Google Sheets `append` is not idempotent either. A timed-out-but-successful append retried twice
writes the same week's row up to three times. **This is the workflow's only write and the thing the
whole dedupe design exists to protect.**

**Owner:** Coder. Non-blocking, but it defeats the stated dedupe key under exactly the conditions
dedupe is for.

---

### MAJOR 5 — The dedupe check and the write are up to six hours apart

`alreadyAppended` is computed in node 9, from the snapshot read at node 7, at the very start of the
run. The append happens at node 20, after a wait of up to six hours at node 15.

**What happens at runtime.** Two runs started inside that window — a manual re-run while the first
is still waiting, or a `misfirePolicy: coalesce` catch-up alongside a manual run — both read "no row
yet", both pass node 10, both append. Two rows for one week. `weekLabel` is the stated dedupe key
and nothing enforces it at write time.

**Owner:** Designer to rule — re-read the sheet immediately before the append, or accept and
document. Non-blocking; low likelihood, high consequence.

---

### MAJOR 6 — The PDF binary is assumed to survive the approval pause. Nothing proves it.

Node 14 writes binary property `data`. Node 17 attaches
`options.attachmentsUi.attachmentsBinary[0].property = "data"` (path and field name verified correct
against the Gmail v2.2 schema). Between them sit node 15, a `sendAndWait` that returns a fresh
approval item, and node 16.

**What happens at runtime if the binary is dropped.** Node 17 fails with "binary property data not
found", `onError: stopWorkflow` ends the run — after a human has already approved. No email, no
summary, no snapshot row. PRD step 16 fails permanently.

The Coder flagged this on the node's notes and left it. Nobody has proven it.

**Owner:** Tester to prove; Coder to fix if it fails.

---

### MAJOR 7 — The snapshot row carries no numbers, so PRD §9 cannot be met

Node 20 writes six columns: `week_label, week_start, week_end, run_at, synthetic, notes`. No metric
value, no target, no pipeline figure, no revenue.

**Consequence, traced.** Node 9 sets `revenue: null` on every historical point it reads back from
that sheet, so node 12's `trendPoints.length >= 2` gate can never pass, so the revenue trend chart
in PRD §6 and step 12 can never render from real history. PRD §9 "Weekly trends are visible from
historical snapshots" is unreachable by construction.

Root cause is PRD §12 item 5 (the metric list) still OPEN — `metric-definitions.md` is deliberately
empty. The Coder did not create this gap. It arrives exactly where it was always going to arrive.

**Owner:** Leo / Designer. Blocking for "done against the PRD". Not blocking a Tester run.

---

### MAJOR 8 — Supabase and the Support Log contribute nothing to the brief

Node 9 builds `metrics` only from Finance Actuals rows, and `pipeline` only from HubSpot. The
`supabase` and `support` results are classified, counted, and then never used again.

**What happens at runtime.** A brief with `dataComplete: true` and no warning banner, containing
zero product-usage numbers and zero support numbers — against a PRD that names both as sources (§3)
and asks for them to be normalised and calculated (§4 steps 8 and 9). **A complete-looking brief
that quietly omits two of five datasets.**

**Owner:** Designer / Leo, gated on PRD §12 item 5. Non-blocking for testing, blocking for done.

---

### MAJOR 9 — Node 12 reads the OpenAI result by guessing four field names

```js
narrative = ai && (ai.content || ai.text || ai.output_text || (ai.message && ai.message.content) || '');
```

If none matches, the brief prints "Narrative pending: the OpenAI node is disabled until it is
reviewed and switched on." That sentence is factually wrong once the node is enabled, it goes into a
document destined for leadership, and the call has already been paid for.

**Owner:** Coder. Must be replaced with the real field, read off one live execution, before node 11
is ever enabled a second time. Non-blocking while node 11 is disabled.

---

### MINOR 1 — QuickChart uses a long GET URL against the Integrator's explicit recommendation

Integrator item 2: "The Coder should always use `chart/create` for the revenue trend line, which
grows every week, rather than waiting for a failure." Node 12 builds
`https://quickchart.io/chart?width=600&height=350&c=<encoded config>` for both charts. Today the
trend has no points. It gains one label and one value every week. When it eventually passes the
practical URL limit the chart becomes a broken image inside the Doc, silently. **Owner:** Coder.

### MINOR 2 — `synthetic` is hardcoded `false` on the only write

Any Tester run reaching node 20 writes a row indistinguishable from a real one, which then blocks the
real Monday run for that week through `alreadyAppended`, and enters the trend as real history.
`CLAUDE.md` requires every non-real row to be marked. **Owner:** Tester protocol below; Coder if a
test flag is wanted.

### MINOR 3 — `'Europe/London'` is hardcoded five times inside node 2

Once as the `timezone` field, four times inside the `$now.setZone(...)` expressions — and the three
date formulas never read the field they sit beside. PRD §12 item 9 (time zone and exact week
boundary) is still OPEN, so that string *will* change, and it will have to change in five places plus
the workflow setting. **Owner:** Coder.

### MINOR 4 — `range` is set and ignored on four Google Sheets nodes

Nodes 5, 6, 7 and 20 carry `range: "A:F"` / `"A:H"` alongside
`options.dataLocationOnSheet.values.rangeDefinition: "detectAutomatically"`. The validator reports
`range` "won't be used". Harmless, but a reader will believe the read is bounded to six columns when
it is auto-detecting the header row. **Owner:** Coder, cosmetic.

### MINOR 5 — `appendAttribution` is inconsistent across leadership-facing messages

Node 17 sets it `false`. Node 15 sets it `true`. Nodes 18, 21, 22 and 25 leave `additionalFields`
empty so the default applies. The approval message and the leadership Telegram summary will carry an
n8n attribution line. **Owner:** Coder.

---

## Observations

**O1.** Every pipeline stage value and every revenue figure is JSON-encoded into a public
`quickchart.io` GET URL and embedded in the Doc. **Business numbers leave the building in a URL
anyone holding it can re-render.** Free service, no auth. Not a defect against any rule currently
written down; Leo should know.

**O2.** Neither Code node has `onError`, so a bug in either stops the run and reaches the Error
Trigger. Deliberate per architecture section 9, and QA agrees — a loud failure beats a silently wrong
brief.

**O3. Naming.** All 25 nodes follow `Original Name - Action`. Checked one by one against the live
workflow. No exceptions.

**O4. Connections.** The `connections` object was read directly and compared to architecture section
3, line by line. The five-way fan-out from node 2; the five Merge inputs at indexes 0, 1, 2, 3, 4 in
the designed order (HubSpot 0, Supabase 1, Finance 2, Support 3, Snapshot 4 — no off-by-one); the
main line; all three IF nodes with both outputs connected; the Error Trigger island. **20 connection
entries, no dangling output, no dead branch. It matches.**

**O5. Live settings confirmed on the instance,** not read from the local file:
`timezone: Europe/London`, `errorWorkflow: iIypy4KWtaqsowAQ` (points at itself, so the Error Trigger
will fire), and **no `executionTimeout` key at all** — which is what `n8n-gotchas.md` and Integrator
item 13 require for a workflow that pauses six hours. Correct.

**O6.** `triggerCount: 0`, `totalExecutions: 0`, `active: false`, `pinData: null`. Never run, never
activated, no pinned data hiding a result.

---

## The safety-critical traces

### One OpenAI call per brief — what is proven, what is only designed

Path: node 8 Merge → node 9 Code → node 10 IF → node 11 OpenAI.

- **Node 8** — `mode: chooseBranch`, `chooseBranchMode: waitForAll`, `output: empty`,
  `numberInputs: 5`. All four re-read from the schema; `output: empty` is labelled "A Single, Empty
  Item" and is offered only in this exact mode combination; `numberInputs` accepts 5. **Designed, not
  proven.** The schema says what the option is called, not what the node does when one input carries
  an error-shaped item or zero items. Integrator item 11 says the same.
- **Node 9** — `mode: runOnceForAllItems`, and the function's only `return` is
  `return [{ json: {...} }];`. The whole body was read. There is no path that returns more than one
  item. **This is proven by reading the code**, independent of the Merge. Even if the Merge emitted
  5,000 items, node 9 emits exactly one.
- **Node 10** — an IF. It routes, it never multiplies.
- **Node 11** — `retryOnFail: true, maxTries: 2`. Worst case from one execution is two charges.
- **No fan-out node exists** anywhere between node 1 and node 11. All 25 node types were checked. No
  `Split In Batches`, no `Split Out`, no `Item Lists`, no `Loop Over Items`, no sub-workflow call.
  Architecture section 6 Lock 4 holds.

**Verdict.** Node 9 alone guarantees one item reaches node 11, and that guarantee is proven by
reading source, not by trusting an unverified Merge. Lock 1 remains unproven and must still be
tested, but **it is not load-bearing for the cost guarantee.** The single-call requirement is
structurally sound.

**Node 11 is `disabled: true` on the live workflow. Confirmed.**

### The only-one-write claim

- Node 20 appends to document `1TdTEZ1Jh-4Ne62LmwFfza25pYSttembIuji1o--VElA`, gid `1425370562` — the
  Weekly Snapshot sheet in `data-sources.md`. Correct target, `mappingMode: defineBelow`, all six
  columns named explicitly in the sheet's own order.
- Node 3 is `deal:search`, node 4 is `row:getAll`. **Nothing writes to HubSpot or Supabase.**
- Nodes 5, 6, 7 are `operation: read`. No other sheet is written.
- **One exception nobody has stated plainly.** Node 13 creates a Google Doc. That is a write to
  Google Drive. PRD step 13 requires it, so it is intended — but `CLAUDE.md`'s sentence "the workflow
  is read-only apart from one write: the snapshot row" is not literally true of this build, and the
  Doc creation is unbounded: one file per run, forever, in My Drive root, with no cleanup.
- **Can the append run twice for one week?** Not inside one execution — one item reaches node 20.
  Across executions, yes: see MAJOR 5 and MAJOR 4.

### A failed source must not produce a complete-looking brief

| Source fails | Node 9 | The brief | Snapshot |
|---|---|---|---|
| HubSpot errors | `ok: false`, named in `missingSources`, `dataComplete: false` | warning banner first line of the Doc, of the approval message and of the Telegram summary; Pipeline section reads "No pipeline data available this week." | blocked at node 19 |
| Supabase errors | same | banner only — nothing in the brief was ever going to come from Supabase (MAJOR 8) | blocked |
| **Finance errors** | same, **plus** `metrics.length === 0` → `readyToBuild: false` | no brief at all; node 22 alert carrying `blockReason` | blocked, **and no paid call** |
| Support errors | same | banner only (MAJOR 8) | blocked |
| Snapshot errors | same, **plus** `alreadyAppended` stays `false` because the check is skipped | banner; no revenue trend | blocked at node 19 — which is the only thing preventing a duplicate row |

**Empty rather than failed.** Any sheet returning zero rows is classified `ok: true, rowCount: 0` and
treated as a real empty week, with no warning. That is correct behaviour — and it is the case
BLOCKER 3 makes unsafe.

**One fact the Tester needs before it starts.** Finance Actuals currently holds a header row and no
metric rows, because `metric-definitions.md` is empty by design. So `metrics.length === 0`, so
`readyToBuild` is `false`, so **a run today stops at node 10 and does nothing.** And HubSpot and
Supabase have no credentials, so `dataComplete` can never be `true`, so **node 20 is unreachable**
without seeding data first.

### No snapshot row on a failure, a rejection or a timeout

- **Failed run:** `onError: stopWorkflow` on nodes 13, 14, 15, 17 and 20 ends the execution before
  the append. Confirmed by reading each node.
- **Rejected:** node 16 false → node 21 → end. Node 20 is not on that branch. Confirmed from the
  `connections` object.
- **Timeout:** the same branch, because `=== true` treats an absent key and `false` identically.
  Confirmed — **once BLOCKER 1 is fixed.** Today it is accidentally correct for the wrong reason:
  everything routes false.
- **Partial data:** node 19 false → node 23 → end. Confirmed.

### Node 16 handling `approved` being absent

**In form, yes, and QA credits it.** `=== true` inside the expression collapses `undefined`, `null`
and `false` to a real boolean before the `boolean/equals` operator with `typeValidation: "strict"`
ever sees it, so a missing key cannot throw. That is the right defensive shape.

It is defeated by BLOCKER 1, because the path is wrong, so it is *always* the absent case. Fix the
path; keep the `=== true`.

---

## Rulings on the five items the Coder flagged

### 1. Node 2's five `$now` calls — HONOURS the rule. Accepted, with one change required.

`CLAUDE.md` says: "The week boundary is decided in one place, then passed down. Never recomputed per
node." **The rule is about place, not count.** All five calls sit inside node 2. No other node calls
`$now`, `DateTime.now()` or any relative date expression — all 25 were checked. Node 9 does call
`DateTime.fromObject`, but it builds the previous week label *from the `weekLabel` node 2 already
decided*. That is passing down, not recomputing.

The architecture's "expect exactly three hits" was a proxy for the rule. A proxy is not the rule.

The Coder was right not to gamble. Whether a Set node assignment can read a sibling set moments
earlier in the same node is genuinely unsettled, and getting it wrong would have left `runId` — the
dedupe key, and nothing else — blank at runtime. Duplicating a five-token formula to remove that risk
is a good trade.

`triggeredAt` is an improvement, not scope creep. Without it, `run_at` would have needed `$now` at
node 20, *outside* node 2, which would have broken the rule for real.

**The one change required is MINOR 3.** `'Europe/London'` is hardcoded four times in the expressions
plus once as a field the expressions never read, and PRD §12 item 9 is still OPEN, so that string is
going to change. Reduce it to one place, or write the five-edit requirement on the node's notes.
Five silent copies of a value known to be provisional is not acceptable.

### 2. The HubSpot placeholder filter — ACCEPTABLE in an unrunnable branch. Conditionally.

PRD §12 item 4 is OPEN and PARKED, and `CLAUDE.md` says a build may not start on an open item.
Strictly, node 3 should not exist yet. But the Designer put it in the architecture, the architecture
was approved, the node has no credential, and it cannot reach HubSpot. An unrunnable branch carrying
an honest placeholder is not the same as shipping a wrong business rule.

**What makes it acceptable:** it is named on the node's own `notes`, named in the Coder's report,
`hs_lastmodifieddate` is a technical field nobody will mistake for a business decision, and it cannot
execute.

**What makes it conditional:** MAJOR 1 and MAJOR 2. The placeholder is not merely provisional, it is
*malformed*. A placeholder is allowed to be wrong about the business. It is not allowed to be wrong
about the schema — because that is what lets everyone believe the node is "fully shaped and testable
once a credential exists" when it is not. Fix MAJOR 1 and MAJOR 2, keep the placeholder, and it
passes.

### 3. No Drive folder parent — A DEFECT. Non-blocking. Fix before deployment, not before testing.

The Coder's reasoning for not pointing at a trashed folder is right and QA would have made the same
call.

But "My Drive root" is not a neutral outcome. One new file per run, forever, with a name that repeats
whenever a week is re-run, loose among everything else in that Google account, no cleanup, no owner.
Within a year that is fifty-two documents. Combined with MAJOR 3 it can be more than one a week. And
the approval link Leo taps points into that pile.

Architecture assumption A9 says "The Doc and the PDF live in one named Drive folder". The assumption
was not met and was not escalated as a change — it was quietly dropped. **That silence is the part
ruled a defect, not the decision.**

**Required:** Leo names one folder in `<owner-google-account>`, the id goes into
`docs/knowledge/data-sources.md`, and node 13's metadata part gains `"parents": ["<id>"]`. One line
in the body expression.

### 4. The four `SET_ME_` placeholders — CONFIRMED SAFE. No change.

Read on the live workflow:

- `Supabase - Fetch Product Usage`: `tableId: "SET_ME_supabase_table_name"`;
  `SET_ME_supabase_date_column` in `orderBy` and in both `filters.conditions[].keyName`.
- `chatId: "SET_ME_telegram_chat_id"` on nodes 15, 18, 21, 22 **and 25** — that is five nodes, not
  four fields. One edit lands in five places.
- `Gmail - Send Brief To Leadership`: `sendTo: "SET_ME_leadership_email_list"`.

None can be mistaken for real data. None is a plausible table name, a numeric chat id or an email
address. Each **fails loudly at its own node rather than half-working**: Gmail rejects a string with
no `@`, Telegram rejects a non-numeric chat id, Supabase rejects a table that does not exist. That is
the right property for a placeholder.

**A sixth placeholder nobody listed:** node 14's `fileId.cachedResultName` is the literal string
"Google Doc created by node 13 (dynamic id)". Cosmetic, the `value` beside it is a real expression.
Recorded here so it is not later mistaken for a fault.

### 5. The two expression warnings — BOTH FALSE POSITIVES. Confirmed by reading the text.

**Node 13, `body`.** The whole expression was read. Every n8n reference carries `$`:
`$('Code - Build Brief HTML And Chart URLs').item.json.docTitle` and `.html`. The bare identifiers the
linter reacts to are `JSON` in `JSON.stringify(...)` — a JavaScript global — and the literal text
`application/json` inside a header string, which is not a variable at all. **False positive.**

**Node 25, `text`.** Same read. The only n8n reference is `$json`, which has its `$`. Everything else
(`d`, `wfName`, `nodeName`, `errMsg`, `e`) is a local declared inside the IIFE. **False positive.**

Ruled on the expression text. But note: node 13's body has a real problem the linter cannot see
(MAJOR 3), and node 12 feeding it has BLOCKER 2. **"The warnings are benign" must not be read as
"these nodes are fine."**

---

## Acceptance criteria — the definition of done

All 34 must be true. The Tester verifies. King checks against this list.

### Group A — true before any execution

1. Node 16's `leftValue` and node 21's `approved` read both use the field path n8n's `sendAndWait`
   actually emits, still with a strict `=== true`, so `false`, `null` and an absent key all route to
   node 21.
2. Node 12 contains zero `.item` references. Both cross-node reads use `.first()`.
3. Nodes 3, 4, 5, 6 and 7 each carry `alwaysOutputData: true` alongside their existing
   `onError: continueRegularOutput`.
4. Node 3 sets deal properties on a parameter that exists in the HubSpot v2.2 schema, and its date
   filter's `type` and `operator` are a pair the schema offers together.
5. Node 11 is still `disabled: true`.
6. Node 17's `sendTo` is Leo's own address — not `SET_ME_leadership_email_list`, and not any
   leadership address.
7. Node 20 targets document `1TdTEZ1Jh-4Ne62LmwFfza25pYSttembIuji1o--VElA`, gid `1425370562`,
   `mappingMode: defineBelow`, six columns in the order
   `week_label, week_start, week_end, run_at, synthetic, notes`.
8. `settings` has no `executionTimeout` key, and `settings.errorWorkflow` is `iIypy4KWtaqsowAQ`.
9. No `Split In Batches`, `Split Out`, `Item Lists`, `Loop Over Items` or `Execute Sub-workflow` node
   exists anywhere in the workflow.

### Group B — proven by running, OpenAI still disabled

10. One execution reaches node 9 with the Merge having received all five inputs, and node 9 outputs
    exactly one item. The Merge's own output item count is read off the execution and recorded.
11. On that execution, node 9's `sources` object classifies all five correctly against what each
    source actually held, and `dataComplete` matches. The exact item shape a
    `continueRegularOutput` failure produces, and the exact shape a Google Sheets read of an empty
    sheet produces, are both pasted into `docs/knowledge/n8n-gotchas.md`.
12. With at least two real metric rows for the current `weekLabel` and two for the previous
    `weekLabel` in Finance Actuals, `readyToBuild` is `true` and the run reaches node 12.
13. Node 12 produces `html`, `telegramSummary` and `weekLabel`, and does not throw.
14. Node 13 returns a Google Doc id, and opening that Doc by hand shows **both QuickChart images
    actually rendered inside it**. If they are not there, Route A has failed and Route B replaces
    nodes 13 and 14 — a Designer decision, not a Coder patch.
15. Node 14 returns a binary property named `data` containing a PDF that opens.
16. Node 15 sends one approval message, and only one, and the execution enters a waiting state.
17. On Approve, node 16 takes the true branch and node 17 receives a binary property `data` it can
    attach. The email arrives with the PDF on it.
18. The snapshot sheet gains **exactly one** new row: `week_label` equal to the ISO week label node 2
    computed, `week_start` and `week_end` equal to node 2's ISO strings, `run_at` a real timestamp,
    `synthetic` FALSE, `notes` empty when all five sources were read.
19. Re-running in the same week yields `readyToBuild: false` with `blockReason` naming the existing
    row, the run stops at node 10, and the snapshot sheet still holds exactly one row for that week.

### Group C — proven by deliberate breakage

20. Point one Google Sheets fetch at a document id that does not exist. The run continues; that
    source appears in `missingSources`; `dataComplete` is `false`; the `dataWarning` sentence is the
    first line of the brief HTML, the first line of the approval message and the first line of the
    Telegram summary; node 19 takes the false branch; **no row is added to the snapshot sheet.**
21. Repeat against a sheet that exists but holds only a header row. That source is classified
    `ok: true, rowCount: 0`, no warning is raised, and the brief treats it as a real zero.
22. Break Finance Actuals specifically. `readyToBuild` is `false`, the run stops at node 10, node 22
    fires, **node 11 is never reached**, and the execution shows node 11 with zero input items.
23. Decline the approval. No email, no Telegram summary, no snapshot row — and node 21's message says
    the brief was **declined**, not that nobody responded.
24. Let the approval expire. Same three outcomes, and node 21's message says it **expired**. The item
    n8n emits on expiry is recorded in `docs/knowledge/n8n-gotchas.md`.
25. Force node 13 to fail. The execution stops, the Error Trigger fires, node 25 runs, and a message
    **actually arrives** in the Telegram chat naming node 13.
26. No test row is left in the snapshot sheet with `synthetic` FALSE. Every row written during testing
    is deleted by hand or marked.

### Group D — before the workflow is switched on

27. A Telegram credential exists and is attached to nodes 15, 18, 21, 22 and 25, and `chatId` is a
    real chat id in all five.
28. Node 17's `sendTo` is the real leadership list, set only after criterion 17 passed with Leo's own
    address.
29. Node 13's request body names a real Drive folder parent, and that folder id is recorded in
    `docs/knowledge/data-sources.md`.
30. `metric-definitions.md` holds a block for every metric in the KPI table, and Finance Actuals holds
    a row per metric per week. PRD §12 item 5 is closed.
31. PRD §12 item 9 is closed: the time zone and the exact week boundary are written down, and node 2
    matches them.
32. The real HubSpot business filter from PRD §12 item 4 replaces `hs_lastmodifieddate`, or node 3 is
    disabled and the brief names HubSpot as missing every week.
33. Node 11 is enabled by Leo, one call is made, the real cost is recorded in
    `docs/knowledge/money-facts.md`, and the execution shows node 11 received **exactly 1 input item**
    and produced exactly 1 output item.
34. Node 12 reads the narrative from the real field name proven in 33, and the brief carries the
    narrative, not "Narrative pending".

---

## What the Tester must do

**Prerequisites, or the run does nothing:**

1. Finance Actuals needs at least two metric rows for the current week label and two for the previous
   one. Without them every run stops at node 10.
2. HubSpot and Supabase have no credentials. Expect both classified failed on every run, so
   `dataComplete` is `false`, so **node 20 is unreachable** without simulating.
3. Telegram has no credential. **Nothing past node 14 can be tested until Leo creates one.**

**Test with real data:** criteria 10 to 19, in order. Stop at 15 if the Telegram credential does not
exist.

**Deliberately break:** criteria 20 to 25. Break Finance Actuals first — it is the one that proves the
paid call is never reached.

**Nodes that spend money or touch a real external system when this workflow runs. The Tester must
know all seven before it executes anything.**

| Node | What it does for real |
|---|---|
| 11 `OpenAI - Write Executive Narrative` | **The only node that spends money.** Currently `disabled: true`. Keep it disabled for the entire first pass. Enable only on Leo's word, after criterion 22 has passed. |
| 13 `HTTP Request - Create Google Doc From HTML` | Creates a real Google Doc in Leo's Drive on **every** run. Free, but leaves a file each time, in My Drive root, with no folder. Delete them after testing. |
| 15 `Telegram - Request Brief Approval` | Sends a real Telegram message and pauses the execution for up to six hours. |
| 17 `Gmail - Send Brief To Leadership` | **Sends a real email.** Safe only while `sendTo` is the placeholder. Point it at Leo's own inbox before the first run that can reach it. |
| 18, 21, 22, 25 | Real Telegram posts. |
| 20 `Google Sheets - Append Weekly Snapshot Row` | **Writes a real row to the live snapshot sheet with `synthetic` FALSE.** A test row is indistinguishable from a real one, blocks the real Monday run for that week, and enters the trend as real history. Delete it by hand. |
| Node 12's chart URLs | Sends real pipeline and revenue numbers to `quickchart.io` in a public URL. |

---

## Bottom line

**4 blockers. Not fit to hand to the Tester yet.**

**The worst thing found:** node 16 reads the approval answer at `$json.approved`. n8n puts it at
`data.approved`. Leo taps Approve, the workflow decides he did not, and the brief is never sent to
leadership — silently, on every run, with the execution finishing green. **The workflow's entire
purpose is defeated by one wrong field path that validation cannot see.**

**Route the fixes back to the Coder** for blockers 1, 2 and 3, then run QA again before the Tester
starts. Blocker 4 needs Leo to create the Telegram credential.

---

# CORRECTION — MAJOR 1 is withdrawn. 2026-09-13, after the Coder's rework.

**MAJOR 1 above is wrong. The Coder was right. The node was correct as originally built.**

QA ruled that HubSpot deal properties belong at `filters.properties` and that
`additionalFields.properties` does not exist. The Coder refused the fix, re-read the live schema
twice, and said `additionalFields.properties` is real and unconditional for `deal:search`, while
`filters.properties` belongs to `deal:getAll`.

**The main session checked this independently rather than taking either side's word.** Two
`get_node` searches against `nodes-base.hubspot` on this instance, 2026-09-13:

**Search for `properties` — both paths exist for deals.**

| Path | Display name | Notes |
|---|---|---|
| `filters.properties` | Deal Properties to Include | carries the warning "By default, the results will only include Deal ID", gated `showWhen: @version > 2` |
| `filters.propertiesCollection.propertiesValues.properties` | Deal Properties to Include | the older collection form |
| `additionalFields.properties` | Field Names or IDs | "Whether to include specific Deal properties in the returned results" |

That alone does not settle which collection belongs to which operation, because the search output
flattens away the resource and operation scoping.

**Search for `sortBy` — this is what settles it.** `sortBy` exists at exactly one path:
**`additionalFields.sortBy`**, and nowhere else. In HubSpot's API, `sortBy`, `direction` and
`query` are **search** parameters. They have no meaning on a plain list call.

**Therefore `additionalFields` is the collection n8n uses for `deal:search`**, and
`additionalFields.properties` is the correct place to name deal properties on a Search. QA had it
backwards: it read the `filters.properties` warning text, which belongs to `deal:getAll`, and
attributed it to the Search operation.

**Ruling: MAJOR 1 is withdrawn. Acceptance criterion 4 is amended** to read: node 3 sets deal
properties at `additionalFields.properties`, and its date filter's `type` and `operator` are a
pair the schema offers together.

**What is still not proven.** The operation scoping was inferred from where `sortBy` lives, not
read directly from a `displayOptions` block, and nobody has run this node against a real HubSpot
account. HubSpot has no credential on this instance. **The Tester must confirm, on the first real
run, that deals come back carrying `amount` and `dealstage` and not as bare ids.** If they come
back bare, this correction is wrong and MAJOR 1 stands.

**The lesson, which is the reason this correction is written out in full.** QA was right to
challenge the node, and the Coder was right to refuse a fix it could not verify. Both did their
job. The finding was resolved by a third read of the schema, not by seniority. Do not let a QA
finding overrule a builder who has re-read the source and can say exactly what it says.
