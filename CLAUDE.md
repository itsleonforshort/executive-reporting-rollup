# CLAUDE.md

The rules for this project. The proof behind each rule lives in `docs/knowledge/`.

## Be concise

**Short is the default. Every reply. No exceptions unless Leo asks for detail.**

- Say the one thing that matters. Then stop.
- If a reply can be three lines, make it three lines.
- Do not pad a reply to make it look complete.
- Do not explain a thing twice.
- Cut it before you send it. A first draft is usually twice as long as it needs to be.

## Number what needs Leo

**Anything Leo has to decide, answer, do or check goes in a numbered list.**

- One item per line. Never buried in a paragraph.
- Number them even when there are only two.
- Keep each item to one line. He should be able to count them at a glance.
- This does not break the one-question rule below. The numbered list shows what is
  outstanding. The single question at the end is the one thing to answer now.

## How to reply — read this first, every time

Leo is not a developer. He does not use a terminal.
**These are hard limits, not preferences. Check your reply against them before you send it.**

- Lead with the next action, or the decision he has to make.
- Keep it under 10 lines.
- One idea per line. A short list, not paragraphs.
- Short sentences. Simple words. No comma chains. Needs a second comma? Split it in two.
- Do not recap. He was there. No status summaries, no "where we left off".
- Report the result, not the journey. No tool names. No reasoning.
- One question at a time. Never end with a menu of open decisions.
- Never tell him to run a command. Run it yourself.
- For anything he does by hand, give numbered click-by-click steps in the n8n web interface.
- Never dump raw workflow JSON unless he asks. Describe what it does instead.
- Do not be flashy or clever. Plain and clear wins.
- Longer is allowed only when he asks for detail, or for numbered steps in n8n.

## Start here

1. This file, in full.
2. `docs/knowledge/README.md` — the index of the evidence files.
3. `docs/PRD.md` — the contract. Read its version line first.
4. `docs/state/project-state.md`, section 0 only.
5. Whatever architecture doc section 0 points you to, under `docs/architecture/`.
6. The newest file in `docs/state/handoffs/`, if one exists.

## Read before you act

Open the matching file **before** you touch that area. Do not work from memory.

| Doing this | Read first |
|---|---|
| Anything about what the brief contains | `docs/PRD.md` |
| Connecting or changing a data source | `docs/knowledge/data-sources.md` |
| Any number that appears in the brief | `docs/knowledge/metric-definitions.md` |
| Any paid call, model choice or cost estimate | `docs/knowledge/money-facts.md` |
| Your first n8n call this session | `docs/knowledge/n8n-environment.md` |
| A Code node or a cross-node expression | `docs/knowledge/n8n-gotchas.md` |
| Naming a node, or the ten build steps | `docs/knowledge/n8n-gotchas.md`, `n8n-environment.md` |

A file in that table may not exist yet. Create it the first time you learn something that
belongs in it. Do not invent one to fill a gap.

## What this project is

One n8n workflow. The **Executive Reporting Rollup**. Built for Leo.

It replaces the weekly deck an analyst builds by hand. About 20 nodes.

1. It runs every Monday at 07:00.
2. It reads five sources: HubSpot deals, Supabase usage, and three Google Sheets —
   finance actuals, the support log, and last week's snapshot.
3. It merges them, normalises them, then works out variance against target,
   week-over-week change, pipeline movement and revenue trend.
4. It sends the numbers to an AI model, which writes a short executive narrative.
5. It draws two charts: a pipeline bar chart and a revenue trend line.
6. It builds a Google Doc, exports a PDF, emails it to leadership, and posts a short
   summary to Telegram.
7. It appends this week's numbers to the snapshot sheet.

**The workflow is read-only apart from one write: the snapshot row.** A reporting failure
must never change a source system.

The brief is for executives. It is short, it is plain, and every number in it can be
traced back to the row it came from.

`docs/PRD.md` is the contract, version 1.0, written by Leo. King checks the build against
it, and scope does not grow without the PRD changing first. Its §12 lists what is still
open — a build may not start on an open item.

## Never do these

- Never print a number the workflow did not actually fetch. No filler. No estimate. No
  placeholder that looks like data.
- Never let the AI narrative invent a cause or a number. It explains only what the
  calculated metrics show.
- Never write to HubSpot or Supabase. The snapshot sheet is the only write in the build.
- Never let one failed source produce a complete-looking brief. Name the missing dataset
  on the brief itself.
- Never append a snapshot row for a run that failed. A bad row poisons next week's
  comparison.
- Never seed synthetic history into a sheet the real reporting reads, without saying so
  in the sheet itself. Mark every synthetic row.
- Never change a metric definition to make a number look better. Change it in
  `docs/knowledge/metric-definitions.md` first, with the reason.
- Never put a secret in a text field. Credentials only.
- Never build a node from memory. Call `get_node` first, every time.
- Never trust the node catalogue. A node listed here may still refuse to run here.
- Never skip the agent pipeline.
- Never edit another project's workflows on this n8n instance.
- Never use Claude's own Google connector to touch this project's data. Leo said so on
  2026-09-13. Only the credentials that live in his n8n count. Claude's connector is signed
  into a different Google account, so anything it creates lands in the wrong place.

## Money

- **There is no spending guard. It was removed on 2026-09-14 at Leo's request.** Nothing
  blocks a paid call now. `unlock spending` no longer does anything. See global `CLAUDE.md`.
- **So tell Leo before you spend anything, every time, and wait for a yes.**
- Never let an agent write a paid call. Agents read. You hand-write every paid body.
- Know the exact body before you send it. Read the echoed params back afterwards.
- Run the paid node disabled first. It costs nothing and catches real faults.
- Every paid call needs a reason. Log every paid call with its cost.
- Tell Leo before any run costing more than a few dollars. He decides.
- **Auto-refill is OFF.** Leo confirmed this on 2026-09-14, straight after the spending guard
  was removed. The credit balance is now the only hard stop on a runaway job. When it runs out,
  the call fails instead of buying more. Do not assume there is headroom.
- Only the Tester triggers paid calls, and only after QA passes.

The paid call here is the narrative step, PRD §4 step 10. It is cheap per run. The real
risk is a loop that calls it once per row instead of once per brief. It runs on **OpenAI**,
not Gemini. Leo decided that on 2026-09-13, and the Gemini credentials here are out of
quota. Read `docs/knowledge/money-facts.md` before you wire that node.

## Build rules

1. Set every parameter yourself. Never rely on an n8n default.
2. Validate, then verify. Read the `connections` object with your own eyes.
3. Search templates before building from scratch.
4. Build the error path at the same time as the happy path.
5. Use the agents. Never build and review your own work.

The ten steps, with the exact tool for each, are in `docs/knowledge/n8n-environment.md`.
Prefer `n8n_update_partial_workflow` over `n8n_update_full_workflow` when editing.

## Design defaults

- The Code node is a last resort. Try an expression, then Edit Fields, then Code.
- A Set node feeding 0–1 consumers is almost always wrong. Inline it at the consumer.
- Per-item iteration is automatic. Do not add a Loop Over Items node to make it loop.
- Prefer a named node reference over `$json` in workflows with branches.
- Webhook payloads live under `$json.body`, not `$json`.
- Do date math inline with Luxon, not with a DateTime node.
- One source fetch per branch. Merge the branches once, at the end.
- Every fetch gets a timeout and a retry. A slow API must not stall the whole run.
- The week boundary is decided in one place, then passed down. Never recomputed per node.

## Node naming

Keep the original node name. Add a dash. Then say what the node does.

```
Schedule Trigger - Weekly Monday 7am
HTTP Request - Fetch Sales Pipeline
IF - Any Source Failed?
Code - Compare To Last Week
Gmail - Send Approval Request
```

Both halves are required. The Coder enforces this on every node in every workflow here.

## The agent pipeline

Eight agents live in `.claude/agents/`. Invoke them by name with the Agent tool.
Seven build. `humanizer` sits outside the pipeline and rewrites text only.

Use them every time. Skipping the pipeline is the most repeated mistake across Leo's
projects. The agent that wrote a node is the worst judge of it.

Order: Designer → Integrator → Coder → QA → Tester → King. The Orchestrator runs all of it.

| Agent | Job | Writes to n8n? |
|---|---|---|
| `designer` | Decides the architecture before anything is built | No |
| `integrator` | Verifies every external API against live provider docs | No |
| `coder` | Builds and updates. Follows the design, does not redesign it | **Yes, the only one** |
| `qa` | Audits the build and defines what "done" means | No |
| `tester` | Runs it with real data and with deliberate failures | Runs only |
| `king` | The only agent that may approve. Returns `APPROVED_FOR_DEPLOYMENT: true/false` | No |
| `orchestrator` | Owns state, order, retries and failure routing | No |

- A stage may not start until the one before it reports complete.
- QA defines done, not the builder. Every QA report ends in numbered criteria.
- Failures route backwards to the owning agent, then run forward again.
- King's default answer is false. A missing report is a failure, not a neutral absence.
- Three failures at the same stage stops the pipeline and escalates to Leo.

## Where things are

- `docs/PRD.md` — the contract. `docs/knowledge/` — the evidence behind this file.
- `docs/architecture/` — Designer output. `docs/reports/` — agent reports.
  `docs/state/project-state.md` — Orchestrator state. `docs/state/handoffs/` — handoffs.
- `workflows/` — local JSON copies. `samples/` — sample source data and sample briefs.
- Two n8n instances exist. Target the self-hosted one. Ask Leo before touching the cloud
  one. Both URLs are in `docs/knowledge/n8n-environment.md`.
- Session wrap-up triggers, and what to do when one fires: `C:\Users\AMD\.claude\CLAUDE.md`.
