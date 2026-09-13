# Executive Reporting Rollup

An n8n workflow that replaces the weekly leadership deck an analyst builds by hand.

Every Monday at 07:00 it reads five data sources, works out what the numbers mean, has an AI model
write a short narrative, draws two charts, builds a Google Doc, exports a PDF, **asks a human to
approve it**, then emails leadership, posts a Telegram summary, and records the week in a snapshot
sheet.

25 nodes. One workflow. Read-only apart from a single appended row.

---

## The idea

Weekly reporting fails in a predictable way. Someone assembles the numbers by hand, the numbers are
right, and then that person is busy for three weeks and the report quietly stops. Automating it is
easy. Automating it *honestly* is the hard part, and that is what most of this repo is about.

Three rules drive nearly every decision in the build:

1. **Never print a number the workflow did not actually fetch.** No filler, no estimates, no
   plausible-looking placeholders.
2. **Never let one failed source produce a complete-looking brief.** A source that could not be
   read is named on the brief. A source that was read and held nothing is a real zero, and the two
   are never shown the same way.
3. **A reporting failure must never change a source system.** Everything is read-only except one
   write: a single row appended to a snapshot sheet, and only on a run that fully succeeded.

---

## How it works

```
Schedule Trigger - Weekly Monday 7am
  └─ Set - Define Reporting Week            the week boundary is decided once, here

     ├─ HubSpot - Fetch Deals And Pipeline          ─┐
     ├─ Supabase - Fetch Product Usage               │
     ├─ Google Sheets - Fetch Finance Actuals        ├─ five sources, in parallel
     ├─ Google Sheets - Fetch Support Log            │  each wired on BOTH its main
     └─ Google Sheets - Fetch Last Week Snapshot    ─┘  and its error output

  └─ Merge - Wait For All Five Sources
  └─ Code - Normalise And Calculate Metrics    classifies each source ok / empty / failed,
                                               then computes variance against target,
                                               week-over-week change, pipeline movement
                                               and revenue trend
  └─ IF - Enough Data To Build Brief?
       ├─ false → Telegram - Alert Run Blocked      says why, in plain words
       └─ true  → OpenAI - Write Executive Narrative
                  Code - Build Brief HTML And Chart URLs
                  HTTP Request - Create Google Doc From HTML
                  Google Drive - Export Brief As PDF
                  Telegram - Request Brief Approval        waits up to 6 hours for a human
                  IF - Was The Brief Approved?
                    ├─ false → Telegram - Alert Brief Not Sent
                    └─ true  → Gmail - Send Brief To Leadership
                               Telegram - Post Brief Summary
                               IF - All Sources OK For Snapshot?
                                 ├─ true  → Google Sheets - Append Weekly Snapshot Row
                                 └─ false → NoOp - End Run Without Snapshot

Error Trigger - Catch Unhandled Failure → Telegram - Alert Workflow Failed
```

---

## Design decisions

**A human approves before anything is sent.** The approval is a Telegram tap, not an email link.
Corporate mail gateways pre-fetch links in inbound mail, and a pre-fetched approval link is an
approval granted by a scanner. A Telegram tap cannot be faked that way.

**Each source feeds the Merge on two connections** — its main output and its error output — both
landing on the same input index. That guarantees exactly one item per source whether it succeeded
or failed, so the calculation step can always tell the three outcomes apart. `chooseBranch` /
`waitForAll` waits on inputs rather than on individual connections, which was measured rather than
assumed.

**The brief is built as one HTML string and uploaded to Drive for conversion**, rather than
assembled with the Google Docs node, which cannot insert an image at all. Drive's HTML-to-Doc
conversion *does* fetch remote `<img src>` URLs during conversion — also measured, and it is what
puts the charts in the document.

**Exactly one AI call per brief, never one per row.** The calculation node runs once for all items
and returns a single item, so the paid step downstream cannot fan out. This is the most important
cost property of the build and it holds structurally, not by luck.

**The paid node ships disabled.** Anyone importing this workflow has to enable the narrative step
deliberately. Nothing can spend money by accident on a first run.

**Non-idempotent steps never retry.** Creating a document, sending an email and appending a row are
each set to a single attempt. A retry after a timeout that actually succeeded would send the brief
to every leadership inbox twice.

**No workflow-level execution timeout, deliberately.** The approval step waits up to six hours, and
[n8n#15123](https://github.com/n8n-io/n8n/issues/15123) reports a Wait node outlasting a workflow
timeout can cause an infinite loop.

---

## Verified behaviour

Every item below was confirmed on a real execution against live accounts, not inferred from
configuration.

- All five sources classify correctly as read, empty, or failed — including the case where a source
  returns nothing because it genuinely had nothing, versus one that could not be reached.
- The five-input Merge fires and the calculation step emits exactly one item.
- Variance against target and week-over-week change compute correctly, checked against figures
  written down before the run rather than after.
- The Google Doc is created, both charts render inside it, and the PDF exports at the expected size.
- The Telegram approval request is delivered and the execution pauses for the human.
- A failed source is named on the brief rather than silently dropped.
- A blocked run sends an alert saying exactly why, and writes nothing.

---

## Repo layout

```
CLAUDE.md                      the rules the build follows
docs/PRD.md                    the contract
docs/architecture/             the design, and the source-classification ruling
docs/knowledge/                evidence behind every rule
docs/reports/                  integrator, coder, QA and tester reports
docs/state/                    project state and session handoffs
workflows/                     the workflow JSON
samples/                       sample source rows and sample briefs
```

### The part worth reading even if you never touch this workflow

**[`docs/knowledge/n8n-gotchas.md`](docs/knowledge/n8n-gotchas.md)** — every entry proven by running
on a real instance rather than taken from documentation:

- `n8n_update_partial_workflow` given a whole `parameters` object **deletes every key you did not
  mention**, and validation does not catch it, because the deleted keys all have defaults. Use
  dot-notation paths.
- A Telegram bot cannot message a person first. They must message it, or every send fails with
  `chat not found` — and if the alert node swallows errors, nobody is ever told.
- `alwaysOutputData: true` makes an empty source emit one blank item, not zero, so any code testing
  `items.length === 0` is dead code.
- A node with **no credential at all** does not behave like a node that failed. Do not redesign
  around a failure you only ever saw without credentials attached.
- An HTTP Request sending a raw body will not parse its response as JSON even when told to. Use
  `autodetect`.
- Reading a config tells you what it says. Running it tells you what it does. Three careful
  read-only review passes walked past a Schedule Trigger with no schedule on it; one execution
  found it in nineteen seconds.

---

## How it was built

Seven specialist agents, in a fixed order, defined in `.claude/agents/`:

| Agent | Job | Writes to n8n? |
|---|---|---|
| `designer` | Decides the architecture before anything is built | No |
| `integrator` | Verifies every external API against live provider docs | No |
| `coder` | Builds and updates. Follows the design, does not redesign it | **Yes, the only one** |
| `qa` | Audits the build and defines what "done" means | No |
| `tester` | Runs it with real data and with deliberate failures | Runs only |
| `king` | The only agent that may approve deployment | No |
| `orchestrator` | Owns state, order, retries and failure routing | No |

The rule that mattered most: **the agent that wrote a node is the worst judge of it.**

There is also a documented case of a reviewer being **wrong** and the builder being right, settled
by a third reading of the source rather than by seniority. It is written up in full at the end of
[`docs/reports/qa-v1.md`](docs/reports/qa-v1.md), because how it was settled matters more than who
turned out to be right.

---

## Setup

This is a specific workflow wired to specific accounts rather than a drop-in template. To run your
own copy:

1. **Import** `workflows/executive-reporting-rollup-v1.json` into a self-hosted n8n instance.
2. **Connect credentials** for Google Sheets, Google Drive, Gmail, Telegram and OpenAI, plus
   HubSpot and Supabase if you want those two sources. Credentials are created in the n8n web
   interface; nothing in this repo holds a key.
3. **Create three Google Sheets** — a weekly snapshot, finance actuals and a support log. Exact
   headers are in [`docs/knowledge/data-sources.md`](docs/knowledge/data-sources.md). Finance
   actuals is a tall sheet, one row per metric per week, so no metric name is hardcoded anywhere.
4. **Define your metrics** in [`docs/knowledge/metric-definitions.md`](docs/knowledge/metric-definitions.md)
   and add matching rows to the finance sheet. The repo ships with four sample metrics and twelve
   sample rows, every one marked `synthetic = TRUE` so they can never be mistaken for real figures.
   One metric must contain the word "revenue" for the trend chart to render.
5. **Message your Telegram bot once** before the first run. A bot cannot open a conversation, so
   until a human messages it, every send fails.
6. **Replace each `SET_ME_` placeholder** with your own values — the Supabase table and date column,
   and the leadership distribution list. These are deliberate: each one fails loudly at its own node
   rather than half-working with a plausible guess.
7. **Enable the narrative node** when you are ready for it to start costing money.

---

## No secrets in this repo

API keys live outside it, in the Claude Code settings file, read as environment variables.
`.mcp.json` references `${N8N_API_KEY}` and never holds a literal key, and it is gitignored anyway.
Credential **ids** appear in the docs; those are references, not secrets, and are useless without
the instance and an account on it.

**Personal identifiers are redacted throughout**, and replaced with angle-bracket placeholders such
as `<your-n8n-instance>` and `<owner-google-account>`. That covers both n8n instance URLs, the
account emails, and the Telegram chat id, which appears as `SET_ME_telegram_chat_id` in the
workflow JSON like every other placeholder. **They are redacted on purpose. Do not replace them
with real values in a tracked file** — supply your own at import time, in n8n.

---

## License

MIT. See [LICENSE](LICENSE).
