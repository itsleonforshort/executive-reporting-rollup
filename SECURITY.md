# Security Policy

## What this repo is

Documentation and a workflow definition. There is no running service here and nothing in this repo
executes on its own. The risk surface is what the workflow does **once you import it into your own
n8n instance and attach your own credentials.**

## Reporting a vulnerability

Please open a [security advisory](../../security/advisories/new) rather than a public issue, so the
problem is not disclosed before it can be looked at.

Expect a first reply within seven days. This is a personal project, not a funded product, so please
set your expectations accordingly.

## What counts as a vulnerability here

- A credential, key, token or other secret found anywhere in this repo or its history.
- A pattern in the workflow that would leak data from the importing user's own accounts.
- Anything in the documentation that would lead someone to expose a key by following it.

## What does not

- The `SET_ME_` placeholders. They are deliberate and they fail loudly on purpose.
- Credential **ids**. They are references and are useless without the instance and an account on it.
- The chart URLs. See the note below, which is a known and documented design trade-off.

## Known design trade-offs

**Chart data leaves in a URL.** The brief's two charts are rendered by
[QuickChart](https://quickchart.io), a free public service, and the chart data is JSON-encoded into
a `GET` URL embedded in the document. **Anyone holding that URL can re-render the chart and read
the numbers in it.** There is no authentication on that request.

This is documented rather than hidden because it is a real consideration for anyone putting
commercial figures through this workflow. If that matters for your data, render the charts
somewhere you control before importing.

## Secrets

No key, token or credential value appears in this repo. Keys live outside it, read as environment
variables. Personal identifiers are redacted throughout and replaced with angle-bracket
placeholders.

**If you fork this and add your own values, check your `.gitignore` before your first commit.**
`.mcp.json`, `.env` and anything holding a literal key must stay out of git.
