# Money — the habits, and what this project can spend

Read this before any paid call, any cost estimate, or any test run that calls an AI model.

This project has no expensive step yet. The habits below were learned the expensive way on
another project, on this same machine. They carry over.

## What can cost money here

| Step | Paid? | Notes |
|---|---|---|
| HubSpot, Supabase, Google Sheets reads | Usually not | Inside plans Leo already has. **Confirm per source.** |
| The narrative step, PRD §4 step 10 | **Yes** | OpenAI. One model call per brief. Small. |
| Charts, Google Docs, PDF, email, Telegram | No | |
| A test run | **Yes, if it reaches the narrative step** | Disable that node while testing anything else. |

The real risk here is not the price of one call. It is a loop that calls the model once
per row instead of once per brief. Check the node's input count before you enable it.

## The narrative runs on OpenAI. Decided 2026-09-13.

Leo chose OpenAI for PRD step 10 on 2026-09-13. The credential **"OpenAI account"**
(`DBmj9DTgWeT0tOGw`, type `openAiApi`) already exists on the self-hosted instance.

- One call per brief. Never one per row. Check the node’s input count before enabling it.
- The exact model id and request body are not settled yet. The Integrator confirms both
  against live OpenAI docs before the Coder builds the node.
- OpenAI is billed per token, separately from any seat Leo owns. **Auto-refill is OFF** since
  2026-09-14 — see the update at the end of this file.
- Cost is not yet measured on this project. Log the first real call with its cost.
- **Rough estimate only, not confirmed.** At around 2,000 input and 400 output tokens, one
  narrative call looks like **well under one cent**. The Integrator could not confirm OpenAI
  pricing on 2026-09-13: the page fetch came back summarised, and a web search returned
  model names that could not be matched to any real OpenAI model from a trusted source.
  **Treat every published price here as unconfirmed** until a human opens OpenAI'''s pricing
  page and reads the row for the exact model id chosen. Full detail in
  `docs/reports/integrator-v1.md`.

## Why not Gemini

PRD v1.0 named Gemini for the narrative. PRD v1.1 no longer does. On 2026-09-07 both Gemini credentials on this same
n8n instance answered a **single** call with:

```
{"error":"The service is receiving too many requests from you"}
```

- `cARcyYfBRC78974E` — named "Google Gemini - Funded Project", despite the name
- `BxZmKFDPOmxKNsih`

That is Google's 429 for an exhausted free quota with no card paying. It is not a burst
problem, and retrying does not help. Proven on executions 474, 475 and 476.

That is why step 10 moved to OpenAI. See the section above.

**Do not re-test Gemini unless Leo says he has funded that Google Cloud project.** Two
dead ends were walked on another project before this was found.

## The habits

- **NEVER let an AI agent make a paid call.** Agents read. A human hand-writes every paid
  body. On another project an agent was told in plain words to submit nothing, and it
  submitted real jobs three times.
- **Run the paid node DISABLED first.** The single most valuable habit found on this
  machine. It costs nothing and it has caught real faults.
- **Know the exact body before you send it.** Read the echoed parameters back afterwards.
- **Every paid call needs a reason.** Do not re-run one to "check something".
- **Log every paid call** with its estimated cost.
- **Tell Leo before a run that will cost more than a few dollars.** He decides.

## The guard

- A `PreToolUse` hook blocks paid calls. Leo types `unlock spending` to allow one. It is
  good for one call, or one hour, whichever comes first.
- The rules for it live in the global `C:\Users\AMD\.claude\CLAUDE.md`.
- **It fails closed.** An unexpected block means the guard needs fixing. Say so. Do not
  work around it.
- **A "cost preview" flag is not a spending guard.** One was ignored twice, and the
  server charged anyway.

## Billing state

- **AUTO-REFILL IS OFF** since 2026-09-14. It was ON before that date, and older notes in this
  file say so. An empty balance now stops a runaway job instead of topping itself up.
- A Pro subscription never pays for the API. Proven twice, on two different providers.
  If a workflow calls an API, that API is billed separately from any seat Leo owns.

---

## Update — 2026-09-14. The guard is gone and auto-refill is off.

Two changes on the same day, both Leo's decision, and they pull in opposite directions. Read them
together or you will misjudge the risk.

**1. The spending guard was removed.** Leo's words: *"I JUST WANT THAT GUARD TOTALLY REMOVED."*
He typed `unlock guard` and both hook entries were deleted from
`C:\Users\AMD\.claude\settings.json`. **Nothing blocks a paid call now.** `unlock spending` is an
ordinary phrase and does nothing. No paid call is logged anywhere unless you write the entry
yourself. Backups sit in `C:\Users\AMD\.claude\` if it is ever wanted back.

**2. Auto-refill was switched off**, confirmed by Leo in the same exchange. This was the right
move and it partly offsets the first. Recorded on his say-so — nobody here can see his OpenAI
billing page, so it is his statement, not a verified reading.

**What this means in practice.** The soft stop is gone and the hard stop is now the credit
balance. A runaway job is no longer prevented, and it is no longer funded past the balance
either — it will spend whatever is there and then fail. **The balance is not a safety net. It
stops the bleeding only after the money is already spent.**

**So the rule that matters is the one that was always true, and is now the only one:** tell Leo
what a call costs and wait for a yes, every time. Never let a subagent make a paid call. Run a
paid node disabled first. Watch for a call inside a per-row loop, which is how $6 of junk got
submitted in the meme-to-video project and the reason the guard existed at all.

**Still true and unchanged:** no money has ever been spent on this project. Zero. Node 11
`OpenAI - Write Executive Narrative` is still `disabled: true` and has never run.
