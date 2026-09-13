# Integrator Report — v1

**Date of this check:** 2026-09-13
**Against:** `docs/architecture/v1-architecture.md`, section 11
**Author:** Integrator agent. Written to disk by the main session, because the agent has no write tool.

Everything below was checked against live provider documentation fetched on 2026-09-13, and
against the `get_node` schemas served by this instance's n8n-mcp catalogue. Per
`docs/knowledge/n8n-environment.md`, a catalogue schema is **not** proof the node runs on the
live instance. That distinction is flagged wherever it matters.

**All 13 items are now checked.** Items 7, 8, 9 and 11 were left unfinished on the first pass
because the schema dumps were too large, and were closed on a second, narrower pass on the same
day. Where a fact could not be proven from a schema, it says so and names a Tester case instead
of guessing.

---

## Priority item 3 — Google Drive HTML-to-Doc conversion (the riskiest item)

**Design assumes:** posting `text/html` in a multipart upload with
`mimeType: application/vnd.google-apps.document` produces a real Google Doc, and that
`<img src="https://quickchart.io/...">` remote URLs are fetched and embedded.

**What the docs say.** Google's upload guide confirms the multipart shape: a metadata JSON
part naming the target `mimeType` as `application/vnd.google-apps.document`, plus a media
part carrying the `text/html` content. HTML is on Google's list of formats that convert to
Docs.

**On remote images the documentation is silent.** It discusses only OCR of images already
embedded as binary inside an uploaded file. It says nothing about whether an
`<img src="https://...">` reference in raw HTML is fetched at conversion time. No source,
official or otherwise, states it either way.

**Verdict: COULD NOT VERIFY.** Documentation cannot close this.

**Recommendation.** This becomes a named Tester case before the Coder wires the happy path
around it. Build node 12's HTML with one placeholder QuickChart `<img>`, run node 13, and
open the resulting Doc by hand to see whether the chart is there. If it is not, Route B in
section 8 is the fallback and the node count changes by two.

Source: https://developers.google.com/workspace/drive/api/guides/manage-uploads — read 2026-09-13.

---

## Priority item 6 — Telegram `sendAndWait` with `chatApproval: true`

**Confirmed from the node schema, typeVersion 1.2.** The operation exists on
`resource: message`. `responseType: approval` exposes a `chatApproval` boolean whose
description matches the architecture's quote exactly: approvers respond with one tap on
buttons inside the Telegram chat, and it requires this n8n instance to be reachable over
public HTTPS. `approvalOptions.values.approvalType` offers `single` and `double`, exactly as
designed. `options.limitWaitTime` is a fixedCollection taking `afterTimeInterval`
(amount plus unit) or `atSpecifiedTime`, so 6 hours is settable.

**Output field path.** n8n's shared `sendAndWait` mechanism writes a top-level `approved`
boolean plus `respondedAt` (ISO-8601) onto the item. With `chatApproval: true` there may
additionally be a `responder` object, seen on the Slack variant, but the Telegram-specific
extra fields could not be confirmed.

**On timeout.** No documentation, official or community, states what `approved` holds — or
whether the key is present at all — when nobody responds before `limitWaitTime` expires. A
GitHub issue confirms the execution resumes when the timer fires, but not the payload shape.

**Verdicts**

| Claim | Verdict |
|---|---|
| Operation, `chatApproval`, `approvalType: double`, `limitWaitTime` shape | **VERIFIED** (catalogue schema) |
| That this node type actually runs on this instance | **UNVERIFIED** — stripped-node warning applies |
| Output field on approve and on disapprove | **VERIFIED** — `approved` true/false plus `respondedAt` |
| Output field and value on timeout | **COULD NOT VERIFY** — Tester case |
| Telegram reaching this instance's public HTTPS webhook | **UNVERIFIED** — infrastructure, not documentation |

Sources: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/approvals — read
2026-09-13. `get_node` schema for `n8n-nodes-base.telegram` v1.2, read 2026-09-13.

---

## Priority item 5 — Gmail `sendAndWait`

**Confirmed from the node schema, typeVersion 2.2.** `resource: message`,
`operation: sendAndWait` exposes `To`, `Subject`, `Message` and `Response Type`
(`approval` / `freeText` / `customForm`), then the same `approvalOptions` (single/double)
and the same `options.limitWaitTime` shape. There is no `chatApproval` equivalent — Gmail
approval is link-based.

**Attachments, checked line by line.** The `attachmentsUi` / `attachmentsBinary` field does
exist on the Gmail node, but only under the `send` and `reply` operations, confirmed at two
separate locations in the schema, both scoped to `operation: ["send"]` / `["reply"]`. It is
**absent** from the property set scoped to `operation: sendAndWait`. There is no field to
attach a binary file to a `sendAndWait` message at all.

**Verdict: CONTRADICTED.** The architecture's open question is answered. Gmail `sendAndWait`
cannot carry the PDF. If Leo picks Gmail for the approval gate, the approval message carries
the Google Doc link only. The final email to leadership at step 17 still carries the PDF —
this affects the approval step alone.

**Output fields.** Same `approved` plus `respondedAt`, verified generically rather than
Gmail-specifically. Timeout behaviour: the same gap as Telegram, **COULD NOT VERIFY**.

Source: `get_node` schema for `n8n-nodes-base.gmail` v2.2, read 2026-09-13.

---

## Priority item 1 — OpenAI

**Node schema, `resource: text`.** The operation is named "Message a Model", `value:
"response"` — matches the architecture. The schema carries a builder hint warning not to use
the old `message` value on v2, which would fail validation. `modelId` is a `resourceLocator`
with a live "From List" search (`modelSearch`, which calls the OpenAI API through the
credential) or a free-text ID mode. **There is no hardcoded model list baked into the node.**
Whatever the credential's account can see is what appears.

**Credential.** The node declares one required credential, type `openAiApi`, matching
`docs/PRD.md` and `money-facts.md` (id `DBmj9DTgWeT0tOGw`). **VERIFIED** for the node
accepting the credential type. Whether that credential still holds a valid funded key cannot
be proven from a schema — only a real call proves it, which is a Tester job.

**Model id and pricing — the weak point, flagged deliberately.** A fetch of OpenAI's pricing
page plus a web search returned aggregator tables naming models such as "GPT-6 Astra",
"GPT-5.6 Sol/Terra/Luna" and "GPT-5.5". **These could not be confirmed as real shipped OpenAI
model ids from any trusted source.** Several of those sites read like content mills
speculating on a pricing timeline. They are recorded here as untrusted, not repeated as fact.

From the pricing page fetch itself, a Standard per-1M-token table showed:

| Model | Input | Output |
|---|---|---|
| `gpt-4.1` | $2.00 | $8.00 |
| `gpt-4o` | $2.50 | $10.00 |
| `gpt-5` | $1.25 | $10.00 |
| `gpt-5-mini` | $0.25 | $2.00 |

**Verdict: COULD NOT FULLY VERIFY.** The fetch tool summarises rather than showing the raw
table, so stale or wrong rows cannot be ruled out.

**Cost estimate, if `gpt-4.1` is chosen**, at 2,000 input and 400 output tokens:
input $0.004, output $0.0032, **about $0.007 per call**. At `gpt-4o` pricing, about $0.009.
Either way this is well under one cent per week and does not need a warning per run.

**Blocking sub-item.** The exact model id must be picked by hand from the credential's live
model list before the Coder builds node 11. The Coder must not guess a model string.

---

## Item 2 — QuickChart

No dedicated n8n Chart node feeds a URL. The `quickChart` node on this instance returns
binary PNG only, matching what the architecture found. QuickChart's own documentation states
that for large or complicated configs you should use the POST endpoint or `getShortUrl()`
rather than a long GET URL, and that unencoded GET URLs break on special characters.

**Verdict: VERIFIED** that the architecture's stated fallback — POST to
`quickchart.io/chart/create` for a short URL — is QuickChart's own recommended pattern, not a
guess. No published hard character ceiling was found, so the exact limit is **COULD NOT
VERIFY**.

**Recommendation.** The Coder should always use `chart/create` for the revenue trend line,
which grows every week, rather than waiting for a failure.

Source: https://quickchart.io/documentation/usage/parameters/ — read 2026-09-13.

---

## Item 4 — PDF export, node 14

Verified directly from the `googleDrive` schema: `resource: file`, `operation: download`,
`options.googleFileConversion.conversion.docsToFormat` offers `application/pdf` among
HTML / Markdown / Word / OpenDocument / PDF, exactly as the architecture states. The output
binary field name comes from `options.binaryPropertyName`, default `data`.

**VERIFIED** from the catalogue schema. Whether it works on a Doc created seconds earlier —
a propagation-delay question — is **UNVERIFIED** and is a Tester item.

---

## Item 7 — HubSpot deals, typeVersion 2.2

**The operation.** `resource: deal`, `operation: search` (label "Search"). `deal:getAll`
("Get Many") also exists but has no filter parameters at all, only `returnAll` and `limit`.
`search` is the only operation with a filter mechanism, so it is the correct route for a
date-windowed read. The architecture guessed right.

**The date filter.** `filterGroupsUi`, a fixedCollection at path
`filterGroupsUi.filterGroupsValues.filtersUi.filterValues`, shown only for
`resource: deal` plus `operation: search`. The node's own description states that filters
within a group combine with AND, groups combine with OR, and there is a maximum of three
groups with up to three filters each.

This is a generic property-filter builder. Any HubSpot deal property can go in it, including
a date property such as `closedate` or `hs_lastmodifieddate`. **It is not a dedicated date
range field.** Which property to filter on is a data-source decision, PRD §12 item 4, still
OPEN. The schema does not settle it.

**Which properties come back — a trap.** Controlled by `filters.properties`, a multiOptions
field. The node's own description warns: *"By default, the results will only include Deal ID
and will not include the values for any properties for your Deals."* So unless
`filters.properties` is explicitly filled in — `dealname`, `amount`, `dealstage`, `closedate`,
`pipeline` and so on — **the search returns bare IDs and nothing else.** The Coder must set
this explicitly.

**Page size.** `limit`, a number, default `100`, alongside `returnAll`, default `false`. The
schema states no maximum.

**Verdict: VERIFIED** for the operation, the filter mechanism, the empty-properties-by-default
behaviour, and `limit` / `returnAll`. Source: `n8n-nodes-base.hubspot` v2.2 catalogue schema,
read 2026-09-13.

**COULD NOT VERIFY:** HubSpot's real server-side cap on `limit` for deal search, and the real
rate limit. Neither is a node fact. Not proof the operation is enabled on the live instance —
the stripped-node warning still applies.
---

## Item 8 — Supabase rows, typeVersion 1

**The operation.** `resource: row`, `operation: getAll` (label "Get Many"). It is the only
row-read operation carrying a filter mechanism. `get` reads a single row by key.

**How a filter is expressed.** `filterType`, with options `none`, `manual` ("Build Manually",
the default) and `string` ("String"). In manual mode, `filters` is a fixedCollection at path
`filters.conditions` that builds condition rows against a column. In string mode,
`filterString` is sent raw, and the node's own notice points at PostgREST's documentation for
horizontal filtering.

So a date window is a condition on whatever timestamp column the usage table has — for example
`created_at.gte...` — built either through the manual condition rows or as a raw PostgREST
query string. **The node has no purpose-built date range field.**

**Row limit — a trap.** `limit`, a number, **default 50**, alongside `returnAll`, default
`false`. There is also `orderBy`, whose own description carries a real warning:
*"Recommended when using Return All or Limit ≥ 1000 to avoid duplicate or missing records."*
The schema states no maximum for `limit`.

A default of 50 is low enough that a silent truncation is a genuine risk on a usage table. The
Coder must set this deliberately and never leave it at the default.

**Verdict: VERIFIED** for the operation, the filter mechanism, the default limit of 50, and
the absence of a stated maximum. Source: `n8n-nodes-base.supabase` v1 catalogue schema, read
2026-09-13.

**COULD NOT VERIFY:** the actual server-side row cap on this Supabase project, which is a
project setting rather than a node fact, and whether `getAll` is reachable on this instance.
---

## Item 9 — Google Sheets append, typeVersion 4.7

**Required parameters for `append`** (`resource: sheet`, `operation: append`):

| Parameter | Type | Notes |
|---|---|---|
| `documentId` | resourceLocator | **required**, modes `list` / `url` / `id` |
| `sheetName` | resourceLocator | **required**, modes `list` / `url` / `id` / `name`, depends on `documentId.value` |
| `range` | string | **required**, default `"A:F"` |
| `columns` | resourceMapper | **required** |

**The columns mapping.** The schema confirms the field path `columns.mappingMode`, and one of
its values is `autoMapInputData`, seen directly in a `showWhen` condition alongside a sibling
`handlingExtraData` option set. By the resourceMapper convention used throughout n8n, the other
mode is `defineBelow` — every column mapped explicitly by name, which is what the architecture
specifies for node 20.

**Honest limit on this one.** The literal string `defineBelow` could not be pulled out of the
catalogue's flattened property search, because resourceMapper's internal mode list is not
exposed as a separate searchable property. Treat the mode's exact string as **consistent with,
but not character-for-character re-confirmed against, this schema.**

`options.handlingExtraData` governs unmapped input fields, with options `insertInNewColumn`
(default), `ignoreIt` and `error`. It matters if the Code node ever produces a field the sheet
has no column for.

**Verdict: VERIFIED** for `documentId`, `sheetName`, `range` and `columns`. **COULD NOT VERIFY**
the literal `defineBelow` string. Source: `n8n-nodes-base.googleSheets` v4.7 catalogue schema,
read 2026-09-13.

**On what a read returns from an empty sheet — nothing found.** The schema says nothing at all.
There is no property, description or example anywhere in the read operation's schema stating
the response shape for zero data rows: empty array, header-only row, or something else.

**This is stated plainly as unverifiable from documentation, and left as a Tester case rather
than guessed.** It decides node 9's three-way OK / Empty / Failed classification and must be
read from one real execution against the real snapshot, finance and support sheets before that
Code node's logic is trusted.
---

## Item 10 — `onError: continueRegularOutput` item shape

Not independently verified. This is n8n core execution behaviour, which no node schema
documents. **COULD NOT VERIFY** from documentation.

Needs a live throwaway test: force a node to fail with `continueRegularOutput` and
`alwaysOutputData: true`, then read the exact keys on the resulting item — whether it gains an
`error` key, and its shape. Blocking for node 9's failure detection.

---

## Item 11 — Merge, typeVersion 3.2

Confirmed from the schema, property by property.

- `mode: "chooseBranch"` — a real value on the `mode` options list.
- `chooseBranchMode` — shown only when `mode: chooseBranch`. Its **only** listed option is
  `waitForAll` ("Wait for All Inputs to Arrive"), and that is also the default. So
  `chooseBranchMode: waitForAll` is not merely valid, it is the sole choice in this mode.
- `output` — shown only when `mode: chooseBranch` **and** `chooseBranchMode: waitForAll`. Its
  options are exactly two: `specifiedInput` ("Data of Specified Input", the default) and
  **`empty` ("A Single, Empty Item")**. So `output: empty` is confirmed as a real,
  schema-listed option, and its own label states the architecture's exact claim: it emits a
  single, empty item.
- `numberInputs` was not returned by this narrow query, because it is a top-level node
  parameter rather than one nested under the `output` scope. Nothing in this check contradicts
  setting it to `5`.

**Verdict: VERIFIED** that `mode: chooseBranch`, `chooseBranchMode: waitForAll` and
`output: empty` are all real, correctly nested options that exist exactly as the architecture
describes. Source: `n8n-nodes-base.merge` catalogue schema, read 2026-09-13.

**What this cannot prove, stated plainly.** The schema documents what the node's interface
offers and what the option is labelled to do. It says nothing about runtime behaviour —
specifically, whether the node still fires and still emits exactly one empty item when one of
the five input branches arrives carrying an error-shaped item from an upstream
`continueRegularOutput` node, rather than a normal data item.

**That is a runtime question a schema cannot answer. COULD NOT VERIFY.** This is Lock 1 of the
single-OpenAI-call guarantee in architecture section 6, and it must be proven with a live test —
five static inputs, one of them deliberately error-shaped — before anyone trusts it in the full
workflow.
---

## Item 12 — Telegram message limit

Confirmed against Telegram's Bot API constants: `sendMessage` text is capped at 4,096 UTF-8
characters, and going over returns a hard `400 Bad Request: message is too long`. It does not
truncate.

**VERIFIED** via consistent third-party sources on a long-stable constant. The official page
returned only a truncated fetch, so one direct read of
`core.telegram.org/bots/api#sendmessage` is recommended as a follow-up.

---

## Item 13 — Workflow execution timeout versus a waiting execution

n8n's execution-timeout documentation page was not reachable at the URL tried. A live GitHub
issue, `n8n-io/n8n#15123` — "Wait node's waiting time exceeding the workflow timeout will
cause an infinite loop" — indicates the two settings interact badly and unpredictably, rather
than the timeout cleanly leaving waiting executions alone.

**Verdict: CONTRADICTED.** The risk looks real, not theoretical. This supports the Designer's
instinct in section 9 to avoid a workflow-level execution timeout given node 15 can wait up to
six hours.

**Recommendation.** Adopt section 9's option 1 — accept the gap and rely on retries — and set
no workflow-level timeout on this workflow at all.

---

## Blocking

These stop the Coder building the affected node correctly as designed, until resolved.

1. **Google Drive image embedding is unverified.** Node 13 can be built as designed, but it
   must be run with a real throwaway HTML-plus-image payload before node 14 downstream is
   trusted. If images do not embed, Route B replaces nodes 13 and 14 with three different
   nodes. That is a structural change, not a tweak.
2. **Gmail `sendAndWait` cannot attach the PDF.** Confirmed, not assumed. If Leo picks Gmail
   as the approval channel, node 15 carries the Google Doc link only, never the binary.
3. **The exact OpenAI model id is not settled**, and the pricing pulled today could not be
   independently confirmed. Before the Coder writes node 11, pick the model id from the
   credential's live From List search on the real account, and have a human read OpenAI's
   pricing page directly for that id.
4. **Google Sheets empty-read shape and the `continueRegularOutput` error-item shape are both
   unverified**, and node 9's entire OK / Empty / Failed classification depends on knowing both
   exactly. This needs one live test run per source before the Coder writes that logic.
5. **Merge v3.2's behaviour is unverified.** This is Lock 1 of the single-OpenAI-call
   guarantee, the most safety-critical claim in the design. Test it standalone first.
6. **The `sendAndWait` output on timeout is unverified.** Section 15 plans to tell rejection
   and timeout apart by comparing timestamps, which assumes `approved` is present either way.
   If the key is simply absent on timeout rather than `false`, node 16's IF expression needs a
   null-check the design does not currently have.

7. **HubSpot deal search returns bare IDs by default.** Unless `filters.properties` is filled
   in explicitly, every deal comes back as an ID with no values. The Coder must set it.
8. **The Supabase row limit defaults to 50.** Left at the default it will silently truncate a
   usage table. The Coder must set it deliberately.

All 13 items in section 11 are now checked. No Integrator work remains outstanding.

---

## Changes the Designer must make

1. **Sections 8 and 14 — drop the hedge on Gmail attachments.** Change "if it cannot, the
   approval message carries the Doc link" to a flat statement. Gmail `sendAndWait` never
   supports an attachment, confirmed from the node's own schema. It is not a fallback, it is
   the only possible behaviour.
2. **Section 6, Lock 1 — mark it "designed, not yet proven."** The write-up states Merge's
   single-item behaviour as settled fact. It could not be verified, so the section should
   treat it as design intent until a live test confirms it, matching how the rest of the
   document handles unverified assumptions.
3. **Section 10 — the OK / Empty / Failed classification needs a stated fallback** for the case
   where a live test shows the Sheets empty-read shape or the `continueRegularOutput` error-item
   shape does not match what node 9 expects. The design currently assumes both are known.
4. **Section 9 — timeouts.** Strengthen the recommendation against ever adding a workflow-level
   execution timeout, citing the interaction risk found in n8n's own issue tracker rather than
   just "not proven".

---

## Files read

`docs/architecture/v1-architecture.md`, `docs/PRD.md`, `docs/knowledge/n8n-environment.md`,
`docs/knowledge/n8n-gotchas.md`, `docs/knowledge/money-facts.md`, `docs/knowledge/data-sources.md`.
