# Coder Report — v1

**Date:** 2026-09-13
**Built against:** `docs/architecture/v1-architecture.md` (v1), informed by `docs/reports/integrator-v1.md`
**Workflow name:** `Executive Reporting Rollup - Weekly Leadership Brief`
**Workflow id:** `iIypy4KWtaqsowAQ`
**Instance:** self-hosted, `https://<your-n8n-instance>`
**Status:** built, validated, connections verified by hand. **Not activated. Not run.**

---

## 1. The sheet-id swap mid-build

My task instructions gave me three Google Sheet ids. Partway through research (after I had
already read the HubSpot/Supabase/Telegram/Gmail schemas but before I created anything on the
instance) the Orchestrator sent a correction: those three ids were dead — created under the
wrong Google account and trashed — and gave three replacement ids under Leo's own account.

**Nothing had been created yet when the correction arrived**, so there was no fix-up to do.
I verified all three replacement ids live, via a read-only `n8n_explore_node_resources` lookup
against the real "Google Sheets account" credential, before using any of them:

| Sheet | File id (used) | Tab gid (used) |
|---|---|---|
| Weekly Snapshot | `1TdTEZ1Jh-4Ne62LmwFfza25pYSttembIuji1o--VElA` | `1425370562` |
| Finance Actuals | `1MJg-FLrykkAAtSDLdCLCuChgVDxI7db9K0pJAetQyhE` | `2085318769` |
| Support Log | `13w0EhJU5taXmmtJ0CUXIqN8P8nfQC489oyXaTsf0MHk` | `1845818240` |

The live lookup also surfaced something worth flagging: each tab is actually named **"Untitled"**,
not "Sheet1". I used the numeric **gid** in every `sheetName` resource locator instead of a tab
name, per the Orchestrator's explicit instruction — a future tab rename cannot silently break
this workflow. `docs/knowledge/data-sources.md` already reflects these ids and this reasoning
(it was updated in parallel with my build); I did not need to touch it further.

---

## 2. What I built — all 25 nodes

Every node uses the name the architecture specifies, verbatim. Every node type and parameter
was read live with `get_node` before I configured it — several schemas (HubSpot, Google Sheets,
OpenAI, Telegram, Gmail, Google Drive, HTTP Request) were too large for a single call and were
paged through in chunks rather than guessed at.

1. **Schedule Trigger - Weekly Monday 7am** — weekly interval, Monday, 07:00, `misfirePolicy: coalesce`. Time zone lives on the workflow (`Europe/London`), not this node.
2. **Set - Define Reporting Week** — see §3, the one deviation that needs sign-off.
3. **HubSpot - Fetch Deals And Pipeline** — `deal:search`, `returnAll: true`, `additionalFields.properties` filled in explicitly (avoids the bare-ID trap). No credential attached. See §4 for the placeholder filter.
4. **Supabase - Fetch Product Usage** — `row:getAll`, `returnAll: true` (avoids the limit-50 trap). No credential attached. `tableId` and the date column are `SET_ME_` placeholders — Supabase is entirely unspecified (PRD §12 item 3, parked).
5–7. **Google Sheets — Fetch Finance Actuals / Support Log / Last Week Snapshot** — `read`, real ids and gids above, ranges A:F / A:H / A:F matching each sheet's real column count. Credential attached.
8. **Merge - Wait For All Five Sources** — `chooseBranch` / `waitForAll` / `output: empty`, 5 inputs. Noted on the node as unverified (Integrator item 11).
9. **Code - Normalise And Calculate Metrics** — see §5.
10. **IF - Enough Data To Build Brief?** — tests `readyToBuild` from node 9 by named reference.
11. **OpenAI - Write Executive Narrative** — **`disabled: true`**, as required. See §6 for the model choice.
12. **Code - Build Brief HTML And Chart URLs** — see §5.
13. **HTTP Request - Create Google Doc From HTML** — multipart/related POST to the Drive upload endpoint. See §7.
14. **Google Drive - Export Brief As PDF** — `file:download`, `googleFileConversion.conversion.docsToFormat: application/pdf`.
15. **Telegram - Request Brief Approval** — `sendAndWait`, `approvalType: double`, `chatApproval: true`, 6-hour limit. `chatId` is `SET_ME_telegram_chat_id`. No credential attached. `retryOnFail: false` deliberately.
16. **IF - Was The Brief Approved?** — see §8, the defensive null-check.
17. **Gmail - Send Brief To Leadership** — `send`, PDF attached via `attachmentsUi.attachmentsBinary`. `sendTo` is `SET_ME_leadership_email_list`.
18. **Telegram - Post Brief Summary** — posts node 12's summary text. Same `SET_ME_telegram_chat_id`.
19. **IF - All Sources OK For Snapshot?** — tests `dataComplete` from node 9.
20. **Google Sheets - Append Weekly Snapshot Row** — `append`, `columns.mappingMode: defineBelow`, all six columns named explicitly, `synthetic` hardcoded `false`.
21. **Telegram - Alert Brief Not Sent** — fires on rejection or timeout. See §8 for how it tells them apart.
22. **Telegram - Alert Run Blocked** — carries `blockReason` from node 9.
23. **NoOp - End Run Without Snapshot** — explicit end marker.
24. **Error Trigger - Catch Unhandled Failure** — a second trigger, workflow's own error-workflow setting points at itself (`settings.errorWorkflow = iIypy4KWtaqsowAQ`), set after creation once the id existed.
25. **Telegram - Alert Workflow Failed** — defensive, try/catch-wrapped text (see §9).

---

## 3. Deviation: node 2 has 5 `$now` calls, not 3

Architecture section 5 expects exactly three `$now` hits inside node 2 (`weekStart`, `weekEnd`,
`weekLabel`), with `runId` implied to just copy `weekLabel`'s value within the same node.

I could not confirm that an n8n Set node's assignments can read a sibling assignment set moments
earlier in the same node call — the common, well-documented behaviour is that each assignment's
expression is evaluated against the *original* incoming item, not a progressively-built one. If
that is right, a formula like `{{ $json.weekLabel }}` inside the same Set node would silently
resolve to nothing, and `runId` would come out blank — a correctness bug that would only show up
at runtime, on the one field this whole design uses as the dedupe key.

Rather than risk that, I duplicated the `weekLabel` formula for `runId` (one more `$now` call),
and added a sixth field, `triggeredAt`, so that node 20's `run_at` column has a real timestamp
without any node *outside* node 2 calling `$now` either. Node 2 now has 5 `$now` references
instead of 3 — all still confined to that one node, so the actual rule ("date math lives in one
place, passed down") is intact, but a literal `grep -c '$now'` on node 2 will find 5, not 3.

**This needs Designer/QA sign-off**, not a silent pass. I documented the reasoning directly on
the node's `notes` field so it is visible on the canvas, not just in this report.

---

## 4. HubSpot: a placeholder filter, because item 4 is still open

PRD §12 item 4 (which pipeline, which stages count as "in pipeline") is open and parked. I could
not invent a business answer to that. What I built instead: a **functional** filter —
`hs_lastmodifieddate` (a property every HubSpot account has) `GTE`/`LT` the week bounds, converted
to epoch-millis with `Date.parse(...)`, because HubSpot's Search API expects milliseconds for
datetime filters, not ISO strings. This makes the node syntactically complete and testable once a
credential exists, but it is **not** the real business filter PRD item 4 needs. I set
`authentication: "appToken"` (HubSpot Private App / Service Key) rather than the node's own
default of `"apiKey"`, because HubSpot deprecated plain API keys — whoever creates the HubSpot
credential should create a Private App token, not an API Key one. Both the filter and the
authentication choice are called out on the node's `notes`.

---

## 5. The two Code nodes

**Node 9** classifies each of the five sources as OK / Empty / Failed without ever crashing on
an unrecognised shape (per the explicit instruction to guard this): any node reference that
throws, returns something other than an array, or returns a single item carrying an `error` key
is treated as **Failed**, never silently as OK or as a zero. It computes variance, week-over-week
change, pipeline movement by stage, and a revenue trend list, and decides `readyToBuild` /
`blockReason` / `alreadyAppended` for the idempotency check. One caveat I could not remove: the
snapshot sheet has no metric columns yet (by design — `metric-definitions.md` is empty), so every
historical point in `revenueTrend` carries `revenue: null` until a real "revenue" metric key
starts appearing in Finance Actuals. The chart code (node 12) already treats fewer than two real
points as "not enough history" rather than drawing a broken chart.

**Node 12** builds the whole brief as one HTML string (Route A), the two QuickChart `<img>` URLs,
the Telegram summary, and reads the OpenAI narrative defensively — since node 11 is disabled, and
the exact "simplified output" field name for that OpenAI operation was never independently
confirmed, node 12 tries several plausible field names and falls back to a visible
"Narrative pending" placeholder rather than an empty paragraph.

---

## 6. OpenAI: disabled, and the model was picked from a live list, not memory

Per `money-facts.md`, node 11 is built with **`disabled: true`** and must stay that way until
QA passes and Leo/Tester explicitly enable it.

The Integrator flagged the exact model id as a blocking unknown. Rather than type one from
memory — the gotchas file is explicit that this fails only after billing, not at validation — I
queried the *live* model list through the real OpenAI credential (`DBmj9DTgWeT0tOGw`) using a
read-only `listSearch` call. That is a metadata lookup, not a generation call, so it does not
touch the spending guard. I chose **`gpt-5-mini`**, the cheapest model in the Integrator's
confirmed-from-OpenAI's-own-pricing-page table ($0.25 / $2 per 1M tokens). This is recorded on
the node's `notes` so it is easy to double check before enabling.

---

## 7. HTTP Request node 13: the multipart body

Google's multipart upload needs `multipart/related`, not the HTTP Request node's built-in
"Form-Data" option (which produces `multipart/form-data` — a different, incompatible boundary
convention). I built the body by hand: `contentType: "raw"`, an explicit
`multipart/related; boundary=...` header, and an expression that assembles the metadata JSON part
and the HTML part with the right `Content-Type` headers between them. I did **not** set a Drive
folder parent — the `Executive Reporting Rollup` folder referenced in `data-sources.md` was
created under the same now-trashed Google account as the original sheets and was never
re-verified, so the Doc is created in the credential's own My Drive root instead. This is safer
than pointing at a folder that might not exist for this account, but it does mean the Doc will
not automatically appear next to the three sheets. Worth a decision once someone checks Drive by
hand.

---

## 8. The two defensive checks the task specifically asked for

- **Node 16** (`IF - Was The Brief Approved?`): the condition is
  `$('Telegram - Request Brief Approval').item.json.approved === true`. A strict `=== true`
  comparison means `false`, `null`, and a completely **absent** `approved` key (the unverified
  timeout case) all evaluate to `false` and take the same "not sent" branch. Nothing assumes the
  key exists.
- **Node 21** (`Telegram - Alert Brief Not Sent`): tells a rejection apart from a timeout by
  checking whether `approved` is exactly `false` (declined) versus anything else, including
  absent (timed out) — rather than the timestamp comparison the architecture describes. Both
  outcomes take identical actions (no email, no summary, no snapshot row); only the alert wording
  differs, so this substitution carries no correctness risk, and it is simpler than computing
  elapsed time against a request timestamp that may not exist either.

---

## 9. Error Trigger's output shape — genuinely unverifiable here

`get_node` describes a node's *configurable input parameters*, not a trigger's *runtime output
shape*, and there is no live way to inspect what `Error Trigger - Catch Unhandled Failure`
actually emits without running it, which I am not permitted to do. Node 25's text is wrapped in a
try/catch with fallbacks for every field it reads, specifically so a wrong guess about that shape
cannot make the failure-alert itself fail silently.

---

## 10. SET_ME placeholders left in the workflow

| Node(s) | Field | Why |
|---|---|---|
| HubSpot (3) | credential | none exists on the instance |
| Supabase (4) | credential, `tableId`, both `keyName`s, `orderBy` | PRD §12 item 3, parked; no credential to pick real values from |
| Telegram (15, 18, 21, 22, 25) | credential, `chatId` = `SET_ME_telegram_chat_id` | PRD §12 item 8, open; no Telegram credential exists yet |
| Gmail (17) | `sendTo` = `SET_ME_leadership_email_list` | PRD §12 item 7, open |

No plausible-looking fake value was used anywhere — every unknown is an obvious `SET_ME_` string.

---

## 11. What is left unattached / not yet possible

- HubSpot, Supabase, Telegram credentials do not exist on this instance. The three HubSpot/
  Supabase/Telegram-carrying nodes are fully shaped but cannot run until Leo creates those
  credentials in the n8n web interface.
- The real HubSpot pipeline/stage filter (item 4), the Supabase table and date column (item 3),
  the email list (item 7), and the Telegram chat id (item 8) are all still open PRD items.
- The riskiest unverified assumption in the whole design — whether Google Drive's HTML-to-Doc
  conversion actually fetches the QuickChart `<img>` URLs — is unchanged by this build. It still
  needs the throwaway test the Integrator specified before node 13/14 are trusted.
- Whether the PDF binary field survives node 15 (Telegram `sendAndWait`) and node 16 (IF)
  unchanged, so Gmail (node 17) can attach it, is also still unverified — flagged on node 17's
  notes.
- Merge's single-empty-item behaviour with a possibly-error-shaped input (Lock 1 of the
  single-OpenAI-call guarantee) is unchanged by this build and still needs a live throwaway test.

None of these are things I could resolve without either a credential I do not have or a live run
I am not permitted to make. All are named explicitly, on the affected node's own `notes` field
and here, rather than built around silently.

---

## 12. Validation and verification

- `mcp__n8n-mcp__validate_node` — run individually on every non-trivial node configuration
  (HubSpot, Supabase, Merge, IF, Google Sheets append, OpenAI, Telegram sendAndWait, Gmail send,
  Code, Schedule Trigger, Set, HTTP Request, Google Drive, Telegram sendMessage, NoOp, Error
  Trigger) before assembly. All passed with zero errors. Google Sheets append raised four
  warnings (suggesting `valueInputMode`, and that `range` include the sheet name) — I added
  `valueInputMode: "USER_ENTERED"`; the "range should include sheet name" warning is a
  false-positive for this build, since the sheet is already identified by the `sheetName`
  resource locator (gid), not folded into the range string.
- `mcp__n8n-mcp__n8n_validate_workflow` on the assembled workflow: **`valid: true`, 0 errors**,
  25 nodes (24 enabled, 1 deliberately disabled), 2 trigger nodes, **26 valid connections, 0
  invalid**, 26 expressions validated. Three warnings, all expected/benign:
  - "Connection to disabled node: OpenAI ... from IF ..." — correct and intentional; node 11
    must stay disabled.
  - Two "possible missing $ prefix" expression warnings, on node 13's body (triggered by the
    literal string `application/json` and the built-in `JSON.stringify`) and node 25's text
    (triggered by the same class of keyword match on `$json`/`JSON`, not an actual bug) — both
    checked by hand against the actual expression text; both false positives from a
    keyword-matching linter, not real errors.
- **`n8n_get_workflow`, read by hand, three times**: `mode: structure` to confirm the full
  `connections` object — every fan-out, every Merge input index, both IF branches on all three
  IF nodes, and the Error Trigger island — matches architecture section 3 node for node, branch
  for branch, with no dangling outputs. Then `mode: filtered` twice more, across ten of the
  highest-risk nodes (HubSpot, Supabase, both Code nodes, OpenAI, HTTP Request, both approval-path
  Telegram nodes, Gmail, and the snapshot append), to confirm `onError`, `retryOnFail`,
  `maxTries`, `waitBetweenTries`, `disabled`, `credentials` and the full code/expression text all
  persisted exactly as configured, with no truncation.
- I deliberately did **not** run `n8n_autofix_workflow` — there were no real errors to fix, and
  running an automatic fixer against a workflow with deliberately unusual constructs (a
  large hand-built multipart body, IIFE expressions) seemed more likely to cause an unwanted
  change than to catch anything real.
- The workflow was **not activated and not run**, per the hard rule. `n8n_test_workflow` was
  never called.

---

## 12a. A second, stricter validation pass

After the report above, I re-ran `n8n_validate_workflow` with `profile: "strict"` as a final
check (the earlier pass used the default `runtime` profile). Still **0 errors**, but it surfaced
more warnings worth going through honestly rather than leaving buried:

- **"Property 'range' won't be used - not visible with current settings"** on nodes 5, 6, 7 and
  20. The schema I read never showed a display condition on the top-level `range` field, so this
  was a surprise. My working theory, which I could not fully confirm from the schema alone: once
  `sheetName` uses the resource-locator "list" mode with a gid (exactly what the Orchestrator
  asked for), the node's real range-detection mechanism is the nested
  `options.dataLocationOnSheet` setting, and the top-level `range` becomes a vestigial field the
  schema still marks `required: true` but the running node ignores. I fixed the substance —
  added `options.dataLocationOnSheet.values.rangeDefinition: "detectAutomatically"` explicitly
  on all four nodes, which is the field this validator implies actually governs behaviour — and
  left the top-level `range` populated rather than removing it, since removing a
  schema-`required` field seemed more likely to break something than leaving an inert one in
  place. "Detect automatically" is the correct behaviour here regardless: it reads the real
  header row, which is exactly what a 6- or 8-column sheet needs. The warning still fires after
  the fix; I read that as the validator continuing to flag the now-confirmed-inert field, not as
  a sign the fix did nothing. **Worth QA independently confirming once a real read is possible.**
- **"telegram node without error handling"** on node 15, despite it explicitly carrying
  `onError: "stopWorkflow"`. I believe the checker treats `stopWorkflow` as equivalent to "no
  handling configured" because it is also what happens with no `onError` set at all. It is set
  deliberately here, matching architecture section 9's retry table exactly (retry off, because a
  retry would double-send the approval). Left as is.
- **"Code nodes can throw errors - consider error handling"** on nodes 9 and 12. No `onError` on
  either, matching architecture's retry table, which does not list either Code node. A bug in
  this code should surface as a real failure and reach the Error Trigger, not be swallowed.
- **"Hardcoded nodeCredentialType detected"** on node 13. `nodeCredentialType` is supposed to be
  a literal string naming the credential type (`googleDriveOAuth2Api`) — that is how
  `predefinedCredentialType` auth is configured on the HTTP Request node. Not a real issue.
- The two "possible missing $ prefix" expression warnings (node 13's body, node 25's text) are
  unchanged from the first pass — confirmed false positives, see §12 above.
- Fixed as a genuine, low-risk improvement: node 14's `fileId` resource-locator now carries a
  `cachedResultName` ("Google Doc created by node 13 (dynamic id)"), since the value is a runtime
  expression and had none. This is UI cosmetics only — the warning itself said the workflow would
  run regardless — but it was a one-line fix with no downside.
- Left alone, same reasoning as the first pass: the two sub-workflow suggestions (architecture
  section 9 decided against sub-workflows, with reasons), and the `alwaysOutputData` suggestion
  on node 13 (would work against its deliberate `stopWorkflow` behaviour).

Connections were re-verified with `n8n_get_workflow` (`mode: structure`) after these two fixes:
still 20 connection entries, identical to architecture section 3, no regressions.

## 13. Fixes after QA v1

Applied to the live workflow via `n8n_update_partial_workflow`, in three batches. No rebuild.
Not activated. Not run. Node 11 (`OpenAI - Write Executive Narrative`) confirmed still
`disabled: true` throughout and after.

Re-validated after: `n8n_validate_workflow` returns `valid: true, 0 errors, 3 warnings` — the
same three benign warnings from the very first pass (disabled-node connection, and two
keyword-matching linter false positives on `JSON`/`$json`, already explained in §12). Connections
re-read via `n8n_get_workflow` (`mode: structure`): still 20 connection entries, byte-identical
to before the fixes — none of this touched the wiring.

### BLOCKER 1 — fixed, both places

- **Node `IF - Was The Brief Approved?`.** Before:
  `={{ $('Telegram - Request Brief Approval').item.json.approved === true }}`. After:
  `={{ ($('Telegram - Request Brief Approval').item.json.data?.approved ?? $('Telegram - Request Brief Approval').item.json.approved) === true }}`
  — exactly QA's suggested shape. Reads the documented `data.approved` path first, falls back to
  a top-level `approved` for safety, keeps the strict `=== true` so `false`/`null`/absent all
  still route the same way.
- **Node `Telegram - Alert Brief Not Sent`.** Same bug, same fix, inside the IIFE that decides
  "declined" vs "expired" wording. Before: `var approved = $('Telegram - Request Brief Approval').item.json.approved;`.
  After: `var approved = ($('Telegram - Request Brief Approval').item.json.data?.approved ?? $('Telegram - Request Brief Approval').item.json.approved);`

### BLOCKER 2 — fixed, both reads, plus the fallback text it protected

- **Node `Code - Build Brief HTML And Chart URLs`.** Before:
  `const metricsData = $('Code - Normalise And Calculate Metrics').item.json;` and
  `const ai = $('OpenAI - Write Executive Narrative').item.json;`. After: both use `.first().json`
  instead of `.item.json`, matching node 9's already-correct pattern. No loop node exists
  anywhere in the workflow, so `.first()` is safe here exactly as architecture section 5
  reasons for node 2.

### BLOCKER 3 — fixed, all five nodes

`alwaysOutputData: true` added to nodes 3, 4, 5, 6 and 7, alongside their existing
`onError: continueRegularOutput`. Verified present on all five via `n8n_get_workflow`
(`mode: filtered`) after applying.

### MAJOR 1 — investigated, NOT changed. QA's finding was itself wrong.

QA's instruction was "call `get_node` and confirm the path before you change it." I did — twice,
independently, reading the raw schema text directly rather than a summarised view, specifically
checking for any `@version` gate I might have missed the first time. **`additionalFields.properties`
is real.** It is a `multiOptions` field labelled "Field Names or IDs", inside the "Options"
collection named `additionalFields`, with `displayOptions.show: { resource: ["deal"], operation: ["search"] }`
— no version condition, unconditional for the search operation. I quote the exact schema block:

```
"displayName": "Options", "name": "additionalFields", "type": "collection",
"options": [ "Direction", "Field Names or IDs" (name: "properties", multiOptions,
loadOptionsMethod: getDealProperties), "Query", "Sort By" ],
"displayOptions": { "show": { "resource": ["deal"], "operation": ["search"] } }
```

`filters.properties` — the path QA proposed instead — belongs to a *different* operation,
`deal:getAll`, not `deal:search` (confirmed from the same schema read: that `filters` collection's
`displayOptions.show` is `{ resource: ["deal"], operation: ["getAll"] }`). The `search` operation
has no `filters` collection at all. Setting `filters.properties` on a node configured with
`operation: "search"` would be a completely inert parameter — invisible in the UI and unused at
runtime — which would have reintroduced the exact bare-Deal-ID trap QA is worried about, while
looking fixed. **I left `additionalFields.properties` as originally built** and added a note on
the node recording that this was re-verified during QA v1. Flagging this discrepancy back to the
Orchestrator rather than silently applying an incorrect instruction, per my role: I build what's
approved, and a factual schema disagreement is exactly the kind of thing that has to surface, not
get papered over in either direction.

### MAJOR 2 — fixed

Both `hs_lastmodifieddate` filter entries on node 3 changed `"type": "string"` to `"type": "number"`,
matching the schema: the `operator` field only offers `GT`/`GTE`/`LT`/`LTE` when `type` is
`"number"`; under `"string"` the options are `CONTAINS_TOKEN`/`EQ`/`HAS_PROPERTY`/`NOT_HAS_PROPERTY`/`NEQ`.
`GTE`/`LT` are what the date-window filter needs, so `type` had to change, not the operators.

### MAJOR 3 — fixed

Node 13 (`HTTP Request - Create Google Doc From HTML`): `retryOnFail` changed `true` → `false`,
`maxTries` changed `3` → `1`. A create is not idempotent; retrying a timed-out-but-completed
request would leave two or three Docs for the same week.

### MAJOR 4 — fixed

Node 20 (`Google Sheets - Append Weekly Snapshot Row`): same change, `retryOnFail: false`,
`maxTries: 1`. An append is not idempotent either, and this is the workflow's only write.

### MAJOR 9 — fixed

Node 12's narrative fallback text changed from
`"Narrative pending: the OpenAI node is disabled until it is reviewed and switched on."` to
`"Narrative unavailable this week: the automated summary could not be generated or read for this run."`
— true whether node 11 is disabled, or enabled and the field-name guess in the `try/catch` above
it fails to match the real output shape. No longer asserts a specific cause that could become
false the moment someone enables the node.

### MINOR 3 — resolved via the notes, not via a code restructure

QA offered two options: reduce `'Europe/London'` to one place, or document the five-edit
requirement on the node's notes. Reducing to one place is not achievable inside a single Set
node without either (a) trusting one assignment can read a sibling set moments earlier in the
same node call — the exact risk I built around originally, still unverified — or (b) adding a
second node, which is a structural change I was told not to make. I took QA's second option:
node 2's `notes` now names the five exact locations and instructs that all five be changed
together when PRD §12 item 9 closes.

### MINOR 5 — fixed, one policy everywhere

Standardised `appendAttribution: false` (no "sent automatically with n8n" line) across every
human-facing message: node 15 (`options.appendAttribution`, was `true`), node 17 (already
`false`, unchanged), and nodes 18, 21, 22, 25 (`additionalFields.appendAttribution`, previously
absent so the node's own default of `true` applied). Confirmed via `n8n_get_workflow` that all
five now read `false` and node 17 is unchanged.

### What I did not touch, per explicit instruction

BLOCKER 4 (Telegram credential — Leo's), MAJOR 5 (dedupe-window timing — a Designer ruling),
MAJOR 6 (PDF-binary-survives-approval — a Tester question), MAJOR 7 and MAJOR 8 (snapshot has no
metrics; Supabase/Support unused — both gated on PRD §12 item 5, Leo/Designer), MINOR 1
(QuickChart long-URL — a future improvement, not applied to avoid adding nodes beyond the
approved 25), MINOR 2 (synthetic flag for test rows — a Tester-protocol matter), MINOR 4 (the
`range` field being cosmetically unused — already addressed by the `dataLocationOnSheet` fix in
the first validation round, nothing further to do).

## 14. Files touched

- Updated on the n8n instance: workflow `iIypy4KWtaqsowAQ`,
  "Executive Reporting Rollup - Weekly Leadership Brief" (still inactive, still never run).
- `workflows/executive-reporting-rollup-v1.json` — local copy, refreshed twice: once to reflect
  this file (§13, the QA v1 fixes) and, in the same pass, to catch up the strict-profile fixes
  from §12a that the previous save had missed (`cachedResultName` on node 14,
  `dataLocationOnSheet` on nodes 5, 6, 7 and 20). It now matches the live workflow exactly, field
  for field, confirmed by the `n8n_get_workflow` reads in §13.
- `docs/reports/coder-v1.md` — this file, §13 added.
- `docs/knowledge/data-sources.md` — unchanged this round.

---

## 15. Telegram wiring — closing QA BLOCKER 4 (2026-09-14)

Leo created the Telegram credential and supplied his chat id. Wired both into the five
Telegram nodes via `n8n_update_partial_workflow`. No rebuild. Not activated. Not run.

**Schema check first, per the hard rule.** Called `get_node` on `nodes-base.telegram`
(`search_properties` for `chatId`, plus a `standard`-detail pull) before writing anything.
Confirmed for both operations used in this workflow:
- `chatId` sits at the same top-level path, `parameters.chatId`, for both `sendAndWait`
  (gated on `resource: message` + `operation: sendAndWait`) and `sendMessage` (gated on
  the wider message-operation list including `sendMessage`). No nesting difference between
  the two — my working assumption going in was that they might differ, and the schema
  says they do not.
- Credential type is `telegramApi` (`hasCredentials: true` on the node; the exact type
  name matches what `data-sources.md` already recorded from the live credential list).

**What changed, all five nodes, via `updateNode` operations (dot-notation `updates`, not a
full parameter replace, so nothing else on any of these nodes was touched):**

| Node | `parameters.chatId` | `credentials` |
|---|---|---|
| n15 `Telegram - Request Brief Approval` | `SET_ME_telegram_chat_id` | `{ telegramApi: { id: xkNXmRcwoz0Htiqy, name: "Executive Rollup Bot Telegram account" } }` |
| n18 `Telegram - Post Brief Summary` | `SET_ME_telegram_chat_id` | same |
| n21 `Telegram - Alert Brief Not Sent` | `SET_ME_telegram_chat_id` | same |
| n22 `Telegram - Alert Run Blocked` | `SET_ME_telegram_chat_id` | same |
| n25 `Telegram - Alert Workflow Failed` | `SET_ME_telegram_chat_id` | same |

Read back live afterward with `n8n_get_workflow` (`mode: filtered`, all five node names) and
confirmed all five now carry the real chat id and the credential block, with `text`/`message`,
`additionalFields`, `options`, `appendAttribution`, `onError`, `retryOnFail`, `maxTries`,
`waitBetweenTries`, `webhookId` and `notes` byte-identical to before the edit.

**Validation after:** `n8n_validate_workflow`, `profile: runtime` — `valid: true`, **0 errors**,
**3 warnings** (disabled-node connection 10→11, plus the same two false-positive `$`-prefix
warnings on node 13 and node 25 already triaged in §12/§12a), 25 nodes, 24 enabled, 26 valid
connections. Unchanged from the pre-edit baseline in every count.

**Connections and node 11, re-confirmed by hand:** `n8n_get_workflow` (`mode: structure`)
read after the edit — `connectionCount: 20`, node-for-node identical to §2/§12a, and
`"id": "n11"` still carries `"disabled": true`. Neither was touched by this change, and
neither should have been.

**Local copy refreshed.** `workflows/executive-reporting-rollup-v1.json` updated to match:
the `chatId` placeholder replaced with `SET_ME_telegram_chat_id` on all five nodes (one `replace_all`
edit, since the placeholder string was identical across all five), and a `credentials` block
plus `webhookId` inserted on each of the five node objects (anchored on each node's unique
`position` line to avoid touching any node's `notes`/`message`/`text` content, some of which
carry escaped `§` and emoji sequences that were safer left untouched). The `webhookId`
values were not previously in the local copy at all — n8n assigns one to every node of a
webhook-capable type on creation, and the local file had never been synced against that
field until now. Verified after editing: no `SET_ME_telegram_chat_id` strings remain in the
file (`grep` confirms zero matches), and all five `credentials` blocks are present.

**BLOCKER 4 is closed.** The only thing still open on the Telegram front is unchanged from
before this edit and is not mine to fix: whether the bot can actually deliver to Leo (it
cannot message him until he has messaged it first, per `data-sources.md`) is a Tester-time
fact, not something `get_node` or `validate_workflow` can confirm.

---

## 16. Stale notes fixed after §15 (2026-09-14, same day)

The Orchestrator caught that the §15 wiring left two node `notes` fields contradicting the
new live parameters — I had correctly left `notes` untouched during §15 (out of scope at the
time), but that meant n15 and n18 still read as if the chat id and credential were missing.
Fixed via `n8n_update_partial_workflow`, `notes` only. No parameter, connection, or
`disabled` state touched.

**Scope check first.** Read every node's `notes` field before changing anything (grepped the
local copy, which was still in sync since §15 never touched notes, then spot-read the four
entries the grep truncated). Only n15 and n18 made a false claim. Everything else that
mentions "Telegram" or "no credential" is either still true (HubSpot on node 3, Supabase on
node 4 — both genuinely still uncredentialed) or purely descriptive (node 17's Gmail note
just names node 15 as "Telegram sendAndWait" in passing, claims nothing about its
credential). Node 21 and node 25's notes never mentioned a missing credential or chat id in
the first place. Node 22 has no `notes` field at all. None of those five needed a change.

**n15 `Telegram - Request Brief Approval` — notes now read:**
> PRD §12 item 13: Leo is the approver (settled). PRD §12 item 8 (Telegram chat) is now
> closed: chatId is SET_ME_telegram_chat_id, Leo's own Telegram user id, given by Leo on 2026-09-14, and
> the credential is Executive Rollup Bot Telegram account (xkNXmRcwoz0Htiqy). retryOnFail is
> deliberately off: a retry would send a second approval request for the same brief, which is
> a correctness bug not a reliability improvement.

The retryOnFail sentence is verbatim unchanged, as instructed. Only the first two sentences
(the false OPEN/placeholder claim) were replaced.

**n18 `Telegram - Post Brief Summary` — notes now read:**
> PRD §12 item 8 (Telegram chat) is now closed. chatId is SET_ME_telegram_chat_id, the same real chat id
> as node 15 -- same chat. onError continues: by the time this runs the PDF has already
> reached leadership, so a Telegram hiccup must not stop the snapshot row being written.

The onError sentence is verbatim unchanged, as instructed. Only the first two sentences were
replaced.

**Verification.** `n8n_validate_workflow` (`profile: runtime`) after the edit: identical to
§15's result — `valid: true`, 0 errors, 3 warnings, 25 nodes, 24 enabled, 26 valid
connections. `n8n_get_workflow` (`mode: filtered`) on n15 and n18 confirms the notes above
are live, and that `parameters`, `credentials`, `onError`, `retryOnFail`, `maxTries`,
`waitBetweenTries` and `webhookId` are byte-identical to §15's state — this edit touched
`notes` and nothing else. `n8n_get_workflow` (`mode: structure`) re-confirms `connectionCount:
20` and `n11` still `"disabled": true`.

Local copy (`workflows/executive-reporting-rollup-v1.json`) updated to match both notes
exactly.

---

## 17. Source-classification ruling built (2026-09-14)

**Built against `docs/architecture/v1-source-classification-ruling.md`, section 4, node by
node. Nothing beyond section 4 was touched.** Leo approved all three open questions YES, so
points 1 and 2 (empty sources still get a snapshot row; empty sources are named on the brief
as a plain line, not a warning) are built into node 9's logic. Point 3 (the Tester's free
`alwaysOutputData` experiment on Support Log) was left alone, as instructed.

**Method.** One `n8n_update_partial_workflow` call, 13 operations, `validateOnly: true` first
(passed clean), then applied for real (`operationsApplied: 13`). Then `n8n_validate_workflow`,
then `n8n_get_workflow` in `structure`, then `filtered` on every node I touched, read by eye.

### 17.1 Nodes 3–7 — `onError`

All five (`HubSpot - Fetch Deals And Pipeline`, `Supabase - Fetch Product Usage`, and the three
`Google Sheets - Fetch …` nodes) changed from `onError: "continueRegularOutput"` to
`onError: "continueErrorOutput"`. Read back live: confirmed on all five. Nothing else on these
nodes changed — `parameters`, `credentials`, `retryOnFail: true / maxTries: 3 /
waitBetweenTries: 5000`, `alwaysOutputData: true` are byte-identical to before.

### 17.2 Five new connections

Each of the five nodes' output index 1 (the new error output) now feeds the exact same Merge
input its output 0 already used. Confirmed by eye in the live `connections` object — e.g.
`HubSpot - Fetch Deals And Pipeline` now has two output arrays, `main[0]` and `main[1]`, each
containing a single target `{ Merge - Wait For All Five Sources, index: 0 }`. Same pattern on
all five, at indexes 0–4, matching the original per-source assignment exactly. No existing
connection was removed or re-pointed.

**Node 8 (`Merge - Wait For All Five Sources`) itself was not touched.** Its own parameters —
`mode: "chooseBranch"`, `numberInputs: 5`, `output: "empty"` — and its notes are byte-identical
to before, confirmed by a filtered read after the edit.

**On the connection count.** `n8n_get_workflow mode: structure` reports `connectionCount: 20`,
unchanged from before the edit. This is not a miss — that field counts distinct *source node*
keys in the connections object, and I added new array entries under five keys that were already
present (HubSpot etc. already had an outgoing connection to Merge), so the key count cannot
change from this edit; it would only rise if a new node started sending its first connection.
The number that actually moves is `n8n_validate_workflow`'s `validConnections`: **26 before this
change (QA v2, and reproduced by re-deriving it), 31 after** — exactly +5, matching "five
connections added, none removed" precisely. I'm reporting both numbers rather than the one that
happens to look like the ruling's prose, because the prose and the structure-mode field don't
actually mean the same thing, and picking whichever number matched would have hidden that.

### 17.3 Node 9 — the classifier, rewritten

`classify()` and its five call sites replaced; the rest of the node (`toNum`, `previousWeekLabel`,
the metrics/pipeline/revenueTrend build, `readyToBuild`/`blockReason`) is untouched. Still
`mode: runOnceForAllItems` in effect (not explicitly set — see 17.6), still ends in one
`return [{ json: {...} }]`. Confirmed live: `itemsOutput` shape unchanged, single return intact.

Rule order, read back from the live `jsCode`:

1. `$(nodeName).all(1, 0)` in try/catch. ≥1 items → `state: 'failed'`, note from
   `json.error.message` / `String(json.error)` / `'The node reported an error.'`, truncated to
   300 chars.
2. `$(nodeName).all(0, 0)` in try/catch. Throw → `state: 'failed'`.
3. Belt-and-braces: any main-branch item with an `error` key → `state: 'failed'`.
4. Every item whose `json` has zero keys is dropped — it is `alwaysOutputData` padding, never a
   row.
5. Zero rows left → `state: 'empty'`, `ok: true`, `rowCount: 0`, note `'Empty week: the source
   was read successfully and had no rows.'`
6. Otherwise → `state: 'ok'`, `ok: true`, `rowCount: rows.length`.

**Fail-closed deviation from the ruling as written — flagged, not silent.** The ruling's rule 1
says a throw on `.all(1, 0)` "is never treated as a failure on its own" and falls through to
rule 2. I built it differently, on the Orchestrator's explicit instruction: a throw, or any
non-array return, from `.all(1, 0)` now returns `state: 'failed'` immediately, before the main
branch is even read:

```js
let errorItems;
try {
  errorItems = $(nodeName).all(1, 0);
} catch (e) {
  diagnostics.errorBranchThrew = true;
  diagnostics.errorBranchMessage = String((e && e.message) || e).slice(0, 300);
  return {
    state: 'failed', ok: false, rowCount: 0, rows: [], diagnostics,
    note: ('Could not read the error output of "' + nodeName + '": ' + diagnostics.errorBranchMessage +
      '. Treated as failed -- fail-closed, see docs/architecture/v1-source-classification-ruling.md.').slice(0, 300),
  };
}
if (!Array.isArray(errorItems)) {
  diagnostics.errorBranchThrew = true;
  diagnostics.errorBranchMessage = 'Error output of "' + nodeName + '" did not return a list.';
  return { state: 'failed', ok: false, rowCount: 0, rows: [], diagnostics,
    note: (diagnostics.errorBranchMessage + ' Treated as failed -- fail-closed.').slice(0, 300) };
}
```

**Why this isn't cosmetic.** With the ruling's literal wording, a throw on `.all(1, 0)` combined
with a genuinely-failed source (output 0 legitimately empty, no throw there) walks straight
through rule 2 (no throw) and rule 3 (nothing to check, zero main items) into rule 5 — `state:
'empty'`, `ok: true`. A real failure would be reported as a healthy empty week. That is exactly
the class of bug this whole ruling exists to remove, just relocated to the one code path nobody
has run yet. Since `.all(1, 0)` under `continueErrorOutput` is unmeasured on this instance
(ruling section 5, item 1), I chose not to build a code path whose only failure mode is silently
producing the exact wrong answer. This is a deliberate, narrow addition on top of the ruling —
it changes behaviour in the throw/non-array case only; the documented, expected case (an array,
possibly empty) runs exactly as the ruling specifies. **This is the one place I built something
other than the ruling's literal text, and I'm flagging it rather than treating it as the ruling
itself — it should be confirmed or overruled by whoever owns the ruling once the real behaviour
is measured.**

`sourceDiagnostics` (new, one entry per source): `mainItemCount`, `blankItemCount`,
`errorBranchCount`, `errorBranchThrew`, `errorBranchMessage` — all five fields the ruling names,
nothing added. Not read by anything downstream, exactly as specified.

### 17.4 `rowCount`, and the phantom row

`rowCount` is now `rows.length` **after** step 4 has dropped every key-less item — a lone
`alwaysOutputData` blank (`{ json: {} }`) never reaches the count. A source that fails is
`rowCount: 0` (never `1`, since a failure returns before any row-counting happens at all). A
source that is genuinely empty is `rowCount: 0` with `state: 'empty'`. **The phantom row Finding
1 and NEW-1 both named — `ok: true, rowCount: 1` for a dead Supabase or an empty sheet — cannot
be produced by this code**: the only way to reach `rowCount > 0` is `state: 'ok'` with at least
one item that has a real key, i.e. an actual row.

**Three call sites now use `state`, exactly as the ruling requires**, confirmed in the live code:

- `dealRows = hubspot.state === 'ok' ? hubspot.rows : []` — the "unknown" bar at zero and the
  phantom "one deal opened" both stop being reachable.
- `alreadyAppended` only computed when `snapshot.state === 'ok' || snapshot.state === 'empty'`.
- `revenueTrend` is built from `snapshot.rows`, which are pre-filtered by step 4, so the junk
  point `{ weekLabel: '', revenue: null }` cannot occur. Node 12's `Number.isFinite(p.revenue)`
  filter on `trendPoints` is untouched, left in place as defence in depth, confirmed live.

I also switched `finance.ok` → `finance.state === 'ok'` (the metrics filters and the `available`
flag) for the same reason, even though the ruling names only the three call sites above —
leaving one silent `.ok` reference behind while its neighbours moved to `.state` would have been
the same bug in a different corner. `sources[k].ok` on the output object is still populated,
computed as `state !== 'failed'`, so nodes 10, 12 and 19 (which read `dataComplete`,
`readyToBuild`, `blockReason`) needed no edit — confirmed by re-reading all three live and
finding no reference to any per-source `.ok` field in them.

**New output fields, confirmed live on node 9's `return`:** `emptySources` (plain-English labels
where `state === 'empty'`), `sourceDiagnostics`. `missingSources` is now built from
`state === 'failed'` only — an empty source never appears in it. The "all five failed" block
reason text changed from "All five sources failed or produced no output." to "All five sources
failed." — the old wording was no longer accurate once empty stopped being lumped in with
failed; nothing downstream depends on the old string.

### 17.5 Node 11's prompt and node 12's HTML — the two lists, two instructions

**Node 11 (`OpenAI - Write Executive Narrative`, still `disabled: true`, untouched otherwise):**
system prompt rewritten to add a rule distinguishing the two lists — missing sources are still
to be treated as if they don't exist; empty sources may be named as a real, verified zero (e.g.
"no tickets were logged this week") and must never be called unavailable, missing, or unknown.
Confirmed live: the new rule reads back exactly as written, model/credentials/retry/notes all
unchanged.

**Node 12 (`Code - Build Brief HTML And Chart URLs`):** two-patch surgical edit via
`patchNodeField` rather than a full rewrite, to keep the change to exactly what's needed:

```js
const emptySourcesHtml = (metricsData.emptySources && metricsData.emptySources.length)
  ? '<p>' + metricsData.emptySources.map(s => escapeHtml(s + ': no data this week.')).join('<br/>') + '</p>'
  : '';
```

— inserted into the `html` string right after `dataWarningHtml`, as a plain unstyled `<p>`, not
the yellow warning-banner styling `dataWarningHtml` uses. Confirmed live in the assembled `html`
concatenation. Everything else in node 12 — the KPI table, both chart builds, the
`Number.isFinite(p.revenue)` filter, `telegramSummary` — is untouched.

### 17.6 Validation, and what the warning count became

`n8n_validate_workflow`, `profile: runtime`: **`valid: true`, `errorCount: 0`.** `warningCount:
3` — unchanged from before this edit. **All three are pre-existing and already triaged by QA
v2**, not new: the required "connection to disabled node" notice for node 11 (node 11 stays
disabled), and the two known false-positive `$`-prefix warnings on node 13's `body` and node 25's
`text` (both already named in QA v2's pass-1 triage). No new warning class appeared.
`validConnections: 31` (see 17.2), `expressionsValidated: 26`, `totalNodes: 25`, `enabledNodes:
24`. A `suggestions` array also appeared alongside the warnings — generic, workflow-wide advice
("most nodes lack error handling", "consider sub-workflows"), a different severity tier from
warnings/errors, not counted in `warningCount`, and not new to this specific change.

**One pre-existing item I did not touch, flagged rather than fixed:** node 9's `parameters` has
no explicit `mode` key (relies on the Code node's `runOnceForAllItems` default). This predates
my edit and section 4 doesn't list it, so per "build that, nothing more" I left it alone —
noting it here since `CLAUDE.md` build rule 1 says never rely on a default.

### 17.7 Confirmed unchanged

`n11` `"disabled": true` — confirmed live, untouched. Workflow `"active": false` — confirmed
live. No write added anywhere; node 20 is still the only append in the graph. No
`Split In Batches` / `Split Out` / `Item Lists` / `Loop Over Items` / sub-workflow node — none
existed before, none added. No `executionTimeout` in `settings` — `settings` was never touched
by this edit.

### 17.8 What I could not build as written, and why

Only the one item in 17.3: the ruling's rule 1 says a throw on `.all(1, 0)` is not evidence of
failure; I built it as evidence of failure, on explicit instruction, because the alternative has
a live path to reporting a dead source as a healthy empty week. Everything else in section 4
was built exactly as written — no other place where the live schema or tooling contradicted the
ruling's text.

### 17.9 Still not run

This build has zero executions against it. The Tester's job — re-running the workflow to see
whether `.all(1, 0)` behaves as hoped, whether Supabase now lands in `missingSources`, whether an
empty sheet now lands in `emptySources` with no warning, and the free `alwaysOutputData`
experiment on Support Log — has not started. Node 11 is disabled and the workflow is inactive, so
this build cost nothing.

---

## 18. Correction — `mode` and `language` restored on both Code nodes (2026-09-14)

The Orchestrator caught that node 9's `parameters` held only `jsCode` after §17 — `mode` and
`language` were gone, both matching the Code node's defaults exactly
(`runOnceForAllItems` / `javaScript`), which is the dangerous case: it looks correct at
runtime while no longer being an explicit setting, against `CLAUDE.md` build rule 1.

**What I checked before fixing it.** My own read of node 9 from earlier in this session, taken
*before* I made any change to that node, already showed `parameters: { jsCode: "..." }` only —
so whatever removed `mode`/`language`, it predates this session's rewrite; my `updateNode` call
used dot notation on `parameters.jsCode` specifically and shouldn't touch siblings regardless.
Node 12 was only ever touched by `patchNodeField`, which rewrites the string at one named path
and cannot remove a sibling key — so if it was missing there too, that also predates my patch.
I'm recording this for the sake of an accurate history, not as a reason to skip the fix — the
fix is correct either way, and it's now applied.

**Fix.** One `updateNode` call, two nodes, `parameters.mode` and `parameters.language` only.
`jsCode` on both nodes was not part of the operation and is confirmed byte-identical to §17.

Read back live on both nodes:

```
"parameters": {
  "jsCode": "...",       // unchanged from §17
  "mode": "runOnceForAllItems",
  "language": "javaScript"
}
```

`n8n_validate_workflow` (`profile: runtime`) after the fix: unchanged from §17 —
`valid: true`, 0 errors, 3 warnings (same three), `validConnections: 31`, 25 nodes, 24 enabled.

Local copy (`workflows/executive-reporting-rollup-v1.json`) updated: both nodes now carry
`mode` and `language` explicitly ahead of `jsCode`.

---

## 19. Fixes after QA v3 (2026-09-14)

Acting on `docs/reports/qa-v3.md` findings NEW-1, NEW-2, NEW-3, NEW-6, NEW-7, NEW-8 and
NEW-10 only. **NEW-4 and NEW-5 need a test run, not a code change — not touched.** One
`n8n_update_partial_workflow` call, 11 `updateNode` operations, applied atomically
(`operationsApplied: 11`). Every value was read from the live `get_node` schema before
being written — nothing here came from memory. Node 11 confirmed still `disabled: true`
before and after. Workflow confirmed still `active: false`. Not run.

### 19.1 NEW-1 — Schedule Trigger given a real schedule

Read live from `nodes-base.scheduleTrigger` v1.4 before writing anything: `triggerAtDay`
is a `multiOptions` field with **Monday = 1, Tuesday = 2, Wednesday = 3, Thursday = 4,
Friday = 5, Saturday = 6, Sunday = 0**, defaulting to `[0]`. This matches what QA's fix
instruction assumed — no contradiction, but it was confirmed rather than assumed, per the
task's explicit instruction to check the day numbering before writing it.

Node 1 (`Schedule Trigger - Weekly Monday 7am`) `parameters` set from `{}` to:

```json
{
  "rule": { "interval": [ { "field": "weeks", "weeksInterval": 1, "triggerAtDay": [1], "triggerAtHour": 7, "triggerAtMinute": 0 } ] },
  "misfirePolicy": "skip"
}
```

Read back live, byte-for-byte this shape. **The workflow will now run once a week, every
Monday, at 07:00, in the workflow's own time zone (`settings.timezone: "Europe/London"`,
confirmed unchanged in the same read).** `misfirePolicy: "skip"` — n8n's own default value,
now a recorded decision rather than an unset field, exactly as the task required.

### 19.2 NEW-2 and NEW-3 — Gmail attachment field and retry policy

Read live from `nodes-base.gmail` before writing: `options.attachmentsUi.attachmentsBinary
.property` has two schema entries with different defaults (`""` under one operation
scoping, `"data"` under another) — QA's own point, confirmed independently. Set explicitly
rather than relying on either default.

Node 17 (`Gmail - Send Brief To Leadership`) changes, all in one `updateNode` call:

- `options.attachmentsUi.attachmentsBinary[0].property` — was absent (empty object), now
  `"data"`. Matches node 14's binary output property.
- `retryOnFail` — `true` → `false`. `maxTries` — `3` → `1`. `waitBetweenTries` — removed
  (was `5000`, now absent). Same fix already applied to nodes 13 and 20: a Gmail send is
  not idempotent, so a retry after a timeout could deliver the brief to leadership up to
  three times.

Read back live: `attachmentsBinary: [{ "property": "data" }]`, `retryOnFail: false`,
`maxTries: 1`, no `waitBetweenTries` key. `sendTo` confirmed untouched —
`"SET_ME_leadership_email_list"`, exactly as instructed.

### 19.3 NEW-7 — parameters that were silently sitting on defaults

Every value below was read from the live schema first; none were guessed:

- **`onError: "stopWorkflow"`**, set explicitly on nodes 13, 14, 17, 20. `onError` is not a
  node-type-specific property — `get_node search_properties` for `"onError"` returned zero
  matches on both `nodes-base.gmail` and `nodes-base.httpRequest`, confirming it is a
  generic per-node engine setting rather than something in any node's own parameter schema.
  I could not re-derive its default the same way I did every other value in this list.
  I applied `"stopWorkflow"` because it is n8n's documented third `onError` state (the
  `validate_workflow` tool's own suggestion text names it by that exact string) and because
  it matches the behaviour nodes 13/14/17/20 already had with no `onError` set — nothing
  observed contradicts it. **Flagging this one rather than presenting it as schema-verified
  the same way as the rest**, per the instruction to say so rather than adapt silently.
- **`resource` / `operation`**, set explicitly:
  - Nodes 5, 6, 7 (Google Sheets fetch): `resource: "sheet"`, `operation: "read"` — schema
    default for `resource` is `"sheet"`, and `operation` under `resource: "sheet"` defaults
    to `"read"` (labelled "Get Row(s)").
  - Node 14 (Google Drive): `resource: "file"` — schema default. `operation` was already
    explicit (`"download"`) before this fix, so only `resource` was added.
  - Node 17 (Gmail): `resource: "message"`, `operation: "send"` — both schema defaults,
    and `"send"` is a valid operation under `resource: "message"`, confirmed from the same
    schema read.
- **`emailType: "html"`** on node 17. Schema marks it `required: true`; under
  `resource: "message"` + `operation: "send"` it defaults to `"html"` — matches the body,
  which is already HTML.
- **`limitType: "afterTimeInterval"`, `resumeUnit: "hours"`** on node 15, added alongside
  the existing `resumeAmount: 6` inside `options.limitWaitTime.values`. Both are the live
  schema's defaults, confirmed from `nodes-base.telegram`. The six-hour approval window is
  unchanged; it is now a recorded decision, not an implicit one.
- **`chooseBranchMode: "waitForAll"`** on node 8. Confirmed from the live `nodes-base.merge`
  schema: this is the field's only possible value when `mode: "chooseBranch"` — cannot go
  wrong, now explicit anyway.

No value in this list turned out to differ from what QA's fix instruction assumed.

### 19.4 NEW-8 — `sourceDiagnostics` stripped from node 11's payload only

Node 11 (`OpenAI - Write Executive Narrative`) stays `disabled: true`, untouched apart from
the one field the task allowed. The user-message content (`parameters.responses.values[1]
.content`) changed from serialising node 9's whole output to serialising it with
`sourceDiagnostics` removed:

```
={{ JSON.stringify($('Code - Normalise And Calculate Metrics').item.json, (k, v) => k === 'sourceDiagnostics' ? undefined : v) }}
```

Chosen over an `Object.entries`/`fromEntries`/`filter` rewrite as the simpler, more common
JS idiom for this exact job (a `JSON.stringify` replacer that omits one key), on the
"try an expression before Code" default — this is a one-line expression edit on an
existing field, not a new node. Node 9's own output is untouched: `sourceDiagnostics` is
still computed and still returned, exactly as the Tester needs it — only what node 11
*sends onward* changed. The system prompt (role: `system`) was not touched. Read back
live: the new expression is exactly as written above; model id, credentials, `disabled`,
`onError`, `retryOnFail`/`maxTries`/`waitBetweenTries`, and `notes` are all byte-identical
to before this fix.

### 19.5 NEW-6 — node 8's notes rewritten to match the current graph

Old note referred to upstream `continueRegularOutput` nodes and called the Merge behaviour
"the single most safety-critical unverified claim in the design" — both stale since the
source-classification ruling landed. New note, read back live on `Merge - Wait For All
Five Sources`:

> Five inputs, indexes 0-4, one per fetch node (HubSpot 0, Supabase 1, Finance 2, Support
> 3, Snapshot 4). Each input is fed by both outputs of its fetch node -- the main output
> and the error output both land on the same index -- so every input gets at least one
> item whether that source succeeds or fails (ruling:
> docs/architecture/v1-source-classification-ruling.md section 4.2). Verified twice on
> real data, executions 536 and 537: the Merge fired and node 9 emitted exactly one item.
> What is still unverified: whether chooseBranch/waitForAll waits on inputs or on
> individual connections. On every run exactly one of the two connections into each input
> stays silent, and if waitForAll counts connections rather than inputs the Merge could
> stall here with no error and no alert (QA v3 NEW-5) -- needs a live test before being
> trusted.

`mode: "chooseBranch"`, `numberInputs: 5`, `output: "empty"` on node 8 confirmed
byte-identical to before — only `chooseBranchMode` was added and `notes` rewritten.

### 19.6 NEW-10 — inert `waitBetweenTries` removed from nodes 13 and 20

Both already carried `retryOnFail: false`, so `waitBetweenTries: 5000` was dead text.
Removed (set to `null`) on both. Read back live: neither node has a `waitBetweenTries` key
now. `retryOnFail: false`, `maxTries: 1` unchanged on both.

### 19.7 Validation and connections, re-verified by eye

`n8n_validate_workflow`, `profile: runtime`, after all 11 operations: **`valid: true`,
`errorCount: 0`, `warningCount: 3`** — unchanged from QA v3's own baseline. Same three,
confirmed by reading their text: the expected "connection to disabled node" notice for
node 11, and the two pre-existing false-positive "possible missing $ prefix" lints on node
13's `body` and node 25's `text` (both already triaged in §12/§12a) — neither field was
touched by this round of fixes, so their presence is expected, not a regression.
`validConnections: 31`, `invalidConnections: 0`, `expressionsValidated: 26`, `totalNodes:
25`, `enabledNodes: 24`.

`n8n_get_workflow` (`mode: structure`) read after the fixes: the `connections` object is
node-for-node, index-for-index identical to the pre-fix read — same 25 nodes, same
`disabled` flags (only node 11 `true`), same every fetch node's `main[0]`/`main[1]` both
landing on the Merge's assigned input. No `addConnection`/`removeConnection`/
`rewireConnection` operation was used, so this is the expected result, confirmed rather
than assumed.

`n8n_get_workflow` (`mode: details`) confirmed `settings` unchanged (11 keys, no
`executionTimeout`, `errorWorkflow: "iIypy4KWtaqsowAQ"`, `timezone: "Europe/London"`,
`availableInMCP: true`) — `settings` was never part of any operation in this batch.
`versionCounter` moved from 21 to 22, one atomic update. `executionStats.totalExecutions:
2` — unchanged; nothing was run.

Both Code nodes re-confirmed carrying `mode: "runOnceForAllItems"` and
`language: "javaScript"` explicitly, and node 9's `jsCode` (including `classify()`)
confirmed byte-identical to §17/§18 — this fix set made no `updateNode` call against node
9 at all.

### 19.8 One scope observation, not acted on

Node 20 (`Google Sheets - Append Weekly Snapshot Row`) is also missing an explicit
`resource` (same gap as nodes 5/6/7 before this fix), but QA's NEW-7 list names nodes 5,
6, 7, 14 and 17 for `resource`/`operation` and does not include node 20. Followed the list
as given and left node 20's `resource` unset — flagging it here in case its omission from
the list was an oversight rather than a decision, rather than deciding either way myself.

### 19.9 Files touched

- Workflow `iIypy4KWtaqsowAQ` on the n8n instance — still inactive, still never run,
  `versionCounter` now 22.
- `workflows/executive-reporting-rollup-v1.json` — fully refreshed to match the live
  workflow field-for-field after this round.
- `docs/reports/coder-v1.md` — this section.

---

## 20. Repair — parameters dropped from nodes 11 and 15 during §19 (2026-09-14)

The §19 run was cut off by a rate limit near the end. Most of it landed correctly and was
verified by the main session. But on two nodes the update **replaced the whole `parameters`
object instead of merging into it**, so sibling keys that were never part of the task were
silently lost. This repair restores them and changes nothing else.

**Not activated. Not run. Node 11 still `disabled: true`. No connection touched.**

### 20.1 The cause, named — this is the third time it has happened

`updateNode` behaves in two completely different ways depending on the shape of `updates`:

| `updates` shape | Effect |
|---|---|
| `{ parameters: { … } }` | **Replaces the entire `parameters` object.** Every key not named is deleted |
| `{ "parameters.foo": value }` | **Merges.** Only the named leaf changes; siblings survive |

Node 9 and node 12 lost `mode` and `language` this way (§18). Nodes 11 and 15 lost five and
six keys the same way in §19. The two failures have one cause, and it is not a tooling bug —
it is documented behaviour that reads as harmless until a whole-object form is used by habit.

**Rule for this project from now on: `updateNode` may only ever be called with dot-notation
paths.** Never pass a bare `parameters` object. Every operation in this repair used dot
notation, one path per leaf, including array indexes (`parameters.responses.values[1].role`).

**And read back by comparing against what the node had before, not against what you set.** A
read-back that only checks the keys you wrote will pass cleanly on a node that just lost six
others. Both nodes here were read in full *before* the write, and the post-write read was
compared key by key against that earlier capture.

### 20.2 Schema checked first, every value

Read live before writing anything. Every path and every value in the task matched the live
schema exactly — no contradictions, nothing adapted.

From `nodes-base.telegram` (node typeVersion 1.2):

- `resource` — path `resource`, default `message`, `message` is a valid option.
- `responseType` — path `responseType`, default `approval`, gated on
  `resource: message` + `operation: sendAndWait`. Both already set on the node.
- `postDecisionBehavior` — path `postDecisionBehavior`, default `showOutcome`.
- `approveLabel` — path `approvalOptions.values.approveLabel`, default `✅ Approve`,
  shown when `approvalType` is `single` or `double`.
- `disapproveLabel` — path `approvalOptions.values.disapproveLabel`, default `❌ Decline`,
  shown when `approvalType` is `double` **only**. The node's `approvalType` is `double`, so
  the field applies.

From `nodes-langchain.openAi` (node typeVersion 2.3):

- `resource` — path `resource`, default `text`.
- `operation` — path `operation`, default `response` under `resource: text`.
- `simplify` — path `simplify`, **gated on `operation: response` + `resource: text`**,
  default `true`. Note this property has three separate schema entries with different
  scopings and different defaults (`image:analyze` defaults `true`, `text:classify` defaults
  `false`). Setting it explicitly is what removes the ambiguity.
- `type` — path `responses.values.type`, default `text`, options `text` / `image` / `file`.
- `role` — path `responses.values.role`, default `user`, options `user` / `assistant` /
  `system`.

### 20.3 Node 15 — five parameters restored

One `updateNode`, five dot-notation paths. Read back live:

```json
"resource": "message",
"responseType": "approval",
"postDecisionBehavior": "showOutcome",
"approvalOptions": { "values": {
  "approvalType": "double",
  "approveLabel": "✅ Approve",
  "disapproveLabel": "❌ Decline"
} }
```

The two labels are the buttons Leo taps to approve or decline the brief. Without them the
node relied on defaults that happen to carry the same text — correct by luck, not by record,
and invisible to anyone auditing the build.

**Everything the node already had survived, compared key by key against the pre-write read:**
`operation: "sendAndWait"`, `chatId: "SET_ME_telegram_chat_id"`, the full `message` expression,
`chatApproval: true`, `options.limitWaitTime.values` (`resumeAmount: 6`,
`limitType: "afterTimeInterval"`, `resumeUnit: "hours"`), `options.appendAttribution: false`,
`retryOnFail: false`, `maxTries: 1`, the `telegramApi` credential (`xkNXmRcwoz0Htiqy`),
`webhookId` `9756457e-00a8-43b0-be16-b4ae01ede2ba`, and the `notes`. All byte-identical.

### 20.4 Node 11 — six parameters restored

One `updateNode`, six dot-notation paths, two of them array-indexed. Read back live:

```json
"resource": "text",
"operation": "response",
"simplify": true,
"responses": { "values": [
  { "role": "system", "content": "…", "type": "text" },
  { "content": "={{ JSON.stringify(…) }}", "type": "text", "role": "user" }
] }
```

**`simplify: true` is the one that mattered most.** Node 12 reads the narrative with
`ai.content || ai.text || ai.output_text || ai.message.content` — field names chosen for the
simplified output shape. Without `simplify`, node 11 returns the raw API response, none of
those four names match, and node 12 falls silently back to *"Narrative unavailable this
week"* — on the only node in the workflow that costs real money. A paid call whose result is
then thrown away is the worst possible failure here, and it would have looked like nothing
was wrong.

**`role: "user"` on the second message is the other one with teeth.** `role` defaults to
`user`, so the behaviour was probably right — but the first message is explicitly `system`,
and an unset `role` beside an explicit one is exactly the kind of asymmetry that reads as
deliberate later. With it set, the model provably receives one system turn and one user turn.

**Everything the node already had survived, compared key by key:** the `modelId` resource
locator (`gpt-5-mini`, `cachedResultName: "GPT-5-MINI"`), `builtInTools: {}`, `options: {}`,
`retryOnFail: true`, `maxTries: 2`, `waitBetweenTries: 10000`, the `openAiApi` credential
(`DBmj9DTgWeT0tOGw`), `onError: "continueRegularOutput"`, `disabled: true`, and the `notes`.

Two things specifically confirmed to have survived, because both were correct fixes that must
not be reverted:

1. **The system prompt** still carries the §17.5 `emptySources` rules in full — rule (3)
   naming an empty source as a real verified zero and forbidding "unavailable, missing, or
   unknown", and rule (6) on `dataComplete: false`. Text compared in full against the
   pre-write read: identical.
2. **The `sourceDiagnostics` stripping** (the §19.4 fix for QA NEW-8) is intact and exact:
   `={{ JSON.stringify($('Code - Normalise And Calculate Metrics').item.json, (k, v) => k === 'sourceDiagnostics' ? undefined : v) }}`

**Node 11 remains `disabled: true`.** It was not enabled, and nothing here reaches it.

### 20.5 The three verification items — all three hold, nothing was damaged

Read live, in full, and again *after* the write (auto-sanitization runs over every node on
any update, so a pre-write read alone would not have settled it):

1. **Node 9 and node 12 both still carry `mode: "runOnceForAllItems"` and
   `language: "javaScript"`.** Present and explicit on both.
2. **Node 9's `classify()` is untouched**, including the fail-closed guard QA v3 upheld in
   its ruling 3 — the `try/catch` around `$(nodeName).all(1, 0)` returning
   `state: 'failed'` immediately, and the `!Array.isArray(errorItems)` branch beside it.
   Rule order 1–6 intact, all five `classify()` call sites intact, single top-level
   `return [{ json: … }]` intact.
3. **Node 12's `jsCode` still contains the `emptySourcesHtml` line**, unstyled `<p>`, still
   concatenated into `html` directly after `dataWarningHtml`.

Both Code nodes' `jsCode` is byte-identical to the §17/§18/§19 state. **No repair was needed
on any of the three.**

### 20.6 Validation and connections

`n8n_validate_workflow`, `profile: runtime`: **`valid: true`, `errorCount: 0`,
`warningCount: 3`, `validConnections: 31`, `invalidConnections: 0`,
`expressionsValidated: 26`, `totalNodes: 25`, `enabledNodes: 24`.** Exactly the expected
numbers, unchanged from QA v3's baseline in every count.

The three warnings are the same three already triaged in §12/§12a/§19.7, and none is new: the
required "connection to disabled node" notice for node 11, and the two false-positive
"possible missing `$` prefix" lints on node 13's `body` and node 25's `text`. Neither of those
two fields was touched by this repair.

**`connections` read by eye via `mode: structure`, not inferred from the count.** 20 source-node
keys, unchanged. Each of the five fetch nodes still has both `main[0]` and `main[1]`, each
holding exactly one target, both landing on the same Merge input — HubSpot 0, Supabase 1,
Finance 2, Support 3, Snapshot 4. Indexes 0–4 intact and in the designed order. Every other
connection — both branches of all three IF nodes, the Error Trigger island, the linear
12→13→14→15→16 run — is identical to the pre-repair read. No `addConnection`,
`removeConnection` or `rewireConnection` operation was used, so this is the expected result;
it was confirmed rather than assumed.

`active: false` confirmed on every read. Only `n11` carries `disabled: true`; the other 24
nodes read `disabled: false`. No workflow-level `executionTimeout` was added — `settings` was
not part of any operation in this repair.

### 20.7 The local copy and this report were both stale

The §19 run was cut off before it finished either file, so both were checked rather than
trusted.

**`workflows/executive-reporting-rollup-v1.json` was stale in exactly one way, and it is a
revealing one: it had saved the damaged shape.** Node 11 and node 15's blocks in the local
file matched the *broken* live nodes — missing the same five and six keys. Everything else
from §19 had landed correctly and needed no change: node 1's `rule` block and
`misfirePolicy: "skip"`, node 8's `chooseBranchMode: "waitForAll"` and its rewritten notes,
`resource`/`operation` on nodes 5/6/7/14/17, node 17's
`options.attachmentsUi.attachmentsBinary[0].property: "data"` and `emailType: "html"`,
`onError: "stopWorkflow"` on nodes 13/14/17/20, and the removal of the inert
`waitBetweenTries` from nodes 13, 17 and 20. Only the two damaged node blocks were edited, to
match the now-repaired live state field for field.

**`docs/reports/coder-v1.md` had §19 complete through §19.9.** This §20 is the only addition.

### 20.8 Files touched

- Workflow `iIypy4KWtaqsowAQ` on the n8n instance — one `n8n_update_partial_workflow` call,
  2 `updateNode` operations, `validateOnly: true` first (passed clean), then applied
  (`operationsApplied: 2`). Still inactive, still never run, node 11 still disabled.
- `workflows/executive-reporting-rollup-v1.json` — nodes 11 and 15 brought up to date.
- `docs/reports/coder-v1.md` — this section.

### 20.9 Nothing deviated, and one thing the Orchestrator should note

Every path and value the task specified matched the live schema exactly. There was no place
where the schema disagreed with the task's wording, so nothing was adapted and nothing was
forced.

**The one thing worth carrying forward is §20.1.** Three separate parameter losses in this
project share a single cause, and it will happen a fourth time unless the dot-notation rule is
treated as a hard constraint rather than a preference. It costs nothing to follow and it fails
silently when it is not.

---

## 21. HubSpot and Supabase credentials attached (2026-09-14)

Leo created both missing credentials himself in the n8n web interface. This section attaches
them to nodes 3 and 4. **Nothing else on either node was touched, and no connection was
changed.** Source of record: `docs/knowledge/data-sources.md`, section
"Update — 2026-09-14. HubSpot and Supabase are no longer parked."

### 21.1 Both credentials verified live before the write

Read from the instance with `n8n_manage_credentials`, action `get`, one call each. Not taken
on trust from the task text:

| Node | Credential name | Id | Type | Created |
|---|---|---|---|---|
| n3 `HubSpot - Fetch Deals And Pipeline` | `HubSpot Service Key account` | `ZhEi0614V5vjYKFJ` | `hubspotAppToken` | 2026-09-13T18:27:20Z |
| n4 `Supabase - Fetch Product Usage` | `Supabase account - <supabase-account>` | `34bKthNT33Ghzp2t` | `supabaseApi` | 2026-09-13T18:36:13Z |

`getSchema` was also run on both types and both exist on this instance with the expected
shape — `hubspotAppToken` takes `appToken`; `supabaseApi` takes `host` and `serviceRole`. The
`host` field is the one `data-sources.md` warns must be the bare project URL with no
`/rest/v1/` suffix; the node appends that path itself.

### 21.2 The HubSpot authentication check — the schema agrees

Node 3 already carried `authentication: "appToken"`. Confirmed against the live schema with
`get_node` `mode: search_properties`. `authentication` has exactly three options:

```
apiKey   -> "API Key"
appToken -> "Service Key"
oAuth2   -> "OAuth2"
```

`appToken` is labelled **Service Key**, which is precisely the credential Leo made and the
reason it is named `HubSpot Service Key account`. **The node setting and the credential type
match.** No change was needed and none was made. (Note the schema `default` is `apiKey`, so
the explicit `appToken` here is load-bearing — if it were ever dropped, the node would fall
back to a deprecated auth method.)

### 21.3 The write — dot notation, to the leaf

One `n8n_update_partial_workflow` call, two `updateNode` operations, four dot-notation paths.
`validateOnly: true` first (`valid: true`, `operationsToApply: 2`), then applied
(`operationsApplied: 2`).

```
n3: credentials.hubspotAppToken.id    credentials.hubspotAppToken.name
n4: credentials.supabaseApi.id        credentials.supabaseApi.name
```

**No `parameters` object and no `credentials` object was handed to the tool.** Paths were
taken down to the individual leaf, one level deeper than strictly necessary, because §20.1 is
the most expensive recurring mistake in this project. Neither node had a `credentials` key at
all beforehand, so there was nothing under it that a coarser path could have deleted — the
leaf paths were belt and braces.

### 21.4 Read-back, compared key by key against the pre-write capture

Both nodes were captured in full with `n8n_get_workflow` `mode: filtered` **before** the
write, and read again after. Compared against the earlier state, not just against the values
set.

**Node 3 — every pre-existing key survived, byte for byte:**

- `authentication: "appToken"`, `resource: "deal"`, `operation: "search"`, `returnAll: true`
- `filterGroupsUi.filterGroupsValues[0].filtersUi.filterValues` — both `hs_lastmodifieddate`
  bounds intact: `GTE` on `Date.parse(…weekStart)`, `LT` on `Date.parse(…weekEnd)`, both
  still `type: "number"`. **The deliberate placeholder filter stays** — PRD §12 item 4 is
  still unanswered.
- **`additionalFields.properties` still holds all six names in the original order:**
  `dealname`, `amount`, `dealstage`, `pipeline`, `closedate`, `hs_lastmodifieddate`.
  **Still at `additionalFields`, not moved to `filters`.** That was QA MAJOR 1, it was
  withdrawn as wrong, and `n8n-gotchas.md` records why: `sortBy` exists at exactly one path
  in the 87-property schema, `additionalFields.sortBy`, which makes `additionalFields` the
  search collection.
- Node level: `typeVersion 2.2`, `position [464, -304]`, `retryOnFail: true`, `maxTries: 3`,
  `waitBetweenTries: 5000`, `alwaysOutputData: true`, `onError: "continueErrorOutput"`, and
  the `notes` unchanged.
- Added: `credentials.hubspotAppToken = { id: "ZhEi0614V5vjYKFJ", name: "HubSpot Service Key account" }`

**Node 4 — every pre-existing key survived, byte for byte:**

- `operation: "getAll"`, `returnAll: true`, `matchType: "allFilters"`
- **`tableId: "SET_ME_supabase_table_name"` — unchanged.**
- **`orderBy: "SET_ME_supabase_date_column"` — unchanged.**
- `filters.conditions` — both conditions intact, both still keyed on
  `SET_ME_supabase_date_column`: `gte` on `…weekStart`, `lt` on `…weekEnd`.
- Node level: `typeVersion 1`, `position [464, -144]`, `retryOnFail: true`, `maxTries: 3`,
  `waitBetweenTries: 5000`, `alwaysOutputData: true`, `onError: "continueErrorOutput"`, and
  the `notes` unchanged.
- Added: `credentials.supabaseApi = { id: "34bKthNT33Ghzp2t", name: "Supabase account - <supabase-account>" }`

**The four `SET_ME_` placeholders on node 4 were left in on purpose.** Leo has not given the
table name or the date column. Guessing a plausible table name would turn a loud, obvious
failure into a quiet wrong one, and `CLAUDE.md` forbids a placeholder that looks like data.
They must keep failing loudly until item 3 is answered.

### 21.5 Validation and connections

`n8n_validate_workflow`, `profile: runtime`: **`valid: true`, `errorCount: 0`,
`warningCount: 3`, `validConnections: 31`, `invalidConnections: 0`,
`expressionsValidated: 26`, `totalNodes: 25`, `enabledNodes: 24`.** Identical to the §20.6
baseline in every count.

The three warnings are the same three already triaged in §12/§12a/§19.7/§20.6 — the expected
"connection to disabled node" notice for node 11, and the two false-positive "possible
missing `$` prefix" lints on node 13's `body` and node 25's `text`. None is new, and none of
those fields was touched.

**A clean validation proves nothing on its own here** (§20.1), which is why §21.4 is a diff
against the pre-write state and not a check against the values set.

**`connections` read by eye with `mode: structure`.** 20 source-node keys. All five fetch
nodes still carry both `main[0]` and `main[1]`, each with exactly one target, both landing on
the same Merge input — HubSpot 0, Supabase 1, Finance 2, Support 3, Snapshot 4. Indexes 0–4
intact and in the designed order. Every other edge unchanged: all three IF nodes' branches,
the Error Trigger island, the linear 12→13→14→15→16 run. No connection operation of any kind
was used in this edit.

`Merge - Wait For All Five Sources` read separately and confirmed unchanged:
`mode: "chooseBranch"`, `numberInputs: 5`, `output: "empty"`, `chooseBranchMode: "waitForAll"`,
`typeVersion 3.2`.

`active: false` on every read. **Node 11 `OpenAI - Write Executive Narrative` still reads
`disabled: true`** — confirmed directly, not inferred. No workflow-level `executionTimeout`
was added; `settings` was not part of any operation. The workflow was not activated and was
not run.

### 21.6 Two things the Orchestrator should note

**1. Both nodes' `notes` are now stale, and they were deliberately not fixed.** Node 3's notes
still end *"No credential attached -- none exists on this instance yet"* and tell Leo to create
the credential; node 4's still say *"No credential attached"* and *"there is no live credential
to pick real values from"*. Both statements are now false. The task said to keep the notes and
change nothing else, so they were kept. **This needs a follow-up edit** — a note that tells a
future reader a credential is missing when it is attached is exactly the kind of quiet wrong
information that costs a debugging session.

**2. The local JSON copy's notes were already ahead of live before this edit.** Nodes 3 and 4
in `workflows/executive-reporting-rollup-v1.json` carry an extra trailing sentence that the
live nodes do not — *"RULING v1 (2026-09-14): onError changed from continueRegularOutput to
continueErrorOutput…"*. This drift predates this section. It was left alone rather than
reconciled, because reconciling it in either direction means editing something this task put
out of bounds. Folding it into the note rewrite in item 1 would settle both at once.

### 21.7 Files touched

- Workflow `iIypy4KWtaqsowAQ` on the n8n instance — one `n8n_update_partial_workflow` call,
  2 `updateNode` operations, 4 leaf paths. Still inactive, still not run, node 11 still
  disabled.
- `workflows/executive-reporting-rollup-v1.json` — one `credentials` line added to node 3 and
  one to node 4. Nothing else in the file changed.
- `docs/reports/coder-v1.md` — this section.

### 21.8 Nothing deviated

Every value and path the task specified matched the live schema and the live credential store
exactly. The one item flagged for checking — whether node 3's `authentication: "appToken"`
matches credential type `hubspotAppToken` — **it does**, confirmed from the schema's own
option labels in §21.2. There was no place where the schema disagreed with the task, so
nothing was adapted and nothing was forced.
