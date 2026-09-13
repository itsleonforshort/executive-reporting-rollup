# Ruling — how a failed source is told apart from an empty one

**Version:** v1 ruling, amends `docs/architecture/v1-architecture.md` sections 9 and 10.
**Date:** 2026-09-14
**Author:** Designer agent
**Rests on:** executions 536 and 537 of workflow `iIypy4KWtaqsowAQ`, real data.
**Status:** design only. Nothing was created, changed or run on the n8n instance.

This document replaces architecture section 10 step 2 (the three-outcome table) and the
`onError` column of the section 9 retry table for nodes 3 to 7. Everything else in
`v1-architecture.md` stands unchanged.

---

## 1. The ruling

**Stop asking the item what happened and ask n8n. All five fetch nodes move from
`onError: continueRegularOutput` to `onError: continueErrorOutput`, and each node's second
output is wired into the same Merge input its first output already uses — so the Merge
keeps exactly five inputs and every input still receives at least one item in every case.
Node 9 then classifies a source as FAILED when, and only when, that node's error output
holds at least one item, read with `$('<node>').all(1, 0)` inside a try/catch. Item shape
stops being the signal, because execution 536 proved item shape cannot be a signal: a dead
Supabase and a healthy empty Google Sheet both emit `{ "json": {} }`, byte for byte, and no
amount of JavaScript can separate two identical bytes. `alwaysOutputData: true` stays on all
five — it caused more harm than it has yet prevented, but removing it now would trade a
fault we can detect for an unmeasured stall on the only join in the workflow, so instead it
is stripped of all meaning: its blank item is padding to keep the Merge fed, it is never
evidence of health, and it is never counted as a row. `rowCount` counts only items carrying
at least one key, so an empty source reports `0` and the phantom row disappears at source.
The three outcomes become an explicit `state` field — `ok`, `empty`, `failed` — and only
`failed` reaches `missingSources`.**

---

## 2. The evidence this rests on

1. **Two node types, same failure, different output.** HubSpot with no credential emitted
   `{ "json": { "error": "Node does not have any credentials set" } }`. Supabase with no
   credential emitted `{ "json": {} }`. Same settings, same cause, different bytes.
2. **A failed Supabase and an empty Google Sheet are identical.** Both `{ "json": {} }`.
   This is the whole ruling in one line. Any rule that reads the item can only ever get one
   of the two right.
3. **Node 9 reported `supabase: { ok: true, rowCount: 1 }` for a source it never reached.**
   That breaks `CLAUDE.md` twice over — a failed source made the brief look complete, and a
   row that does not exist was counted.
4. **Every empty sheet reported `rowCount: 1`.** Three of them. A number the workflow did
   not fetch.
5. **The Merge did not stall** with an error item on input 1 and blank items on inputs 3, 4
   and 5, and node 9 emitted exactly one item. Both proven, both reproduced on execution 537.
6. **What was NOT proven:** that the Merge fires when one input receives *zero* items. No
   input was ever starved, because `alwaysOutputData` was on. QA BLOCKER 3 was raised from
   n8n's documentation, not from a run, and it has still never been observed.
7. **`$("<node>").all(branchIndex, runIndex)` is documented**, `branchIndex` being "the
   output branch of the node to use", with the worked example `$("IF").all(1, 0)` for the
   false output. It is documented, not yet measured on this instance — see section 5.

---

## 3. The options, and why the others lose

| Option | Verdict |
|---|---|
| **Error output (`continueErrorOutput`) + `.all(1, 0)`** | **CHOSEN.** n8n's own router decides whether the node threw. It is the only signal that does not depend on what the failing node chose to print, so it works for HubSpot, for Supabase, and for the three Sheets nodes whose failure shape nobody has measured yet. It is the same answer whether a credential is missing, a token expired, or a sheet id is wrong. |
| Per-source expected-shape check (a finance row must carry `week_label`) | **REJECTED as the detector, ACCEPTED as a row filter.** It cannot tell failed from empty — both produce zero shape-valid rows, which is the exact question being asked. Worse, it turns a renamed column into "empty week", which is the same lie in a new coat. It is kept only for what it is good at: deciding whether an item counts as a row. |
| Looking for an `error` key | **REJECTED. Already disproven on real data.** HubSpot names its failure, Supabase does not. This is the rule being replaced. |
| n8n runtime metadata inside the Code node | **REJECTED as the primary.** n8n exposes no per-node error status to a Code node. `$('X')` throwing means "never executed", and a node that failed under `continueRegularOutput` *did* execute, so the throw never fires for the case we care about. `$helpers` being undefined on this instance is the standing warning against designing on a runtime API nobody has run here. The `branchIndex` argument is used, but it is used to read data n8n routed, not to interrogate node status. |
| Execution timing (the 11 ms tell) | **REJECTED outright.** It is a coincidence, not a contract. A cached Sheets read can be fast and a dying API can be slow. Encoding a millisecond threshold into a reporting workflow is how you get a brief that is wrong on a good network day. |
| Fail closed — treat every blank item as failed | **REJECTED.** It removes the phantom row, and it would be safe, but it erases the empty-week outcome the architecture exists to protect. A support log with no tickets would raise a warning every quiet week and block the snapshot row, permanently gapping the trend the snapshot sheet exists to hold. Alert fatigue plus a poisoned history is a worse end state than the bug. |

---

## 4. What the Coder must change

Five node settings, five new connections, one Code node rewritten. **No node is added. No
node is removed. No write is added. The Merge keeps `numberInputs: 5`.**

### 4.1 Nodes 3, 4, 5, 6, 7 — one setting each

On `HubSpot - Fetch Deals And Pipeline`, `Supabase - Fetch Product Usage`,
`Google Sheets - Fetch Finance Actuals`, `Google Sheets - Fetch Support Log`,
`Google Sheets - Fetch Last Week Snapshot`:

| Setting | From | To |
|---|---|---|
| `onError` | `continueRegularOutput` | **`continueErrorOutput`** |
| `alwaysOutputData` | `true` | **`true` — unchanged, see 4.5** |
| `retryOnFail` / `maxTries` / `waitBetweenTries` | `true` / `3` / `5000` | unchanged |

Nothing else on those five nodes changes. Parameters, credentials and `range` stay as they are.

### 4.2 Five new connections

Each fetch node's **output index 1** (the error output) goes to the **same Merge input its
output 0 already uses**:

```
HubSpot - Fetch Deals And Pipeline          output 1 -> Merge - Wait For All Five Sources, input 0
Supabase - Fetch Product Usage              output 1 -> Merge - Wait For All Five Sources, input 1
Google Sheets - Fetch Finance Actuals       output 1 -> Merge - Wait For All Five Sources, input 2
Google Sheets - Fetch Support Log           output 1 -> Merge - Wait For All Five Sources, input 3
Google Sheets - Fetch Last Week Snapshot    output 1 -> Merge - Wait For All Five Sources, input 4
```

**No existing connection is removed or re-pointed.** Connection entries go from 20 to 25.
The Merge still has five inputs, at indexes 0 to 4, in the designed order.

**Why wire the error output into the Merge at all, when node 9 reads it by name?** Because
on a failure, output 0 may deliver nothing, and whether a five-input `chooseBranch` /
`waitForAll` Merge fires with a starved input is still unmeasured. Feeding both outputs into
the one input guarantees that input always receives at least one item — the exact state
execution 536 proved works. The Merge is `output: empty` and carries no data, so what lands
there is irrelevant to everything downstream.

### 4.3 Node 9 — the classifier

Replace `classify()` and its five call sites. Keep the rest of the node as it is. It stays
`mode: runOnceForAllItems` and it still ends in a single `return [{ json: {...} }]`.

**Rule order, and it must be this order:**

1. Read the error branch: `$(nodeName).all(1, 0)` inside try/catch. **If it returns one or
   more items → `state: 'failed'`.** `note` is `json.error.message`, or `String(json.error)`,
   or `'The node reported an error.'` if neither exists, truncated to 300 characters.
   A throw here is treated as "no items on the error branch" and recorded in diagnostics —
   it is never treated as a failure on its own.
2. Read the main branch: `$(nodeName).all(0, 0)` inside try/catch. **A throw here →
   `state: 'failed'`**, note names the node and the message.
3. Belt and braces: any main-branch item carrying an `error` key → `state: 'failed'`. This
   should never fire once 4.1 lands. If it ever does, it means a node type routes its error
   to the main output anyway, and that is worth knowing.
4. Drop every item whose `json` has zero keys. **A key-less item is not a row.**
5. **Zero rows left → `state: 'empty'`, `ok: true`, `rowCount: 0`**, note
   `'Empty week: the source was read successfully and had no rows.'`
6. **Otherwise → `state: 'ok'`, `ok: true`, `rowCount: rows.length`**, note `''`.

**Both `.all()` calls must pass `branchIndex` explicitly.** The documented default is "the
output that connects this node to the node where the expression is used", and after 4.2 two
outputs connect to the same place. Never call `.all()` with no arguments in this node.

**Output shape changes, precisely:**

- `sources.<key>` gains `state` (`'ok' | 'empty' | 'failed'`). `ok` stays, as the derived
  boolean `state !== 'failed'`, so nodes 10, 12 and 19 need no edit.
- `missingSources` is built from `state === 'failed'` only. An empty source never appears there.
- **New: `emptySources`**, plain-English labels of every source with `state === 'empty'`.
- **New: `sourceDiagnostics`**, one entry per source: `mainItemCount`, `blankItemCount`,
  `errorBranchCount`, `errorBranchThrew`, `errorBranchMessage`. Nothing on the brief reads
  it. It exists so that one execution measures everything in section 5 at once instead of
  needing three runs.
- `dataComplete`, `dataWarning`, `readyToBuild`, `blockReason`, `metrics`, `pipeline`,
  `revenueTrend`, `week`, `alreadyAppended` — all keep their names and meanings.

**Three call sites that must use `state`, not `ok`:**

- `dealRows = hubspot.state === 'ok' ? hubspot.rows : []` — with blank items already filtered
  out, the "one deal opened" and the `unknown` bar at zero both stop being possible.
- `alreadyAppended` is computed only when `snapshot.state` is `'ok'` or `'empty'`. When the
  snapshot fetch failed, `alreadyAppended` is meaningless; it stays `false`, and the append
  is blocked anyway because `dataComplete` is false and node 19 tests it. Confirm that chain
  by eye — it is the only thing standing between a failed snapshot read and a duplicate row.
- `revenueTrend` is built from filtered rows only, so the junk point
  `{ weekLabel: '', revenue: null }` cannot be produced. **Leave node 12's
  `Number.isFinite(p.revenue)` filter in place** as defence in depth.

**`readyToBuild` logic is unchanged and stays correct.** A week where all five sources are
genuinely empty now gives `missingSources: []` and `dataComplete: true`, then stops at
`metrics.length === 0` with the honest Finance reason. A week where all five failed gives
`missingSources.length === 5` and stops with the all-sources reason. Both right.

### 4.4 Node 11's prompt — the two lists are not the same instruction

The OpenAI prompt already receives `missingSources`. It must now receive `emptySources` too,
with a different instruction beside it:

- **Missing:** say nothing about this area at all. Do not estimate it, do not infer it.
- **Empty:** this is a real, verified zero. It may be described as none or nil, and it must
  never be described as unavailable.

That is section 10 step 4 finally being able to do what it was written to do. The same split
applies to the brief HTML in node 12: `missingSources` prints as the warning banner it
already prints; `emptySources` prints as one plain line, not as a warning.

### 4.5 `alwaysOutputData` — the crux, answered straight

**It caused more harm than it has prevented.** The harm is measured: it turned a total
Supabase failure into `ok: true, rowCount: 1` and put a phantom row on three healthy sheets,
twice, on real data. The prevention is still hypothetical: the Merge stall it was added to
stop has never been observed, and cannot be observed while the setting is on, because with it
on no input is ever starved. QA BLOCKER 3 was raised from documentation, not from a run.

**It stays on all five anyway, and this is not a dodge.** Removing it today would swap a
fault I can now detect, name and fix for an unmeasured risk sitting on the single join that
every run passes through. If that risk is real, the workflow produces no brief at all on any
week a source is empty — a worse failure than the one being fixed, and one that would only
show up on a Monday morning. The correct move is to defang it, not to delete it on a guess.
So: **its blank item is padding to keep the Merge fed. It is never evidence that a source is
healthy, and it is never a row.** After 4.3, nothing in the workflow reads meaning into it.

**And here is the free experiment that can retire it for good.** After the fix is in and the
re-test in section 5 passes, set `alwaysOutputData: false` on **`Google Sheets - Fetch Support
Log` only** — a sheet that is empty, has a working credential, and contributes no number to
the brief today — and run once. Node 11 stays disabled, so it costs nothing.

- If the Merge still fires and node 9 still emits one item, the stall was imaginary. Take the
  setting off all five and the blank item stops existing at all. Strictly better.
- If the Merge stalls, QA's guess was right. It stays on all five forever, and for the first
  time there is a measurement behind it instead of a document.

Either result is a permanent answer to a question that has been open since QA v1. One run.

### 4.6 Plan B, if the error-branch read does not work

If the re-test in section 5 shows `$('<node>').all(1, 0)` cannot see the error item, do
**not** go back to shape checks. Add five `Set` nodes instead, one on each fetch node's
output 1, named `Set - Mark HubSpot Fetch Failed` and so on, each assigning
`__fetchFailed = true` and `__fetchNote`, each feeding the same Merge input as before. Node 9
then reads `$('Set - Mark HubSpot Fetch Failed').isExecuted` — executed means that source
failed. The graph change in 4.2 is the same either way, so nothing built now is wasted. This
is the fallback, not the plan: it costs five nodes on the canvas and it leans on a different
runtime behaviour, so it is only worth it if the cheaper read fails.

### 4.7 Constraints, checked

| Constraint | Held |
|---|---|
| Node 9 emits exactly one item | Yes. Classification is internal; the single `return` is untouched. |
| No Split In Batches / Split Out / Item Lists / Loop Over Items / sub-workflow | Yes. None added. |
| Node 9 stays the place classification happens | Yes. No Set-node chain. |
| The snapshot row stays the only write | Yes. Nothing added writes anywhere. |
| No number printed that was not fetched | Yes. That is what 4.3 rule 4 is for. |
| No workflow-level `executionTimeout` | Yes. `settings` is not touched. |
| Works while HubSpot and Supabase are parked, and after they are connected | Yes. The error branch fires for a missing credential today and for an API error later. Nothing in the rule mentions credentials. |
| Merge still has five inputs | Yes. `numberInputs: 5`, indexes 0 to 4, original order. Five connections added, none removed. |

---

## 5. Still unproven — must be re-tested before this is trusted

The workflow stays off until items 1 to 4 pass. One run measures all of them, because
`sourceDiagnostics` records everything at once. Node 11 stays disabled; the run costs nothing.

1. **`$('<node>').all(1, 0)` returns the error item** for a node set to
   `continueErrorOutput`. Prove it on `Supabase - Fetch Product Usage`, the node that caused
   this ruling. This is the load-bearing assumption.
2. **Two outputs of one node into one Merge input is accepted at runtime**, the Merge still
   fires, and **node 9 still emits exactly one item** — criterion 10 must be re-proven, not
   assumed to survive.
3. **Supabase is classified `state: 'failed'` and appears in `missingSources`.** That is
   criterion 11, and it is the whole point.
4. **An empty sheet is classified `state: 'empty'`, `rowCount: 0`, absent from
   `missingSources`, no warning raised** — criterion 21 as QA amended it.
5. **The Google Sheets failure shape**, recorded by breaking one sheet id (criterion 20).
   The ruling no longer depends on it, which is why it is safe to keep testing it, and it
   belongs in `n8n-gotchas.md` regardless.
6. **The `alwaysOutputData` experiment in 4.5.** Separate run, after 1 to 4 pass.
7. **Later, once credentials exist:** a healthy HubSpot week with no matching deals produces
   no phantom deal and no `unknown` bar. Latent today, live the day Leo connects HubSpot.

---

## 6. For Leo

1. A source that was read fine but had no rows this week — should the snapshot row still be
   written for that week? I say yes, because an empty week is a real zero and skipping the
   row leaves a permanent hole in the trend.
2. Should an empty source be named on the brief in one plain line, such as "Support log: no
   tickets this week", or stay silent? I recommend naming it, as a line and not as a warning.
3. May the Tester have one free run with `alwaysOutputData` turned off on the Support Log
   node only, to settle whether the Merge ever stalls? Nothing is sent, nothing is written,
   nothing is paid.
