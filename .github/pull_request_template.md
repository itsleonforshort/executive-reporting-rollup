**What this changes**

**Why**

---

**Checklist**

- [ ] No secret in any file, commit message or screenshot. I checked what I **staged**, not what I
      assumed was staged.
- [ ] No personal identifier added — instance URLs, emails, chat ids and document ids stay as
      angle-bracket placeholders like `<your-n8n-instance>`.
- [ ] I did not replace a `SET_ME_` placeholder with a plausible-looking value. They fail loudly on
      purpose.
- [ ] If I changed a documented finding in `docs/knowledge/`, I say below what I ran and what came
      back. Not what the documentation says.
- [ ] If I touched a node, it still follows `Original Node Name - What It Does`, and every
      parameter is set explicitly rather than left at an n8n default.
- [ ] I did not add a retry to a step that creates, sends or appends. Those are single-attempt on
      purpose.

**Evidence**

If this changes behaviour, say what you ran and paste the result. Redact first.
