# n8n traps and conventions

Read this before you write a Code node, any cross-node expression, or any node name.

Copied word for word from the Advance Social Media Generation project on 2026-09-13.
Same n8n instance, so the same traps apply. Nothing here was rewritten.

The `PRD 1.xx` references point at that project's PRD, not this one. They are kept so
the proof can still be found.

## Traps found by running things

Each of these cost time or money to find. None are guessable from documentation.

- **`$helpers` is undefined on this instance.** Code nodes must use `this.helpers`.
- **`$json` does not exist in a Code node set to "Run Once for All Items".** Use
  `$input.first().json` or `$input.all()`. This bit an *error handler*, which would have
  thrown while handling an error. PRD 1.79.
- **`$('Node').first()` returns that node's LAST run in the whole execution, not the
  current item's run.** So any per-card counter read back through a node reference
  inherits the previous card's value. **Per-item state must be seeded once and passed
  hand to hand on the item.** `$runIndex` has the same flaw — it counts a node's runs
  across the execution and never resets per item. This produced four separate blocking
  bugs in one workflow, and a first attempt to fix it swapped `$runIndex` for a node
  reference, which is the same disease. Use `.item` (paired-item) if you must reach
  across, never `.first()`. PRD 1.79.
- **A zero-valued optional property is not the same as an absent one.** Passing
  `'fade-in': 0` to JSON2Video produced a full second of pure black. **Omit what you do not
  want.** PRD 1.61.

## Why both halves of a node name are required

When renaming a node, **never remove the original node name.** Keep it, add a dash, then
write what the node actually does.

```
Append Row - Add to DB
IF - Is Duplicate?
HTTP Request - Fetch Lead Enrichment
Code - Normalise Phone Numbers
Gmail - Send Approval Request
```

The original half keeps the node type obvious at a glance, so anyone reading the canvas
knows what kind of node it is without clicking it. The action half says why it is there.
Both halves are required. A node called just `Is Duplicate?` has thrown away information
that cannot be recovered from the canvas.

This applies to every node in every workflow in this project, and the Coder agent enforces
it while building.

## Traps found by reading node schemas — 2026-09-13

Found by the Integrator while verifying `docs/architecture/v1-architecture.md`. Each one was
read from the node schema this instance serves, not from memory. Full detail in
`docs/reports/integrator-v1.md`.

### Gmail `sendAndWait` cannot attach a file

The Gmail node does have an attachments field. It appears only under `operation: send` and
`operation: reply`. It is **absent** from `operation: sendAndWait`. So an approval email
cannot carry a PDF. It can carry a link, and nothing else.

Proven from the schema at two separate places, both scoped to send and reply. This is not a
limitation to work around. It is the only behaviour the node has.

### The Google Docs node cannot insert an image

`nodes-base.googleDocs` offers three operations on a document: create, get and update. None
of them insert an image. A workflow that needs a chart inside a Doc has to build the document
some other way — uploading HTML to Google Drive and letting Drive convert it is one route.

Confirmed twice on 2026-09-13, once by the Designer and once by hand in the main session.

### The OpenAI node has no built-in model list

`@n8n/n8n-nodes-langchain.openAi` holds `modelId` as a resource locator. The From List option
calls the OpenAI API through the credential, so the list shows whatever that account can
actually reach. There is no hardcoded list.

Two consequences. A model id cannot be verified without the live credential. And a model id
typed from memory will not fail validation — it will fail at run time, after the call is billed.
Always pick it from the live list.

### A workflow-level execution timeout fights a waiting execution

n8n issue 15123 reports that a Wait node waiting longer than the workflow timeout can cause an
infinite loop. Any workflow that pauses for a human approval should have **no** workflow-level
execution timeout set at all.

### HubSpot deal properties: `additionalFields` is Search, `filters` is Get Many

The HubSpot node has two different places to name deal properties, and picking the wrong one
means every deal comes back as a bare id with no `amount` and no `dealstage`.

- **`deal:search`** uses the **`additionalFields`** collection. Deal properties go at
  `additionalFields.properties`.
- **`deal:getAll`** uses the **`filters`** collection. Deal properties go at `filters.properties`,
  and that is the one carrying the warning "By default, the results will only include Deal ID".

**How this was settled.** `sortBy` exists at exactly one path in the whole 87-property schema:
`additionalFields.sortBy`. In HubSpot's API `sortBy`, `direction` and `query` are search
parameters and have no meaning on a list call. So `additionalFields` is the search collection.

A QA audit on 2026-09-13 called this the other way round and raised it as a defect. The Coder
refused the fix and was right. Checked a third time in the main session with two `get_node`
searches. **Still not proven against a live HubSpot account — there is no HubSpot credential on
this instance.** The first real run must confirm deals arrive with their values attached.

---

## Measured on this instance — execution 536, 2026-09-13

The first execution the Executive Reporting Rollup ever had. Workflow `iIypy4KWtaqsowAQ`,
run manually from the Schedule Trigger. `status: success`, 11 nodes executed, 19.1 seconds.
Every shape below is copied from the real execution data, not predicted.

These four shapes were recorded as UNPROVEN in the Integrator report (items 9 and 10) and in
QA v1. They are now proven.

### `alwaysOutputData: true` really does emit one blank item

A source that returns nothing emits exactly this, not zero items:

```json
{ "json": {}, "pairedItem": { "item": 0 } }
```

Measured on `Google Sheets - Fetch Finance Actuals`, `- Fetch Support Log` and
`- Fetch Last Week Snapshot`, all three reading a real sheet that has headers and no data rows
for the requested week.

**So any Code node that tests `items.length === 0` to detect an empty source is dead code once
`alwaysOutputData` is on.** That is QA v2 finding NEW-1 and it is confirmed.

### A node with no credentials does NOT emit a consistent shape

This is the trap, and it is worse than the one above.

Two nodes, both with **no credential attached**, both with `onError: continueRegularOutput` and
`alwaysOutputData: true`, produced **different output**:

| Node | Type | Output | Time |
|---|---|---|---|
| `HubSpot - Fetch Deals And Pipeline` | `nodes-base.hubspot` | `{ "json": { "error": "Node does not have any credentials set" } }` | 10,022 ms |
| `Supabase - Fetch Product Usage` | `nodes-base.supabase` | `{ "json": {} }` | 11 ms |

**HubSpot names its failure. Supabase does not.** Supabase failed for exactly the same reason and
emitted a blank item that is byte-identical to a healthy-but-empty Google Sheet.

**Consequence, measured in the same run.** Node 9 classifies by looking for an `error` key. It got
HubSpot right and Supabase wrong:

```json
"hubspot":  { "ok": false, "rowCount": 0, "note": "Node does not have any credentials set" },
"supabase": { "ok": true,  "rowCount": 1, "note": "" }
```

`missingSources` listed HubSpot only. **A source with no credential at all was reported as healthy
with one row of data.** `CLAUDE.md` says never let one failed source produce a complete-looking
brief, and this is the mechanism by which that would happen.

**Do not detect source failure by looking for an `error` key alone.** It is not reliable across
node types. The 11 ms execution time is the other tell — a real network call cannot be that fast.

### Telegram: `Bad Request: chat not found`

`Telegram - Alert Run Blocked` was fully configured — real credential, real numeric chat id — and
still failed:

```json
{ "json": { "error": "Bad Request: chat not found" } }
```

**A Telegram bot cannot open a conversation.** The person must send the bot a message first, from
their own Telegram account. A correct chat id is not enough on its own. Getting the id from
`@userinfobot` does not count — that is a different bot.

**The node has `onError: continueRegularOutput`, so the failure was swallowed and the execution
finished `status: success` with nobody told.** An alert node that cannot reach anyone, on a
workflow whose whole job is to tell you things, is the exact silent failure QA called BLOCKER 4.

### What worked, and is now proven rather than assumed

- **The five-input Merge did not stall** when two of its inputs carried failure or blank items.
  It passed through and node 9 ran.
- **Node 9 emitted exactly one item** — `itemsOutput: 1`. Acceptance criterion 10 passes on real
  data. The one-paid-call guarantee is no longer only structural.
- **The blank snapshot item produced exactly the junk trend point QA predicted:**
  `{ "weekLabel": "", "revenue": null, "synthetic": false }`. Node 12 filters it on
  `Number.isFinite(p.revenue)`, so it is harmless today.
- **The run stopped where it should.** `readyToBuild: false`, blockReason
  `"No metrics were computed: Finance Actuals had no rows for 2026-W36."` No Doc, no PDF, no
  email, no snapshot row, and node 11 was never reached.

---

## `n8n_update_partial_workflow` silently deletes parameters you did not mention

**Found the hard way, three times, on 2026-09-13 and 2026-09-14. It cost three extra build rounds
and nearly shipped a broken paid node.**

An `updateNode` operation has two shapes and they behave completely differently:

| Shape | Effect |
|---|---|
| `updates: { "parameters": { … } }` | **REPLACES the whole `parameters` object.** Every key you did not list is deleted. |
| `updates: { "parameters.foo": value }` | **MERGES.** Only `foo` changes. Everything else survives. |

**Always use dot-notation paths, one per leaf value, array indexes included**, for example
`"parameters.approvalOptions.values.approveLabel"`. Never hand it a whole `parameters` object
unless you genuinely intend to wipe every key on that node.

`n8n_validate_workflow` does **not** catch this. Every one of these losses validated clean with
`valid: true` and 0 errors, because the deleted keys all had n8n defaults to fall back on.

### What it actually deleted here

**Round 1 — nodes 9 and 12.** Lost `mode: "runOnceForAllItems"` and `language: "javaScript"`.
The one-paid-call guarantee silently stopped being an explicit setting and became a default.

**Round 2 — node 15 `Telegram - Request Brief Approval`.** Lost `resource`, `responseType`,
`postDecisionBehavior`, and both button labels `approveLabel: "✅ Approve"` and
`disapproveLabel: "❌ Decline"` — the actual buttons the human taps to approve the brief.

**Round 2 — node 11 `OpenAI - Write Executive Narrative`.** Lost `resource`, `operation`,
`simplify: true`, the `type: "text"` on both messages, and `role: "user"` on the user message.

**`simplify: true` was the dangerous one.** Node 12 reads the narrative as
`ai.content || ai.text || ai.output_text || ai.message.content` — field names that only exist in
the *simplified* output shape. Without `simplify`, node 11 would return the raw API response,
node 12 would find none of those fields, and the brief would print
*"Narrative unavailable this week"* — **after the paid call had already been billed.** Losing
`role: "user"` would also have sent the model two system messages and no user turn.

### How to not be bitten

1. **Dot-notation paths only.**
2. **After every write, read the node back and compare it key by key against what it had before**,
   not just against the values you set. A diff against the previous state is the only thing that
   catches a deletion.
3. **Capture the node's full `parameters` before you edit it**, so you have something to compare to.
4. Remember that a clean validation proves nothing here.

### A related trap on the same tool

`n8n_get_workflow` with `mode: "structure"` reports `connectionCount: 20` even after five
connections were added. That field counts **distinct source-node keys**, not edges — the five new
error connections were added under node keys that already existed, so it cannot move.
**The real edge count is `validConnections` from `n8n_validate_workflow`**, which went 26 → 31.

---

## Proven on execution 538, 2026-09-14 — error outputs and the Merge

### `chooseBranch` / `waitForAll` waits on INPUTS, not on connections

A Merge input can have more than one connection feeding it, and the Merge still fires when only
one of them delivers. Proven here: all five inputs of a five-input Merge were each fed by **two**
connections — a fetch node's main output and its error output — and on any run only one of the two
carries data. **The Merge fired in 2 ms and emitted one item.**

This makes "wire both outputs of a node into the same Merge input" a safe pattern for turning a
node's success-or-failure into a guaranteed single item at a known index.

### `$('<node>').all(1, 0)` reads the error output, and it does not throw

Branch index 1 is the error output when a node has `onError: continueErrorOutput`. Measured on all
five fetch nodes: `errorBranchThrew` was `false` every time. Reading a branch that has no items
returns an empty array rather than throwing.

### But a node with NO credential does not reliably route to its error output

**This is the trap.** Two nodes, both with no credential at all, both `onError: continueErrorOutput`
and `alwaysOutputData: true`:

| Node | Error output | Main output | Classified as |
|---|---|---|---|
| `nodes-base.hubspot` | **1 item**, "Node does not have any credentials set" | 0 items | failed — correct |
| `nodes-base.supabase` | **0 items** | 1 blank item | empty — **wrong, it is dead** |

Supabase reports success and emits nothing. `alwaysOutputData` then pads the main output with a
blank item that is **byte-identical** to what a healthy Google Sheet with no rows produces.

**So `onError: continueErrorOutput` is not a universal failure detector.** It is only as good as
the node's own willingness to raise an error. Check each node type on real data before relying on
its error branch, and never assume two node types behave the same in the same failure.

Note also: when a node DOES route to its error output, `alwaysOutputData: true` does **not** add a
blank item to the main output. HubSpot showed `mainItemCount: 0`.

### CORRECTION to the section above — a node with NO credential is a special case

The table above, measured on execution 536 and 538, showed `nodes-base.supabase` failing to route
to its error output while `nodes-base.hubspot` routed correctly. **That conclusion was too broad
and is now corrected by execution 539, 2026-09-14.**

**With a credential attached, the Supabase node routes its failure to the error output exactly as
documented.** A deliberately wrong table name produced:

```json
// error output, branch 1
{ "json": { "error": "Could not find the table 'public.SET_ME_supabase_table_name' in the schema cache", ... } }
```

The only change between the two runs was attaching a credential. No code, no connection, no
parameter.

**The real rule, corrected:**

- **A node with NO credential at all does not behave like a node that failed.** Some node types
  (Supabase) return success with nothing rather than raising an error. Others (HubSpot) raise
  `"Node does not have any credentials set"` properly. **Do not test error handling against a node
  that has no credential — you are measuring an artificial state, not a real failure.**
- **Once a credential exists, `onError: continueErrorOutput` is reliable**, on both node types
  tested here.

**The wider lesson, which cost three rounds of design work:** a design decision was nearly taken —
the Designer's Plan B, five marker Set nodes read with `isExecuted` — to work around a fault that
only existed because two nodes had no credentials, which was itself a temporary state the user had
chosen. **Attach the credential and run it before redesigning around its absence.**

### `alwaysOutputData` still pads the main output when the error branch fires

On execution 539 the failing Supabase node emitted items on **both** outputs — one blank item on
main from `alwaysOutputData`, and the real error item on the error branch. So a classifier must
check the **error branch first**, before it counts main-branch items, or a genuine failure reads as
an empty week. Node 9 does this correctly.

This contradicts what execution 538 suggested, where a failing HubSpot showed `mainItemCount: 0`.
**Both shapes occur. Do not rely on the main branch being empty when a node errors.**
