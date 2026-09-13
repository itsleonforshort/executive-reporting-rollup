# Architecture v1 — Executive Reporting Rollup

**Version:** 1
**Date:** 2026-09-13
**Author:** Designer agent
**Designed against:** `docs/PRD.md` v1.2
**Target instance:** self-hosted, `https://<your-n8n-instance>`
**Status:** design only. Nothing has been created on the n8n instance.

Every node type and every parameter named here was read from the live node schema with
`get_node` during this design pass. Nothing is from memory. Where a fact could not be
proven from the schema, it is listed in section 11 as an Integrator verification item, or
in section 12 as an assumption.

---

## 0. Summary

- **25 nodes.** 20 on the happy path, 5 on the error and alert paths.
- **One trigger** (Schedule) plus **one Error Trigger** in the same workflow.
- **Five source branches**, joined once at a single Merge node.
- **One paid node**, the OpenAI narrative. Guaranteed to fire once per run by four
  independent structural locks, described in section 6.
- **One write**, the snapshot row. Blocked on any partial run, any rejection and any
  approval timeout.
- **No sub-workflows in v1.** Reason in section 9.

---

## 1. The node list, in order

Column `TV` is the `typeVersion` the Coder must set. These are the current versions on
this instance as returned by `get_node` on 2026-09-13.

| # | Node name | n8n node type | TV |
|---|---|---|---|
| 1 | `Schedule Trigger - Weekly Monday 7am` | `n8n-nodes-base.scheduleTrigger` | 1.4 |
| 2 | `Set - Define Reporting Week` | `n8n-nodes-base.set` | 3.5 |
| 3 | `HubSpot - Fetch Deals And Pipeline` | `n8n-nodes-base.hubspot` | 2.2 |
| 4 | `Supabase - Fetch Product Usage` | `n8n-nodes-base.supabase` | 1 |
| 5 | `Google Sheets - Fetch Finance Actuals` | `n8n-nodes-base.googleSheets` | 4.7 |
| 6 | `Google Sheets - Fetch Support Log` | `n8n-nodes-base.googleSheets` | 4.7 |
| 7 | `Google Sheets - Fetch Last Week Snapshot` | `n8n-nodes-base.googleSheets` | 4.7 |
| 8 | `Merge - Wait For All Five Sources` | `n8n-nodes-base.merge` | 3.2 |
| 9 | `Code - Normalise And Calculate Metrics` | `n8n-nodes-base.code` | 2 |
| 10 | `IF - Enough Data To Build Brief?` | `n8n-nodes-base.if` | 2.3 |
| 11 | `OpenAI - Write Executive Narrative` | `@n8n/n8n-nodes-langchain.openAi` | 2.3 |
| 12 | `Code - Build Brief HTML And Chart URLs` | `n8n-nodes-base.code` | 2 |
| 13 | `HTTP Request - Create Google Doc From HTML` | `n8n-nodes-base.httpRequest` | 4.5 |
| 14 | `Google Drive - Export Brief As PDF` | `n8n-nodes-base.googleDrive` | 3 |
| 15 | `Telegram - Request Brief Approval` | `n8n-nodes-base.telegram` | 1.2 |
| 16 | `IF - Was The Brief Approved?` | `n8n-nodes-base.if` | 2.3 |
| 17 | `Gmail - Send Brief To Leadership` | `n8n-nodes-base.gmail` | 2.2 |
| 18 | `Telegram - Post Brief Summary` | `n8n-nodes-base.telegram` | 1.2 |
| 19 | `IF - All Sources OK For Snapshot?` | `n8n-nodes-base.if` | 2.3 |
| 20 | `Google Sheets - Append Weekly Snapshot Row` | `n8n-nodes-base.googleSheets` | 4.7 |
| 21 | `Telegram - Alert Brief Not Sent` | `n8n-nodes-base.telegram` | 1.2 |
| 22 | `Telegram - Alert Run Blocked` | `n8n-nodes-base.telegram` | 1.2 |
| 23 | `NoOp - End Run Without Snapshot` | `n8n-nodes-base.noOp` | 1 |
| 24 | `Error Trigger - Catch Unhandled Failure` | `n8n-nodes-base.errorTrigger` | 1 |
| 25 | `Telegram - Alert Workflow Failed` | `n8n-nodes-base.telegram` | 1.2 |

Node 15 is the approval gate. Section 7 gives the Gmail alternative, which drops into the
same slot without changing anything around it.

### Why 25 and not the PRD's 20

PRD §8 estimates about 20 automation nodes. The happy path here is exactly 20 (nodes 1 to
20). The extra 5 are nodes 21 to 25: three alerts, one explicit end-of-run marker, and the
Error Trigger. PRD §7 requires a missing dataset to be flagged and `CLAUDE.md` requires the
error path to be built at the same time as the happy path. The 20-node estimate did not
price that in. This is reported to Leo in section 13.

---

## 2. What each node does, in plain language

**1. `Schedule Trigger - Weekly Monday 7am`**
Fires every Monday at 07:00. `rule.interval` set to a weekly trigger, `triggerAtDay: [1]`
(Monday), `triggerAtHour: 7`, `triggerAtMinute: 0`. `misfirePolicy` set to
`coalesce` so a Monday missed because the instance was down still runs once when it comes
back, rather than being skipped silently. Time zone is set on the workflow, not on the
node. The zone is still OPEN — see section 12, assumption A1.

**2. `Set - Define Reporting Week`**
The single place the week boundary is decided. Details in section 5. Also carries the
`runId` used as the dedupe key.

**3. `HubSpot - Fetch Deals And Pipeline`**
Reads deals for the reporting week. Read only. The exact operation is an Integrator item
(section 11, item 7) — `deal:search` with a date filter is the likely route, `deal:getAll`
the fallback. Whichever is chosen, the date window comes from node 2 by expression, never
recomputed.

**4. `Supabase - Fetch Product Usage`**
Reads usage rows for the week from the usage table or view. Read only. Table, fields and
project URL are PRD §12 item 3, still OPEN.

**5. `Google Sheets - Fetch Finance Actuals`**
Reads the finance actuals tab. Also the source of the per-metric target, per assumption A5.

**6. `Google Sheets - Fetch Support Log`**
Reads the support log tab for the week.

**7. `Google Sheets - Fetch Last Week Snapshot`**
Reads the snapshot sheet. Serves two jobs at once: it supplies last week's numbers for the
week-over-week comparison and the revenue trend, and it supplies the list of week labels
already present, which is how the run knows whether it has already written a row for this
week. Read only at this point in the flow.

**8. `Merge - Wait For All Five Sources`**
The single join point. `mode: chooseBranch`, `chooseBranchMode: waitForAll`,
`numberInputs: 5`, `output: empty`. It waits for all five branches and then emits exactly
one empty item. It carries no data. That is deliberate — see section 6.

**9. `Code - Normalise And Calculate Metrics`**
PRD steps 8 and 9 in one node, running in **Run Once for All Items** mode. It reads each of
the five fetch nodes by name, decides which succeeded, normalises the rows, and computes
variance against target, week-over-week change, pipeline movement and revenue trend. It
returns exactly one item. Output shape in section 4.

**10. `IF - Enough Data To Build Brief?`**
Tests the single boolean `readyToBuild` that node 9 produced. True goes to the paid call.
False goes to node 22 and the run ends without spending anything.

**11. `OpenAI - Write Executive Narrative`**
`resource: text`, `operation: response` (the Responses API, shown in the schema as "Message
a Model"). Credential "OpenAI account", id `DBmj9DTgWeT0tOGw`, type `openAiApi`. It receives
one item and produces one narrative. The prompt is the metrics JSON plus a hard instruction
set: explain only what the numbers show, never invent a cause, never invent a figure, and
say nothing at all about any dataset listed as missing.

**12. `Code - Build Brief HTML And Chart URLs`**
Assembles the whole brief as one HTML string: the data warning banner if any, the executive
summary, the KPI table with target variance and week-over-week columns, two `<img>` tags
pointing at QuickChart URLs, and the risks and observations. Also builds the short Telegram
summary text and the snapshot row values, so nothing downstream has to recompute anything.

**13. `HTTP Request - Create Google Doc From HTML`**
POSTs the HTML to the Google Drive upload endpoint with
`mimeType: application/vnd.google-apps.document`, which converts it to a real Google Doc.
Authenticated with `predefinedCredentialType: googleDriveOAuth2Api`. Returns the new
document id. `options.timeout` set to 60000. See section 8 for why the Google Docs node
cannot do this job.

**14. `Google Drive - Export Brief As PDF`**
`resource: file`, `operation: download`, `fileId` from node 13, and
`options.googleFileConversion.conversion.docsToFormat: application/pdf`. This option was
read directly from the node schema, so the PDF route exists. Output is binary.

**15. `Telegram - Request Brief Approval`**
`operation: sendAndWait`, `approvalType: double` (Approve and Disapprove),
`chatApproval: true`, `options.limitWaitTime` set to 6 hours. The message carries the data
warning banner first, then the headline numbers, then the link to the Google Doc. The
execution pauses here. Full treatment in section 7.

**16. `IF - Was The Brief Approved?`**
Reads the approval result from node 15. True goes to the send path. False goes to node 21.

**17. `Gmail - Send Brief To Leadership`**
Sends the PDF from node 14 as an attachment, using `options.attachmentsUi.attachmentsBinary`
with the binary field name from node 14. Recipients are PRD §12 item 7, still OPEN.

**18. `Telegram - Post Brief Summary`**
Posts the short summary built in node 12. Truncated by expression to stay under Telegram's
message limit. Chat id is PRD §12 item 8, still OPEN.

**19. `IF - All Sources OK For Snapshot?`**
Tests `dataComplete === true` from node 9. This is the last guard before the only write.

**20. `Google Sheets - Append Weekly Snapshot Row`**
`operation: append`, `columns.mappingMode: defineBelow` with every column mapped
explicitly. The first column is `weekLabel`. The workflow's only write.

**21. `Telegram - Alert Brief Not Sent`**
Fires on a rejection or on an approval timeout. Says which week, says nothing was sent to
leadership, says no snapshot row was written, and gives the Doc link so a human can act.

**22. `Telegram - Alert Run Blocked`**
Fires when node 10 says no. Carries the `blockReason` string from node 9, which says
exactly why: all sources down, no metrics computed, or a row for this week already exists.

**23. `NoOp - End Run Without Snapshot`**
The explicit end of the partial-data path. It exists so the canvas states the intent rather
than leaving a branch dangling.

**24. `Error Trigger - Catch Unhandled Failure`**
A second trigger in the same workflow. The workflow's own **Settings → Error Workflow**
must be pointed at this same workflow for it to fire. The Coder must set that.

**25. `Telegram - Alert Workflow Failed`**
Names the workflow, the failed node and the error message.

---

## 3. The connections, stated plainly

### Fan-out from the week boundary

```
1 Schedule Trigger  ->  2 Set - Define Reporting Week
2 Set - Define Reporting Week  ->  3 HubSpot          (branch 1)
2 Set - Define Reporting Week  ->  4 Supabase         (branch 2)
2 Set - Define Reporting Week  ->  5 Sheets Finance   (branch 3)
2 Set - Define Reporting Week  ->  6 Sheets Support   (branch 4)
2 Set - Define Reporting Week  ->  7 Sheets Snapshot  (branch 5)
```

One source fetch per branch, as the design defaults require.

### Join

```
3 HubSpot          ->  8 Merge, input 1
4 Supabase         ->  8 Merge, input 2
5 Sheets Finance   ->  8 Merge, input 3
6 Sheets Support   ->  8 Merge, input 4
7 Sheets Snapshot  ->  8 Merge, input 5
```

Merged once, at the end of the fetches, as the design defaults require.

### Main line

```
8  Merge                 ->  9  Code - Normalise And Calculate Metrics
9  Code                  ->  10 IF - Enough Data To Build Brief?
10 IF, true output       ->  11 OpenAI - Write Executive Narrative
10 IF, false output      ->  22 Telegram - Alert Run Blocked        [run ends]
11 OpenAI                ->  12 Code - Build Brief HTML And Chart URLs
12 Code                  ->  13 HTTP Request - Create Google Doc From HTML
13 HTTP Request          ->  14 Google Drive - Export Brief As PDF
14 Google Drive          ->  15 Telegram - Request Brief Approval    [execution pauses]
15 Approval              ->  16 IF - Was The Brief Approved?
16 IF, true output       ->  17 Gmail - Send Brief To Leadership
16 IF, false output      ->  21 Telegram - Alert Brief Not Sent      [run ends]
17 Gmail                 ->  18 Telegram - Post Brief Summary
18 Telegram              ->  19 IF - All Sources OK For Snapshot?
19 IF, true output       ->  20 Google Sheets - Append Weekly Snapshot Row
19 IF, false output      ->  23 NoOp - End Run Without Snapshot      [run ends]
```

### Error branch

```
24 Error Trigger  ->  25 Telegram - Alert Workflow Failed
```

Nodes 24 and 25 are not connected to anything else. They form a separate island in the
same workflow.

### Every branch accounted for

| Branch | Where it goes | Run outcome |
|---|---|---|
| 10 true | OpenAI, then the full brief | Brief built |
| 10 false | 22, alert | Nothing sent, nothing written, nothing paid |
| 16 true | 17, email leadership | Brief delivered |
| 16 false | 21, alert | Nothing sent, nothing written |
| 19 true | 20, append | Snapshot row written |
| 19 false | 23, end | Brief sent, snapshot deliberately skipped |
| 24 | 25, alert | Unhandled failure reported |

There are no dangling outputs.

---

## 4. What node 9 produces

One item, this shape. Everything downstream reads from it, so nothing is recomputed twice.

```
{
  week:        { weekStart, weekEnd, weekLabel, runId, timezone },
  sources: {
    hubspot:   { ok, rowCount, note },
    supabase:  { ok, rowCount, note },
    finance:   { ok, rowCount, note },
    support:   { ok, rowCount, note },
    snapshot:  { ok, rowCount, note }
  },
  missingSources: [ "Supabase product usage" ],
  dataComplete:   false,
  dataWarning:    "Data warning: Supabase product usage could not be read ...",
  alreadyAppended: false,
  readyToBuild:    true,
  blockReason:     "",
  metrics: [
    { key, label, unit, value, target, variance, variancePct,
      lastWeek, wow, wowPct, sourceName, available }
  ],
  pipeline:     { byStage: [ { stage, value } ], movement: { opened, closedWon, closedLost, net } },
  revenueTrend: [ { weekLabel, revenue, synthetic } ]
}
```

`available: false` on a metric means its source failed. That metric prints as an em dash
with a footnote. It never prints as zero.

---

## 5. The week boundary — set once, passed down

Decided in **node 2, `Set - Define Reporting Week`**, and nowhere else.

It is an Edit Fields node, not a Code node, and not a DateTime node. Date maths is done
inline with Luxon, per the design defaults. Five fields are assigned:

| Field | How it is computed |
|---|---|
| `timezone` | a literal string, the workflow time zone |
| `weekStart` | `{{ $now.setZone($json.timezone).minus({ weeks: 1 }).startOf('week').toISO() }}` |
| `weekEnd` | `{{ $now.setZone($json.timezone).startOf('week').toISO() }}` — exclusive upper bound |
| `weekLabel` | `{{ $now.setZone($json.timezone).minus({ weeks: 1 }).toFormat("kkkk-'W'WW") }}` |
| `runId` | the same value as `weekLabel` |

So a run on Monday 2026-09-14 reports on Monday 2026-09-07 00:00 up to, but not including,
Monday 2026-09-14 00:00.

**How it is passed down.** Every node that needs a date reads it by named reference:

```
{{ $('Set - Define Reporting Week').first().json.weekStart }}
```

`docs/knowledge/n8n-gotchas.md` warns that `$('Node').first()` returns that node's *last*
run in the whole execution. That warning applies to nodes inside a loop. Node 2 sits
between the trigger and the fan-out, it is not inside any loop, and there is no loop node
anywhere in this workflow. It therefore runs exactly once per execution, so its first run
and its last run are the same run. The reference is safe here, and the Coder should not
substitute `$json`, because this workflow has branches and the design defaults prefer a
named reference in that case.

**No other node may call `$now`, `DateTime.now()`, or any relative date expression.** QA
should grep the finished workflow for `$now` and expect exactly the three hits inside node 2.

---

## 6. The single-OpenAI-call guarantee

The requirement is that one call per brief is *structurally impossible* to violate, not
merely intended. Four independent locks, each of which alone would be sufficient.

**Lock 1 — the Merge emits one item.**
`Merge - Wait For All Five Sources` runs in `chooseBranch` / `waitForAll` mode with
`output: empty`. The schema labels that option "A Single, Empty Item". However many rows
the five sources return — 5 or 5,000 — the Merge output is one item. Nothing upstream of
node 8 can change the item count downstream of it.

**Lock 2 — the Code node runs once and returns one item.**
`Code - Normalise And Calculate Metrics` is set to **Run Once for All Items**. In that mode
the node body executes once per node execution regardless of input count, and the
downstream item count is whatever the `return` statement produces. It returns a
single-element array. Per `n8n-gotchas.md`, `$json` does not exist in that mode, so the
node must use `$input.all()` and named node references. That is required anyway, because it
needs to read five different sources by name.

**Lock 3 — the IF never multiplies.**
`IF - Enough Data To Build Brief?` passes items through. It can reduce a stream to zero on
one output, never increase it. One item in, at most one item out.

**Lock 4 — no fan-out node exists on the path.**
There is no `Split In Batches`, no `Split Out`, no `Item Lists`, no sub-workflow call and
no `Loop Over Items` anywhere between node 1 and node 11. Per-item iteration in n8n is
automatic, so none is needed, and the design defaults forbid adding one. **The Coder may
not insert any such node between the Merge and the OpenAI node.** QA should treat the
presence of one as an automatic fail.

**Cost cap as a fifth line of defence.**
The OpenAI node is set to `retryOnFail: true, maxTries: 2`. Even a total failure of every
lock plus a hard API error cannot produce more than two charges from one execution.

**Verification, not just design.**
Per `CLAUDE.md`, the node is built disabled and run disabled first. The Tester enables it
only after QA passes, and then reads the node's input item count off the execution and
records it. QA's done-criteria must include "the OpenAI node received exactly 1 input item".

---

## 7. The approval gate — PRD §12 item 13, still OPEN

The gate sits at node 15, after the PDF and before the email, exactly where PRD step 15
puts it.

### The shape does not change either way

Both candidate nodes use the same `sendAndWait` operation, both take the same
`approvalOptions.values.approvalType: double`, both take `options.limitWaitTime`, and both
return the approval result on the single item they were given. Node 16 reads it with the
same expression in both cases. **Only node 15 swaps. Nodes 14, 16, 17 and 21 are identical
under either choice.** So Leo's answer does not reopen the design.

### Option A — Telegram (recommended)

`n8n-nodes-base.telegram`, typeVersion 1.2, `operation: sendAndWait`,
`approvalType: double`, `chatApproval: true`, `options.limitWaitTime` 6 hours.

The schema note on `chatApproval` reads: "approvers respond with one tap on buttons inside
the Telegram chat, instead of opening a link in the browser. Requires this n8n instance to
be reachable over public HTTPS." This instance is on a public HTTPS host, so that condition
is met.

**Why I recommend it:**

1. **No new credential.** Telegram is already needed for step 17 and for every alert in
   this design. Email approval would add a second channel to set up and keep alive.
2. **A corporate mail scanner cannot accidentally approve.** Many mail gateways pre-fetch
   every link in an inbound message to check it for malware. An approval link that is
   fetched is an approval that is granted. That is a real and well-known failure mode for
   email approval buttons. A Telegram button tap is not pre-fetched by anything. This is
   the strongest argument, because a false approval sends unreviewed numbers to leadership,
   which is the exact outcome PRD v1.2 was written to prevent.
3. **Speed.** One tap on a phone at 07:05 on a Monday. No inbox to open.
4. **Nothing is lost.** The PDF still reaches leadership by email at step 17. Only the
   approval request moves to Telegram, and it carries the Google Doc link so the approver
   can read the full brief before tapping.

### Option B — Gmail

`n8n-nodes-base.gmail`, typeVersion 2.2, `operation: sendAndWait`, `approvalType: double`,
`options.limitWaitTime` 6 hours.

**When to choose it:** if the approver is an executive who does not use Telegram, or if
Leo wants the approval trail sitting in a mailbox for audit. If Option B is chosen, the
mail-scanner risk above must be raised with whoever runs the mail gateway, and the
Integrator must check whether the Gmail `sendAndWait` operation can carry the PDF as an
attachment (section 11, item 5). If it cannot, the approval message carries the Doc link,
the same as Option A.

### The question Leo must answer

Who approves? One name. That answer decides the channel, because the channel should follow
the person.

---

## 8. Charts and the Google Doc — PRD §12 items 11 and 12, still OPEN

### A finding the PRD did not anticipate

PRD step 13 asks for a Google Docs brief "containing metrics, narrative, and charts".
Reading the `n8n-nodes-base.googleDocs` schema, the `update` operation's `object` list is:
footer, header, named range, page break, paragraph bullets, positioned object, table,
table column, table row, text. `positionedObject` supports `delete` only. **There is no way
to insert an image with the Google Docs node.** So the node cannot produce the document the
PRD describes.

### Recommended route — Route A

**Build the whole brief as HTML, then have Google Drive convert the HTML into a Google Doc.**

- Node 12 builds one HTML string containing headings, the KPI table, the narrative, and two
  `<img>` tags whose `src` is a QuickChart URL:
  `https://quickchart.io/chart?c={{ encodeURIComponent(JSON.stringify(config)) }}`
- Node 13 POSTs that HTML to the Drive upload endpoint with
  `mimeType: application/vnd.google-apps.document`. Drive converts it, fetching the two
  remote images as it goes, and returns a real Google Doc.
- Node 14 downloads that Doc as PDF using the Drive node's own
  `googleFileConversion.conversion.docsToFormat: application/pdf` option, which was read
  from the schema and definitely exists.

**Why this route.** It is one node for the whole document. The layout is expressed as HTML,
which handles tables and headings properly. It avoids the alternative's index arithmetic,
described below.

**What makes it risky.** It depends on Drive's HTML-to-Doc conversion fetching remote image
URLs. That is documented Google behaviour but it is not proven on this setup. **This is the
single riskiest assumption in the design.** Integrator item 3.

### Fallback route — Route B

If Route A does not embed the images, fall back to three nodes:

1. `Google Docs - Create Executive Brief` (`n8n-nodes-base.googleDocs` v2, `document:create`)
2. `Google Docs - Insert Brief Text` (`document:update`, `object: text`, `action: insert`)
3. `HTTP Request - Insert Chart Images` — a call to
   `https://docs.googleapis.com/v1/documents/{id}:batchUpdate` with two
   `insertInlineImage` requests, each carrying a QuickChart URL and a character index.

**Why this is the fallback and not the recommendation.** `insertInlineImage` needs an exact
character index in the document body. The brief's text length changes every week, because
the narrative length changes. Getting that index right week after week is the classic
failure mode for this pattern, and it fails silently by putting a chart in the middle of a
sentence. Route A has no indexes at all.

### If the QuickChart URL is too long

A QuickChart GET URL carries the whole chart config, URL-encoded. A long revenue trend
could push past the practical URL length limit. If the Integrator finds that happening, the
fix is to POST each config to `https://quickchart.io/chart/create` first, which returns a
short permanent URL, then use that. That adds two `HTTP Request` nodes and is only taken if
needed. **The `n8n-nodes-base.quickChart` node exists on this instance and was verified,
but it returns binary PNG only, not a URL, so it cannot be used to feed a Doc. It stays
available as a route if charts ever need to be posted to Telegram as images.**

Both routes are marked **Integrator must verify** before the Coder builds.

---

## 9. Decisions

### Sub-workflows

**None in v1.** Nothing in this design is used twice, so extraction would buy nothing and
would cost the visibility that makes the single-call guarantee easy to audit — a
sub-workflow boundary is exactly where item counts get hidden. The strongest candidate for
a future extraction is nodes 12 to 14, "build the document and export the PDF". Revisit
that only if a second brief for a second audience is ever added, which PRD §11 currently
puts out of scope.

### Retry strategy

| # | Node | retryOnFail | maxTries | waitBetweenTries | onError |
|---|---|---|---|---|---|
| 3 | HubSpot | true | 3 | 5000 | `continueRegularOutput` |
| 4 | Supabase | true | 3 | 5000 | `continueRegularOutput` |
| 5 | Sheets Finance | true | 3 | 5000 | `continueRegularOutput` |
| 6 | Sheets Support | true | 3 | 5000 | `continueRegularOutput` |
| 7 | Sheets Snapshot | true | 3 | 5000 | `continueRegularOutput` |
| 11 | **OpenAI** | true | **2** | 10000 | `continueRegularOutput` |
| 13 | HTTP Create Doc | true | 3 | 5000 | `stopWorkflow` |
| 14 | Drive PDF | true | 3 | 5000 | `stopWorkflow` |
| 15 | **Approval** | **false** | 1 | — | `stopWorkflow` |
| 17 | Gmail | true | 3 | 5000 | `stopWorkflow` |
| 18 | Telegram summary | true | 2 | 3000 | `continueRegularOutput` |
| 20 | Sheets append | true | 3 | 5000 | `stopWorkflow` |
| 21, 22, 25 | Alerts | true | 2 | 3000 | `continueRegularOutput` |

Three of these need their reason stated.

- **Node 11, maxTries 2.** Every retry on this node is a real charge. Two tries survives a
  single 429 or 502 and caps the worst case at two charges per execution.
- **Node 15, retry off.** A retry would send a second approval request to the same person
  for the same brief. Two pending approvals for one run is a correctness bug, not a
  reliability improvement.
- **Node 18, continue on error.** By the time this runs, the PDF has already reached
  leadership. A Telegram hiccup must not stop the snapshot row being written, because a
  missing snapshot row breaks next week's comparison, and that is a worse outcome than a
  missing Telegram post.

All five fetches use `continueRegularOutput` plus `alwaysOutputData: true`. That
combination is what makes the missing-source handling in section 10 work, and it is
explained there.

### Timeouts — a rule this design cannot fully meet

`CLAUDE.md` design defaults say "Every fetch gets a timeout and a retry." Every fetch gets a
retry. **Only node 13 can be given a timeout.** The `HTTP Request` node exposes
`options.timeout`; the HubSpot, Supabase and Google Sheets nodes do not expose any timeout
parameter at all. There is no way to set one on them.

Three honest options, for Leo and the Orchestrator to pick from:

1. **Accept it.** Rely on retries. A hung source would hang the run until n8n's own
   execution handling ends it. Simplest, and the risk is low for these four providers.
2. **Replace the slow ones with `HTTP Request` nodes**, which do expose
   `options.timeout`. Costs the convenience of the service nodes and their credentials
   handling, and adds authentication work.
3. **Set a workflow-level execution timeout.** Risky here, because node 15 puts the
   execution into a waiting state for up to six hours, and it is not proven that a
   workflow timeout leaves a waiting execution alone. Integrator item 13.

**My recommendation is option 1 for v1**, with the gap recorded here so it is a known
accepted risk rather than a missed rule. If a source ever does hang in production, move
that one node to option 2.

### Idempotency

**Dedupe key: `weekLabel`**, in ISO week form, for example `2026-W37`. It is computed once
in node 2 and it is the first column of the snapshot sheet.

The re-run protection works like this. Node 7 already reads the whole snapshot sheet as
part of the normal flow. Node 9 checks whether any existing row carries the current
`weekLabel`. If one does, it sets `alreadyAppended: true` and folds that into
`readyToBuild: false` with `blockReason: "A snapshot row for 2026-W37 already exists."`
Node 10 then routes the run to node 22 and it ends — before the paid call, before the
email, and before any second row.

So a Monday re-run, whether by hand or by n8n retrying, cannot write a duplicate row and
cannot send leadership a second copy of the same brief. A deliberate re-send is done by
hand from the Doc that is already in Drive.

**Open question for Leo:** is stopping a re-run outright the behaviour you want, or should
a re-run be allowed to rebuild and re-send while still refusing to write a second row?
Section 14, question 4.

### Database usage

There is no database. The **weekly snapshot Google Sheet** is the only store.

- **Key:** `weekLabel`, first column.
- **Stored:** one row per week, holding each KPI's value and target, the pipeline movement
  figures, the revenue figure, a `generated_at` timestamp, and a `synthetic` flag.
- **The column order is a contract.** Once rows exist, columns may not be reordered or
  renamed without a migration. That is already written into
  `docs/knowledge/data-sources.md`. Node 20 must use `mappingMode: defineBelow` with every
  column named explicitly — never auto-mapping, which would silently reorder.
- **The `synthetic` column is a new requirement this design surfaces.** PRD §5 asks for
  about 12 weeks of seeded pipeline history. `CLAUDE.md` says every synthetic row must be
  marked in the sheet itself. Those two together mean the sheet needs a `synthetic`
  TRUE/FALSE column from day one, and the revenue trend chart must be able to show which
  points are seeded. The PRD does not mention this column. It has to be settled before the
  Coder builds, because it changes the contract.

### Human-review points

Exactly one: node 15. Nothing reaches leadership without it. There is no second gate and no
auto-approve path of any kind.

### Cost exposure

| Node | Costs money | Roughly |
|---|---|---|
| 11 `OpenAI - Write Executive Narrative` | **Yes** | one text call per run, a few thousand input tokens plus a short answer. Pennies. Capped at 2 calls by `maxTries: 2`. |
| 3, 4, 5, 6, 7 source reads | No, within existing plans | confirm per source, per `money-facts.md` |
| QuickChart URLs | No | free public endpoint, rate limited |
| 13, 14 Google Docs and Drive | No | |
| 15, 17, 18, 21, 22, 25 Telegram and Gmail | No | |
| 20 Sheets append | No | |

The exact model id and its price per million tokens are not settled. The Integrator
confirms both against live OpenAI docs, and the first real call gets logged with its cost
in `docs/knowledge/money-facts.md`.

---

## 10. Missing-source handling, end to end

PRD §7: if an individual source fails, flag the missing dataset rather than silently
generating a misleading result. `CLAUDE.md`: never let one failed source produce a
complete-looking brief; name the missing dataset on the brief itself.

**Step 1 — a failed fetch still produces an item.**
Each of nodes 3 to 7 is set to `onError: continueRegularOutput` with
`alwaysOutputData: true`. So each branch always delivers something to the Merge, whether it
succeeded, returned nothing, or threw. The Merge therefore never stalls waiting on a dead
branch. This is why `continueErrorOutput` was not used: an error output would leave that
Merge input empty, and whether a five-input Merge still fires with an empty input is not
something this design will guess at.

**Step 2 — three outcomes are told apart, not two.**
Node 9 classifies each source as one of:

| Outcome | How it is recognised | What it means |
|---|---|---|
| **OK** | rows returned with the expected fields | normal |
| **Empty week** | one item, no `error` key, no expected fields | the source worked, there was genuinely nothing this week. A real zero. |
| **Failed** | the item carries an `error` key and no source fields | the source could not be read. Not a zero. |

That distinction matters. A support log with no tickets is a good week. A support log that
could not be read is an unknown. They must never print the same.

**Step 3 — the warning is built once.**
Node 9 fills `missingSources` with plain English names, sets `dataComplete: false`, and
writes one `dataWarning` sentence:
`Data warning: Supabase product usage could not be read this week. Figures from that source are missing, not zero.`

**Step 4 — the model is told, and told to shut up about it.**
The `dataWarning` and the `missingSources` list go into the OpenAI prompt as an explicit
instruction: say nothing about the missing area, do not estimate it, do not infer it from
the other sources. This backs up the `CLAUDE.md` rule that the narrative never invents a
cause or a number.

**Step 5 — the warning appears on the face of everything.**
The `dataWarning` is the first line of the brief HTML, above the executive summary. It is
also the first line of the approval message at node 15, and the first line of the Telegram
summary at node 18. The approver sees the flaw before they approve.

**Step 6 — a metric with a dead source prints as a dash.**
Any metric whose `available` is false renders as an em dash with a footnote pointing at the
warning line. Never `0`. Never blank. Never a carried-over figure from last week.

**Step 7 — no snapshot row on a partial run.**
Node 19 tests `dataComplete === true`. On a partial run, the brief is still built, still
approved, still emailed, still summarised — and then the run ends at node 23 with no row
written. A snapshot row built from four sources out of five would poison the week-over-week
comparison for every following week.

**Step 8 — if everything is down, nothing is spent.**
If all five sources failed, or the metric list came back empty, node 9 sets
`readyToBuild: false` and node 10 routes to node 22. The run stops before the paid call.

---

## 11. What the Integrator must verify before the Coder builds

1. **OpenAI.** The exact model id for `resource: text`, `operation: response` on node
   version 2.3, and the request body the node builds. Confirm credential
   `DBmj9DTgWeT0tOGw` (type `openAiApi`) is accepted by that node version. Confirm the
   price per million input and output tokens.
2. **QuickChart.** A working Chart.js config for the pipeline bar chart and the revenue
   trend line chart. The practical maximum URL length. Whether
   `quickchart.io/chart/create` short URLs are needed for the trend chart.
3. **Google Drive HTML conversion — the riskiest item.** That a multipart upload of
   `text/html` with `mimeType: application/vnd.google-apps.document` produces a Google Doc,
   and that remote `<img src="https://quickchart.io/...">` URLs are fetched and embedded
   during that conversion. If not, Route B in section 8 is the fallback and the node list
   changes by two nodes.
4. **PDF export.** That `n8n-nodes-base.googleDrive` v3, `file:download` with
   `options.googleFileConversion.conversion.docsToFormat: application/pdf` works on a Doc
   created seconds earlier, and what the output binary field is called.
5. **Gmail `sendAndWait`.** Whether it supports a binary attachment. Exactly what it
   outputs on approve, on disapprove, and on `limitWaitTime` expiry — the field path and
   the values.
6. **Telegram `sendAndWait` with `chatApproval: true`.** The same three outputs. And that
   Telegram can reach this instance's public HTTPS webhook.
7. **HubSpot.** Which operation on node version 2.2 lists deals for a date window —
   `deal:search` with a filter, or `deal:getAll` — which properties come back, the page
   size, and the rate limit.
8. **Supabase.** Which operation on node version 1 returns rows filtered by a date window,
   and its row limit.
9. **Google Sheets v4.7.** The `append` operation with `columns.mappingMode: defineBelow`,
   and what the read operation returns on an empty sheet.
10. **`onError: continueRegularOutput` item shape** on this n8n version. The exact keys in
    the item a failed node emits. Node 9's failure detection depends on it.
11. **Merge v3.2** in `chooseBranch` / `waitForAll` / `output: empty` mode with five
    inputs: confirm it emits exactly one item, and confirm it fires when one input branch
    returned an error item.
12. **Telegram message limit** for node 18 and the alert nodes, so the truncation
    expression uses the right number.
13. **Workflow execution timeout versus a waiting execution.** Whether an execution paused
    at a `sendAndWait` node is killed by a workflow-level timeout. This decides option 3 in
    the timeouts discussion in section 9.

---

## 12. Assumptions — every one of these is a guess until confirmed

| # | Assumption | Why it matters | PRD link |
|---|---|---|---|
| A1 | Time zone is `Europe/London` | changes the trigger time and every week boundary | §12 item 9, OPEN |
| A2 | The week is the seven days ending the Sunday before the run. `weekStart` = the previous Monday 00:00, `weekEnd` = the current Monday 00:00, exclusive | changes every number in the brief | §12 item 9, OPEN |
| A3 | `weekLabel` in ISO `YYYY-Www` form is the dedupe key and the first column of the snapshot sheet | changes the re-run protection | new |
| A4 | The snapshot sheet has a header row and a fixed column order | node 20 maps to it explicitly | §12 item 2, OPEN |
| A5 | Each metric's target lives in the finance actuals sheet, in a `target` column | changes how variance is computed | §12 item 6, OPEN |
| A6 | One person approves, not a quorum | a quorum would need a different node shape | §12 item 13, OPEN |
| A7 | The approval wait limit is 6 hours, 07:00 to 13:00 | decides when a run is treated as expired | new |
| A8 | Charts are PNG, about 600 by 350, from a Chart.js v2 config | affects the QuickChart URL length | §12 item 11, OPEN |
| A9 | The Doc and the PDF live in one named Drive folder | the folder id is not known | new |
| A10 | Gmail is the email sender at step 17 | PRD says "email" without naming a provider | §12 item 7, OPEN |
| A11 | Telegram is available for alerts whichever channel approval uses | nodes 18, 21, 22, 25 all assume it | §12 item 8, OPEN |
| A12 | The narrative is one plain text block of about 250 words, built only from the metrics JSON | changes the prompt and the HTML | new |
| A13 | The snapshot sheet gains a `synthetic` TRUE/FALSE column before any seeding | required by `CLAUDE.md`, not in the PRD | new |

---

## 13. Where this design and PRD v1.2 disagree

Five points. Each needs Leo or the Orchestrator to settle it, and the PRD should be updated
rather than the design quietly diverging.

1. **Node count.** PRD §8 says about 20 nodes. This design is 25: 20 on the happy path, 5
   on the error path. The extra 5 are demanded by PRD §7 and by `CLAUDE.md`'s rule that the
   error path is built at the same time as the happy path. The estimate did not price them.

2. **"Every fetch gets a timeout."** `CLAUDE.md` design defaults require it. The HubSpot,
   Supabase and Google Sheets nodes expose no timeout parameter. The rule cannot be met
   with those nodes. Section 9 sets out three options and recommends accepting the gap for
   v1.

3. **Charts in the Google Doc.** PRD step 13 asks for a Google Docs brief containing
   charts. The n8n Google Docs node has no image-insert capability — proven from its
   schema. A different route is required, and both candidate routes need verification.

4. **The `synthetic` column.** PRD §5 wants about 12 weeks of seeded history. `CLAUDE.md`
   requires every synthetic row to be marked in the sheet. Together they require a
   `synthetic` column that the PRD does not mention, and that column is part of the sheet's
   contract, so it has to exist before the first real row is written.

5. **The snapshot row on a partial run.** PRD §12 item 14 asks what happens to the snapshot
   row after a rejection. It does not ask what happens when a *source* is missing. This
   design blocks the append in both cases. That is a new decision, taken from the
   `CLAUDE.md` rule about not poisoning next week's comparison, and Leo should confirm it.

---

## 14. Open questions Leo must answer before the Coder builds

These are on top of everything already listed as OPEN in PRD §12.

1. **Who approves the brief?** One name. The channel should follow the person, so this one
   answer settles PRD §12 item 13.
2. **How long should the approval wait before it expires?** The design assumes 6 hours.
3. **On a rejection or a timeout, is no snapshot row the right answer?** The recommendation
   in section 15 is yes, with the reasoning.
4. **If the workflow runs twice on the same Monday, should the second run stop completely,
   or rebuild and re-send while refusing to write a second row?** The design currently
   stops it completely.
5. **Does the snapshot sheet get a `synthetic` column?** It has to, if the 12 weeks of
   seeded history in PRD §5 are going in.

---

## 15. The reject path and the timeout path — PRD §12 item 14, still OPEN

**This section is a recommendation. It is not decided. Leo approves or changes it.**

Three outcomes come out of node 15.

### Outcome 1 — Approved

| Question | Answer |
|---|---|
| Does the email send? | Yes. Node 17, PDF attached. |
| Does Telegram post? | Yes. Node 18, the short summary. |
| Is the snapshot row appended? | Yes, **but only if all five sources were read**. Node 19 checks. |

### Outcome 2 — Rejected

| Question | Recommended answer |
|---|---|
| Does the email send? | **No.** The whole point of the gate. |
| Does Telegram post the summary? | **No.** A summary is a publication. |
| Does Telegram post anything? | **Yes**, node 21 posts an alert: which week, that nothing was sent, that no row was written, and the Doc link. |
| Is the snapshot row appended? | **No.** |

**Why no row.** The snapshot sheet is the record of what leadership was actually told. If
nothing was sent, there is nothing to record, and a row would silently claim a brief went
out. More importantly, the usual reason to reject a brief is that a number looks wrong.
Writing a suspect number into the sheet is how it gets picked up as "last week" and reused
seven days later. `CLAUDE.md` is explicit: never append a snapshot row for a run that
failed. An unapproved run is a failed run.

**The alternative, if Leo prefers trend continuity.** Add a `status` column to the snapshot
sheet, append the row with `status: rejected`, and have the revenue trend chart filter to
`status = sent`. That keeps the history complete and keeps the bad number out of the chart.
It costs a new column in the sheet contract. **I do not recommend it**, for the reason
above: a rejected number is usually a wrong number, and a wrong number in the sheet finds
its way out eventually. But the option is real and it is Leo's call.

### Outcome 3 — Nobody answers in time

Treated exactly like a rejection, with different wording.

| Question | Recommended answer |
|---|---|
| Does the email send? | **No.** Silence is not consent. |
| Does Telegram post the summary? | **No.** |
| Does Telegram post anything? | **Yes**, node 21, louder wording: "Brief for 2026-W37 expired unapproved after 6 hours. Nothing was sent." Plus the Doc link. |
| Is the snapshot row appended? | **No.** |
| Is anything lost? | **No.** The Doc and the PDF stay in Drive. A person can read them and send by hand. |

**Why silence must not send.** The opposite rule — send if nobody objects — turns the gate
into decoration and quietly restores exactly the risk PRD v1.2 was written to remove.

### Telling a rejection apart from a timeout

Both may arrive as the same "not approved" result. That is an Integrator item (section 11,
item 5 and 6). **The design is built so that it does not matter:** both outcomes do the same
three things — no email, no summary, no row. Only the alert wording differs, and the wording
is chosen by an expression that compares the time now against the approval request time
held on the item. No extra node, and no correctness depends on the distinction.

That is deliberate. Where an ambiguity cannot be removed, the safest design is one where
the ambiguity cannot cause harm.
