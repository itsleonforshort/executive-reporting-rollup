# The data sources — where every number comes from

Read this before connecting, changing, or debugging any source.

**Status: empty on purpose. Nothing is connected yet.** The five systems are named in
`docs/PRD.md` §3. What they contain, and how they are read, is not yet proven.

Fill one block below the first time a source is actually read. Not before. A source that
has never returned a row does not get an entry here.

## The block to fill, per source

Copy this and fill it in.

```
### <Area> — <System name>

- **Holds:** what this system actually knows
- **Read by:** the n8n node or endpoint used
- **Auth:** which credential, created in the n8n web interface by Leo
- **Date field:** the field the weekly window filters on
- **History:** how far back it can be queried
- **Page size / limit:** the real one, read off a response
- **Rate limit:** the real one, and what a 429 looks like
- **Empty week:** what it returns when there is no data. An empty array, a 404, a zero
- **Proven on:** date, and the execution that proved it
```

## The five sources

| Area | System | What is still missing | Status |
|---|---|---|---|
| Sales | HubSpot | Which pipeline. Which stages count as open pipeline. The credential | **OPEN** |
| Usage | Supabase | Project URL. Table or view. Which fields are the metrics. The credential | **OPEN** |
| Finance | Google Sheets | File id, tab name, column headers | **OPEN** |
| Support | Google Sheets | File id, tab name, column headers | **OPEN** |
| Last week | Google Sheets — weekly snapshot | File id, tab name, and the column order the workflow appends to | **OPEN** |

The snapshot sheet is both a source and the workflow's only write. Its column order is a
contract. Once rows exist, columns may not be reordered or renamed without a migration.

## Rules for this file

- **Paste real responses. Do not describe them from memory.**
- A field name written here must have been seen in a real payload.
- If a source changes shape, update the block the same day and note the date.

## What is actually connected — checked 2026-09-13

Read from the live n8n instance with `n8n_manage_credentials`. 18 credentials exist. Only
five of them matter to this project.

**Connected and ready**

| Credential | Id | Type |
|---|---|---|
| Google Sheets account | `iFXqcyvwh4DYkt3s` | `googleSheetsOAuth2Api` |
| Google Drive account | `iH3ximBTa6XjheVQ` | `googleDriveOAuth2Api` |
| Gmail account | `cFvvHW2uwDptkQvf` | `gmailOAuth2` |
| OpenAI account | `DBmj9DTgWeT0tOGw` | `openAiApi` |
| Executive Rollup Bot Telegram account | `xkNXmRcwoz0Htiqy` | `telegramApi` |

**The Telegram credential was created by Leo on 2026-09-13 at 15:58 UTC**, owned by
`<owner-google-account>`, the correct account. Confirmed live with `n8n_manage_credentials`.
It closes the credential half of QA BLOCKER 4.

**The chat id is a separate thing and is still missing.** Nodes 15, 18, 21, 22 and 25 carry
`chatId: "SET_ME_telegram_chat_id"`. A credential proves the bot exists; it does not say who the
bot should message. The bot also cannot message Leo until Leo has sent it a message first —
Telegram blocks bot-initiated conversations. Both must be true before any Telegram node works.

**Not connected at all**

- **HubSpot.** No credential of any kind. PRD source 1 cannot be read.
- **Supabase.** No credential of any kind. PRD source 2 cannot be read.

Leo has to create these two inside the n8n web interface. Claude never handles the keys.
Both are PARKED by Leo's own decision — the nodes are built and left without credentials
on purpose.

## The three sheets — remade under Leo's own account, 2026-09-13

**The first set is dead. Do not use those ids.** Claude created three sheets and a folder
through the claude.ai Google Drive connector, which is signed into
`<a-different-google-account>`. Leo said he does not want another account involved in this
project at all. He copied the three sheets into `<owner-google-account>`, and the originals
plus their folder were moved to trash on 2026-09-13.

**Dead ids, kept only so nobody wires them by mistake:**
`1iI2Mnr7AZ3qpk0PZIVXiMQE7DijPvuhCYoq20QF_b3Q`,
`1o8ycaEYENc6wNBOYk2Pk2BBn2HSv5TaGPoDVSiJRtS8`,
`1tPas6mmDAEhUSwxuuPVLs7lnr4245sdOqLoxEvpCDYs`,
folder `1iQPCl3nhPfO3F5GhIw0Xlrld4nDGq1A_`.

**The live ids. Supplied by Leo on 2026-09-13. These are the only ones to use.**

| Sheet | File id | Tab gid |
|---|---|---|
| Weekly Snapshot | `1TdTEZ1Jh-4Ne62LmwFfza25pYSttembIuji1o--VElA` | `1425370562` |
| Finance Actuals | `1MJg-FLrykkAAtSDLdCLCuChgVDxI7db9K0pJAetQyhE` | `2085318769` |
| Support Log | `13w0EhJU5taXmmtJ0CUXIqN8P8nfQC489oyXaTsf0MHk` | `1845818240` |

All three are owned by `<owner-google-account>`, the same account the n8n Google Sheets
credential is signed into. Nothing else has access.

The gid is the tab id. n8n's `sheetName` resource locator accepts it directly, which is safer
than matching a tab by name, because a renamed tab would break a name match silently.

**Header rows — these carried over into the copies.**

- Weekly Snapshot: `week_label, week_start, week_end, run_at, synthetic, notes`
- Finance Actuals: `week_label, metric, actual, target, synthetic, notes`
- Support Log: `week_label, ticket_id, opened_at, resolved_at, category, status, synthetic, notes`

**The snapshot sheet has no metric columns on purpose.** No metric is defined yet —
`metric-definitions.md` is deliberately empty. A metric column cannot exist before the metric
has a definition block.

**Finance Actuals uses a tall shape** — one row per metric per week — rather than one column
per metric. Chosen so the sheet names no metric before the metric list exists, and so adding a
metric later needs no column change.

The `synthetic` column is on all three because `CLAUDE.md` requires every seeded row to be
marked in the sheet itself.

## The Google account rule — settled 2026-09-13

**Only the credentials in Leo's n8n are used. Nothing else.** Claude's own Google Drive
connector is signed into a different Google account and must not touch this project's data
again.

The n8n “Google Sheets account” credential is signed into `<owner-google-account>`,
confirmed by Leo. Every sheet this workflow reads or writes lives in that account.

---

## Update — 2026-09-14. HubSpot and Supabase are no longer parked.

Leo added both credentials himself. Both verified live with `n8n_manage_credentials`.

| Purpose | Name | Id | Type |
|---|---|---|---|
| HubSpot node (node 3) | HubSpot Service Key account | `ZhEi0614V5vjYKFJ` | `hubspotAppToken` |
| Supabase node (node 4) | Supabase account - <supabase-account> | `34bKthNT33Ghzp2t` | `supabaseApi` |

**All five data sources now have a credential.** That reverses the standing decision recorded as
"HubSpot and Supabase are PARKED" — Leo's earlier words were *"For supabase and hubspot do not add
accounts yet."* On 2026-09-14 he said to add both. PRD §12 items 3 and 4 move from PARKED back to
OPEN and now need real answers.

### HubSpot

Leo used a **Service Key**, which is why the credential is named that way. The type is
`hubspotAppToken`, which is what node 3's `authentication: "appToken"` expects. The two match.

### Supabase — two things worth knowing

**1. The Supabase account is a different email.** The credential is named
`Supabase account - <supabase-account>`, not `<owner-google-account>`. That is fine and it is
Leo's own choice — the "only Leo's own account" rule in `CLAUDE.md` is about **Google**, because
Claude's own Google connector was signed into a third account and put files in the wrong place.
Supabase has no such trap. Recorded so nobody later reads it as a mistake.

**2. The Host must NOT include the REST path.** Leo's first attempt failed with
*"Couldn't connect with these settings"* because he pasted what Supabase's Data API page shows:

```
https://rnresihcaoeonuuybrex.supabase.co/rest/v1/     <- WRONG, fails to connect
https://rnresihcaoeonuuybrex.supabase.co              <- correct
```

**The n8n Supabase node appends `/rest/v1/` itself.** Give the credential the bare project URL.

**Where to find the two values in the current Supabase interface**, which has moved:
- **Project URL** — Project Settings, then **Data API** under INTEGRATIONS. Not on the API Keys page.
- **Key** — Project Settings, then **API Keys**, then the second tab,
  **Legacy anon, service_role API keys**. The first tab now shows the newer
  `sb_publishable_` and `sb_secret_` keys instead.

### Still needed before node 4 can run

Node 4 still carries two placeholders that only Leo can answer:

- `SET_ME_supabase_table_name` — which table or view holds the product usage data.
- `SET_ME_supabase_date_column` — which column in it holds the date, used for the week filter
  and for `orderBy`.

Node 3's date filter is still the functional placeholder on `hs_lastmodifieddate`. It is shaped
correctly but it is not the real business filter — PRD §12 item 4 decides which pipeline and which
stages count, and that is still unanswered.
