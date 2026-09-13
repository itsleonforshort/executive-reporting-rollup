# Project State — Executive Reporting Rollup

**Last updated:** 2026-09-13
**Maintained by:** the Orchestrator. Any agent may read this; only the Orchestrator writes it.

---

## 0. CURRENT INITIATIVE (2026-09-13) — set up the project, then design the build

### Where it stands

| Check | Value |
|---|---|
| PRD | **v1.3, written by Leo, 2026-09-13.** `docs/PRD.md` |
| n8n workflow | **`iIypy4KWtaqsowAQ`** — Executive Reporting Rollup - Weekly Leadership Brief. 25 nodes. **Inactive, never run** |
| Target instance | self-hosted, `https://<your-n8n-instance>` |
| Credentials connected | Google Sheets, Google Drive, Gmail, OpenAI. **No HubSpot, no Supabase, no Telegram** |
| Money spent | **none.** The OpenAI node is built disabled |

### What exists in this folder

The workbench is set up. `CLAUDE.md`, the PRD, five knowledge files, the eight agents,
the n8n MCP config and the skills plugin. Copied from the Advance Social Media Generation
project where the content was general, written fresh where it was not.

### Pipeline stages — this cycle

| Stage | State | Output |
|---|---|---|
| Designer | **complete** 2026-09-13 | `docs/architecture/v1-architecture.md` — 25 nodes |
| Integrator | **complete** 2026-09-13. All 13 items checked | `docs/reports/integrator-v1.md` |
| Coder | **rework complete** 2026-09-13. 3 blockers + 6 more fixed. MAJOR 1 refused and upheld | `docs/reports/coder-v1.md` §13 |
| QA | **re-audit needed.** First pass found 4 blockers; 3 fixed, 1 is Leo's. MAJOR 1 withdrawn | `docs/reports/qa-v1.md` — 34 criteria, criterion 4 amended |
| Tester | **blocked.** QA says not fit to hand over until the blockers are fixed | — |
| King | not started | — |

### Blocking the Designer

`docs/PRD.md` §12 open items. Three left that change the architecture:

1. **No native chart node in n8n.** Steps 12 and 14 need a rendering route verified by the
   Integrator before the Designer can place nodes.
2. **The approval channel.** Approval itself is settled. Who approves, and whether it
   arrives by email or Telegram, still decides the node.
3. **The reject path.** What happens on a rejection, and on no answer at all. Whether a
   snapshot row is still appended.

### Settled

- **2026-09-13. The narrative model is OpenAI, not Gemini.** Leo decided. PRD is now v1.1.
  Credential "OpenAI account" (`DBmj9DTgWeT0tOGw`, type `openAiApi`) on the self-hosted
  instance. The exact model id is still to be confirmed by the Integrator.
- **2026-09-13. A human approves the brief before it is sent.** Leo decided. PRD is now
  v1.2. This is the new step 15. The old steps 15, 16 and 17 shifted down by one, so the
  snapshot write is now step 18.

### Next action

Leo settled three things on 2026-09-13. He approves the brief himself. HubSpot and Supabase
are parked, with no accounts to be added yet. The three Google Sheets were created.

**Two sources are now dead for v1.** With HubSpot and Supabase parked, the workflow can only
read the three Google Sheets. PRD §4 steps 3 and 4 cannot be built. The Designer has to be
sent back to reshape the build around three sources instead of five, or the build waits.

**Blocking, in order**

1. **The Google account mismatch.** The three sheets sit in `<a-different-google-account>`.
   The n8n credential owner shows `<owner-google-account>`. If those differ, n8n cannot see
   the sheets. Leo must read the account off the credential in the n8n web interface.
2. **The metric list.** `metric-definitions.md` is empty by design. No metric column can be
   added to the snapshot sheet until metrics are defined. PRD §12 item 5.
3. **The approval channel.** Leo approves, but there is no Telegram credential. Gmail works
   today. Telegram was the Designer’s recommendation, on the mail-scanner risk.
4. **Whether to reshape for three sources**, or hold the build until HubSpot and Supabase
   come back.

---

## 1. History

| Date | Event |
|---|---|
| 2026-09-13 | Folder created. Workbench set up. Leo supplied PRD v1.0. |
| 2026-09-13 | Leo chose OpenAI over Gemini for the narrative, step 10. PRD raised to v1.1. |
| 2026-09-13 | Leo required human approval before sending. Added as step 15. PRD raised to v1.2. |
| 2026-09-13 | Designer produced architecture v1. 25 nodes. |
| 2026-09-13 | Integrator first pass. Gmail sendAndWait cannot attach a PDF. Six blocking items found. |
