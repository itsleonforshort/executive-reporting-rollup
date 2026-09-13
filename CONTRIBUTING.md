# Contributing

Thanks for looking. This is a working project rather than a maintained library, so contributions
are welcome but the bar is specific.

## The most useful contribution

**A correction to [`docs/knowledge/n8n-gotchas.md`](docs/knowledge/n8n-gotchas.md), backed by a
real execution.**

Every entry in that file was proven by running something on a real n8n instance and reading the
result, not by reading documentation. If one of them is wrong on your version of n8n, or is true
for a reason other than the one given, that is worth more than any feature.

**Say what you ran and what came back.** "This is wrong" is not actionable. "On n8n 1.x, node type
Y returned this item instead" is.

## Before you open a pull request

1. **No secrets, ever.** Not in a file, not in a commit message, not in a screenshot. Check what
   you staged, not what you think you staged.
2. **Redact personal identifiers.** Instance URLs, emails, chat ids and document ids are replaced
   with angle-bracket placeholders like `<your-n8n-instance>` throughout. Keep it that way.
3. **Do not replace a `SET_ME_` placeholder with a plausible value.** They are deliberate. Each one
   fails loudly at its own node rather than half-working, and that property is the point.
4. **Do not make a claim you have not verified.** This whole project is built on the rule that a
   number you did not fetch does not go in the brief. The same applies to the docs.

## Style

- Node names follow `Original Node Name - What It Does`. Both halves required.
- Set every parameter explicitly. Never rely on an n8n default.
- The Code node is a last resort. Try an expression, then Edit Fields, then Code.
- Never retry a non-idempotent step. Creating a document, sending an email and appending a row are
  all single-attempt on purpose.

## Reporting a problem

Open an issue. If it involves a secret being exposed, use a
[security advisory](../../security/advisories/new) instead — see [SECURITY.md](SECURITY.md).
