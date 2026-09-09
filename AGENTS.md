# AGENTS.md

An opencode project that acts as an **n8n assistant**: it loads n8n's official
skills and connects to a locally-hosted n8n instance's **built-in MCP server**
so opencode can search, build, edit, and execute workflows directly.

## setup.md (portable install runbook)

`setup.md` is a portable guide for standing up this project on a fresh machine
with any CLI coding agent. If asked to set this up on another computer, read
`setup.md` and follow its decision questions (Docker vs npm n8n, base URL,
MCP token method) before making changes. The project defaults to **Docker**
hosting via `docker-compose.yml`.

## Config is project-only

All opencode/n8n configuration lives in this repo — never place anything in
global (~/.config/opencode) scope. The active config file is `opencode.json` at
the repo root.

## Skills (via the n8n-skills plugin)

The n8n skills come from the **official n8n-io/skills** repo, **vendored in-tree**
at `<project>/n8n-skills/` (committed directly, no nested `.git`). `opencode.json`
loads its plugin (`"plugin": ["./n8n-skills/opencode/plugin.ts"]`), which:

- auto-registers all 14 skills from `n8n-skills/skills/` (13 capability skills +
  the `using-n8n-skills-official` meta-skill) — no manual `skills.paths`.
- injects `using-n8n-skills-official` into the system prompt every session.
- fires bash hooks after n8n MCP tool calls (requires `bash` + `jq` on PATH).

Update the vendored skills by re-vendoring from upstream (see setup.md Step 1),
then **restart opencode** — do **not** run `git -C n8n-skills pull` (it resolves
to this repo's `.git`, not the skills upstream). When working
on workflows/nodes/expressions, load `using-n8n-skills-official` first and
follow its routing into the matching capability skill. Follow the skills'
build best practices (validate nodes → build → validate workflow → deploy).

The **n8n MCP server must be named `n8n`** (as in `opencode.json`) or the
plugin's tool-after hooks won't fire (it matches `n8n` in the tool name).

## jq requirement

The plugin's bash hooks need `jq` on PATH (git bash). Installed via
`winget install jqlang.jq`. It lives in the WinGet packages dir which is on
the user PATH; the hooks won't resolve it in a shell started before that
install. If hook reminders stop appearing, verify `jq --version` in bash.

## n8n MCP

Connected in `opencode.json` as a **remote** server:

- URL: `http://localhost:5678/mcp-server/http` (n8n self-hosted default port)
- Auth: `Authorization: Bearer {file:.n8n-mcp-token}` — opencode reads the bare
  token from the **git-ignored** project file `.n8n-mcp-token`. Set
  `"oauth": false` (n8n uses an API-key header, not OAuth — without it opencode's
  OAuth auto-detection fights the header and shows a misleading "needs auth").
  Do not hardcode the token in config or commit it.

### n8n-side prerequisites (done in the n8n UI or Docker, not in code)

- n8n runs in **Docker** via the repo's `docker-compose.yml`
  (`docker compose up -d`), publishing `http://localhost:5678`. The compose
  file enables MCP with `N8N_MCP_MANAGED_BY_ENV=true` and
  `N8N_MCP_ACCESS_ENABLED=true`.
- The MCP **access token** must be created once in the n8n UI:
  **Settings → Instance-level MCP → Connect a client → API key**, then store it
  in the git-ignored project file `.n8n-mcp-token` (opencode reads it via
  `{file:.n8n-mcp-token}`). When MCP is env-managed the Access
  UI is read-only (expected); the token mint still works there.
- A workflow is only reachable for execute/edit once its **Available in MCP**
  setting is toggled on (workflow menu → Settings). Execution eligibility
  requires a webhook/form/schedule/chat trigger. Workflow building/editing
  needs n8n ≥ 2.13.0.

## External-tool setup prompts (MCP auto-install vs manual)

When building/editing an n8n workflow that references an external tool/service
running on the same machine as the user — e.g. a database (Microsoft SQL
Server, PostgreSQL, MySQL), or any other local/third-party service — the tool's
setup can often be automated via an MCP server, just like the n8n MCP above.

Rule: if that tool is present on the user's PC (installed service, running
process, reachable local port, or the user confirms it), STOP and ask which
they prefer before wiring the node:

  "This <tool> setup can be automated. Would you like me to install the
   <tool> MCP server, or set it up manually in n8n?"

- Install the MCP (recommended, present first): add the MCP to opencode.json
  as a project-scoped server entry. Editing opencode.json means loading the
  `customize-opencode` skill first. Connection secrets never go in committed
  config — read them from a git-ignored file via {file:...} (same pattern as
  .n8n-mcp-token) or the git-ignored .env (see Secret handling). Remind the
  user that opencode must be restarted before the new MCP is live, and use it
  once restarted.
- Set up manually: guide the user through creating the n8n credential/node
  in the UI instead (no opencode.json change).

Prompt only when the tool is actually present — never offer an MCP install
for a service the user doesn't have. Applies to any external service a
workflow references (databases, Slack, Gmail, etc.).

### V3 AI Agent + `usableAsTool` nodes: use a sub-workflow tool, not the raw node

Verified on n8n 2.37.10 (2026-09): when an **AI Agent** (LangChain Agent v3.1)
registers a plain `usableAsTool` base node (e.g. `n8n-nodes-base.microsoftSql`)
as its tool, the agent's engine executes it through the
`ExecutionNodeAction` path and the tool **runs with correct inputs but returns an
empty `ai_tool` output** (`[[]]`) — the model sees `Tool: ""` and answers with no
data (e.g. "0 tickets"). The V1/V2 inline path (`makeHandleToolInvocation` +
`addInputData`) is fine; only the V3 engine path is broken for raw `usableAsTool`
nodes with no `supplyData`/`ai_tool` output. The GitHub #26202 FIFO fix is
present in this version and does NOT resolve it.

**Workaround (implemented in the CLN Database Reporter):** expose the base node
through a **sub-workflow-as-tool** instead:
1. Build a small sub-workflow: `Execute Workflow Trigger` (Define Below,
   `workflowInputs.values`) → the base node (expression reads `{{ $json.<input> }}`,
   credential bound, `onError: continueErrorOutput`) → optional `Stop and Error`
   on the error output. Tag it `subworkflow,tool,<domain>`.
2. **Publish it** — `toolWorkflow` refuses inactive targets ("Workflow is not
   active and cannot be executed.").
3. In the agent workflow, wire a `@n8n/n8n-nodes-langchain.toolWorkflow` v2.2
   node (`source: "database"`, `workflowId` RLC → sub-workflow,
   `workflowInputs` defineBelow with `$fromAI("inputName", ..., "string")`) into
   the agent's `ai_tool` input. The DB runs in a normal main-flow context where
   inputs exist, so results come back correctly.

## Build & test speed

This project overrides the official `n8n-workflow-lifecycle-official` default
("verify `get_workflow_details` after every create/update") in favor of a
**verify-once-at-the-end** loop. It's a deliberate trade: faster iteration in
exchange for mid-build wiring bugs surfacing during validation/test instead of
immediately after a create.

- **Batch node-type lookups.** Resolve all node types a workflow needs in one
  `get_node_types([...])` call (array of discriminators) during PLAN, then
  reuse those definitions for the whole BUILD. Never re-fetch a node type
  already resolved this session. In-session reuse beats re-fetching: repeat
  calls return the same definitions and just burn time.
- **One-shot build.** Author the full workflow in a single
  `create_workflow_from_code` → one `validate_workflow` → one
  `get_workflow_details` at the end → one `publish_workflow`. Avoid piecemeal
  `update_workflow` churn.
- **Verify connections once, at the end.** `get_workflow_details` after the
  final build, before publish. **Exception:** after any update that rewires a
  Merge or multi-wire fan-out (the silent-wiring traps validation misses), do a
  quick targeted `get_workflow_details`.
- **Truncated test inspection.** In debug/test loops, fetch executions with
  `get_workflow_execution` using `truncateData: true` and targeted `nodeNames`.
  Full `includeData` only for the final verification pass.
- **Reuse pin data byte-for-byte** across `test_workflow` iterations. Never
  regenerate pin data between runs.
- **Sub-workflow-first testing.** Test a sub-workflow's core logic in isolation
  before wiring the full graph, so a single bug doesn't burn whole-graph runs.
- **Skill-load-once discipline.** Load each capability skill's `SKILL.md` once
  per session (they're short routers); pull `references/*.md` on demand only
  for the step that needs them. Don't re-load a skill body already in context.

## User profile & session continuity

- **Reuse the same opencode session for a workflow.** A workflow's context
  (decisions, naming, preferences) lives in the session; a fresh session loses
  it and forces the user to re-explain. Prefer continuing the existing session
  over starting over for related work on the same workflow.
- **Learn the user over time** via the **git-ignored** local file
  `.n8n-user-profile.md` at the repo root (never committed). It holds this
  user's observed preferences plus a chronological session log.
  - At the **start** of a session, read it and follow the preferences there.
  - At the **end** of a workflow/task, append one short log line: date,
    workflow/task, and the preference/behavior the user showed. Update the
    *Preferences* section if an existing preference changed.
  - Keep entries short and factual. Only record observable preferences (e.g.
    "prefers SQL Server for DB tasks", "wants each workflow in its own
    folder"); do **not** store secrets, tokens, or anything sensitive.
  - If the file is missing (e.g. fresh clone), create it from the template in
    this repo's setup.md or leave a short hand-written start.

## Secret handling

Never commit the n8n MCP token or any API token. Provide it via the git-ignored
project file `.n8n-mcp-token` (referenced with `{file:...}` in `opencode.json`),
a git-ignored `.env` file, or the shell env — never hardcoded in committed config.

## Restart requirement

opencode loads `opencode.json`, agents, skills, and MCP config **once** at
startup. After changing any of these, quit and restart opencode.
