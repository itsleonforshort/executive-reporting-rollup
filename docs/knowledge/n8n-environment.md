# The n8n environment — instances, keys, tools, folder

Read this before your first n8n call in a session, and any time a key or a node type fails.

The instance facts, the key facts and the ten build steps were copied from the Advance
Social Media Generation project on 2026-09-13, word for word. They were proven there, on
this same n8n instance. Only the folder layout and the related-projects table are new.

## This instance strips out some nodes

- **This n8n instance strips out some nodes, and the catalogue lies about it.** `get_node`
  happily returns `nodes-base.executeCommand`, but n8n refuses to run it: *"Unrecognized
  node type"*. **A node existing in the catalogue is not evidence it works here.** Prove it
  on a throwaway workflow first. **But the stripping is selective, not general:**
  `Edit Image` works and the LangChain **MCP Client Tool node runs fine**. That one was
  assumed dead and was not — so do not assume in either direction.

## The two tools you have

**1. n8n-mcp MCP server** (`npx n8n-mcp`) — https://github.com/czlonkowski/n8n-mcp

This is your *data* layer. It knows all 800+ n8n nodes, their real parameter schemas,
thousands of community templates, and it can talk to the live n8n instance to create,
update, validate, test, and activate workflows.

**2. n8n-mcp-skills plugin** (15 skills) — https://github.com/czlonkowski/n8n-skills

This is your *judgement* layer. It teaches how to use the MCP tools well and how to
avoid the mistakes that produce workflows which validate cleanly and then break in
production.

The router skill `using-n8n-mcp-skills` loads automatically at session start and points
you at the right specialist skill. **The skills are the authority.** If a skill
contradicts your training data, trust the skill. If a live MCP tool contradicts a skill,
trust the tool and tell the user the pack may need updating.

## The build process

Follow this order. Do not skip steps.

1. **Understand** — Restate what the user wants in one or two plain sentences and confirm
   it before spending tokens. Ask about trigger type, data sources, and destination if
   they are unclear.
2. **Search templates** — `search_templates` by keyword, task, and node. Show the user
   anything close.
3. **Research nodes** — `search_nodes` to find candidates, then `get_node` for the real
   schema of each one you plan to use.
4. **Design** — Describe the node-by-node plan in plain language and **get the user's
   approval before building.** They should never be surprised by what appears in n8n.
5. **Validate the pieces** — `validate_node` on each configured node before assembling.
6. **Build** — Create the workflow JSON. Wire connections deliberately. Add error paths.
7. **Validate the whole** — `validate_workflow`, fix everything it reports, re-run until
   clean. `n8n_autofix_workflow` can handle the routine fixes.
8. **Deploy** — `n8n_create_workflow`, then `n8n_validate_workflow` by ID, then
   `n8n_get_workflow` to eyeball the connections.
9. **Test** — `n8n_test_workflow` where it is safe to do so. Check `n8n_executions`.
10. **Hand over** — Tell the user in plain words what was built, what it does, what they
    still need to do (usually: connect credentials, then turn it on).

Prefer `n8n_update_partial_workflow` over `n8n_update_full_workflow` when editing an
existing workflow. It is safer and it is what the tooling is optimised for.

## Credentials and secrets

Credentials for the services the workflow calls are created and connected by the user
**inside the n8n web interface**. Claude never asks for, receives, or stores a password or
API key in this project.

**The n8n instance API key is not in this folder.** It lives once, in
`C:\Users\AMD\.claude\settings.json` under `env.N8N_API_KEY`. This project's `.mcp.json`
refers to it as `"N8N_API_KEY": "${N8N_API_KEY}"` and must never hold the literal key.

- The key is read only when Claude Code starts. Changing it needs a restart.
- All of Leo's n8n projects use this same variable.
- If `n8n_health_check` fails with a `401`, the variable is missing or wrong — do not paste
  a key back into `.mcp.json` to fix it.

## The instance

**The real URLs are redacted in this repo because it is public.** They live in `.mcp.json`, which
is gitignored and stays on the machine. Read `N8N_API_URL` there for the live value. Do not paste a
real instance URL back into any tracked file.


Leo has **two** n8n instances. This project targets the self-hosted one.

- **Target (self-hosted):** `https://<your-n8n-instance>` — configured in
  `.mcp.json`. This is where every workflow gets built and deployed.
- **Also exists (n8n Cloud):** `https://<your-n8n-cloud-instance>` — NOT connected. Its
  API key is separate. If Leo ever asks for a workflow "in the cloud one", stop and
  confirm which instance he means before building.

This instance is shared with several other projects. Name things clearly, and never edit
or delete a workflow that belongs to another project.

## The two key screens

n8n has **two** key screens. They are not interchangeable — each does a job the other
cannot.

| Screen | Makes | Env var | What it is for |
|---|---|---|---|
| Settings → **n8n API** | audience `public-api` | `N8N_API_KEY` | Reading, creating and updating workflows. **The Public API rejects the other key with `401`** |
| Settings → **MCP** | audience `mcp-server-api` | `N8N_MCP_ACCESS_TOKEN` | **Running** workflows |

Both live in `C:\Users\AMD\.claude\settings.json` under `env`, and both are read only at
Claude Code startup.

**Why the MCP token is needed.** A **Manual Trigger cannot be fired through the Public API.**
Without the MCP token, any workflow whose only trigger is a manual one has to be run by Leo
clicking *Test workflow* in the browser. With it, workflows can be run from here.

**One extra step per workflow.** The MCP token is not enough on its own — each workflow must
also be marked **"Available in MCP"**. n8n exposes nothing by default. If a run fails with
`WORKFLOW_NOT_EXPOSED`, that is the cause. **Ask Leo before enabling it on any workflow:** it
is a visible, persistent setting, and it makes that workflow runnable by an AI client.

## The folder

There is no application source code here. The repo is a workbench:

- `.mcp.json` — the **n8n-mcp** MCP server (project scope)
- `.claude/settings.json` — the **n8n-mcp-skills** plugin (project scope)
- `.claude/agents/` — the eight agents
- `.gitignore` — keeps secrets out of git
- `CLAUDE.md` — the rules
- `docs/PRD.md` — the contract
- `docs/architecture/` — Designer output, versioned
- `docs/reports/` — agent reports
- `docs/state/project-state.md` — Orchestrator project state
- `docs/state/handoffs/` — session handoffs
- `docs/knowledge/` — this folder. The evidence behind the rules in `CLAUDE.md`
- `workflows/` — local JSON copies of workflows we build
- `samples/` — sample source rows, and sample briefs kept as evidence

## Related projects on this machine

| Folder | What it is | Relationship |
|---|---|---|
| `Advance Social Media Generation` | Image-to-reel automation | Unrelated work. **The source of the n8n facts in this file and in `n8n-gotchas.md`** |
| `automatic-meme-project` | The original meme-to-video system | Unrelated. Read-only |
| `MagentIQ` | FTSE 100 weather report automation | Unrelated. Shares the n8n instance only |
| `n8n-project-2` | Uses the same n8n instance | Unrelated |

## Setup reference

Set up on 2026-09-13, by copying the working config from the Advance Social Media
Generation project. Recorded here in case it ever needs repairing.

MCP server (project scope) — no key in the file, it reads the environment variable:

```bash
claude mcp add n8n-mcp --scope project \
  -e MCP_MODE=stdio -e LOG_LEVEL=error -e DISABLE_CONSOLE_OUTPUT=true \
  -e N8N_API_URL=<instance-url> -e 'N8N_API_KEY=${N8N_API_KEY}' \
  -- npx -y n8n-mcp
```

Skills plugin (project scope):

```bash
claude plugin install n8n-mcp-skills@n8n-mcp-skills --scope project -y
```

Health check: call `n8n_health_check`. If it fails, check the environment variable first,
then the API URL. Update the plugin with `claude plugin update n8n-mcp-skills`.
