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

The n8n skills come from the **official n8n-io/skills** repo, vendored as a
project-local clone at `<project>/n8n-skills/`. `opencode.json` loads its
plugin (`"plugin": ["./n8n-skills/opencode/plugin.ts"]`), which:

- auto-registers all 14 skills from `n8n-skills/skills/` (13 capability skills +
  the `using-n8n-skills-official` meta-skill) — no manual `skills.paths`.
- injects `using-n8n-skills-official` into the system prompt every session.
- fires bash hooks after n8n MCP tool calls (requires `bash` + `jq` on PATH).

Update the skills by pulling the clone: `git -C n8n-skills pull`. When working
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

## Secret handling

Never commit the n8n MCP token or any API token. Provide it via the git-ignored
project file `.n8n-mcp-token` (referenced with `{file:...}` in `opencode.json`),
a git-ignored `.env` file, or the shell env — never hardcoded in committed config.

## Restart requirement

opencode loads `opencode.json`, agents, skills, and MCP config **once** at
startup. After changing any of these, quit and restart opencode.
