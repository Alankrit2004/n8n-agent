# n8n Agent — Setup Guide

This file is a **portable setup runbook** for standing up this project on any
machine with a CLI coding agent (opencode, Claude Code, Codex, Gemini CLI, …).

Hand this file to the agent: `"Read setup.md and set up this project, asking me
the setup questions as you go."` The agent reads it, asks the decision
questions, and configures everything **project-locally only** (never global
`~/.config/opencode` or `~/.claude`).

---

## What this project is

An agent-assisted **n8n workspace**: a CLI coding agent that can search, build,
edit, and execute n8n workflows on a locally-hosted n8n instance.

Two pieces are wired together:

1. **n8n's official skills** (workflow best practices) — auto-loaded by the
   `n8n-skills` opencode plugin.
2. **n8n's built-in MCP server** — lets the agent talk to your n8n instance.

Everything is **project-scoped**: config lives in `opencode.json` at the repo
root; skills are loaded from the vendored `n8n-skills/` clone. If you're on a
non-opencode CLI, the project expected to be opened as that CLI's project.

---

## Prerequisites (check before asking questions)

Run these checks first; report anything missing to the user.

- **Docker** installed and the daemon running (`docker --version`, then
  `docker ps` to confirm the daemon is up). **n8n is hosted with Docker** (this
  repo ships a `docker-compose.yml`). Docker or `docker compose` is required.
- **Git** installed (`git --version`). Needed to clone/pull the skills repo.
- **Node.js** installed and on PATH (`node --version`). Only needed for
  `npx n8n` if the user picks the npm hosting alternative; not required for the
  Docker path.
- For the plugin's post-tool hook reminders:
  - **bash** available on PATH (bundled with Git for Windows; native on macOS/Linux).
  - **jq** available on PATH (`jq --version`). If missing, install it:
    - Windows: `winget install jqlang.jq`
    - macOS: `brew install jq`
    - Linux: `sudo apt install jq` (or your distro's package)
- The project's `n8n-skills/` directory exists and has `skills/`, `hooks/`, and
  `opencode/plugin.ts`. If not, restore it (e.g. `git clone
  https://github.com/n8n-io/skills.git n8n-skills` or re-download).

---

## Questions to ask the user

Ask these **before making changes**. Stop and collect answers.

### Q1 — How is n8n hosted? (npm or Docker)

> **This project defaults to Docker.** n8n runs as a container via the
> `docker-compose.yml` in this repo. Confirm or switch to npm.

- **Option A — Docker (default/recommended):** `docker compose up -d` from this
  repo using the provided `docker-compose.yml`. Reproducible, isolated, keeps
  all n8n config in this project. *(This is what this project is set up for.)*
- **Option B — npm (locally hosted):** `npx n8n` runs n8n directly with Node on
  the host. Simpler for a throwaway install but no compose config; you must
  configure MCP enablement manually.

> If the user picks npm, note that the docker-compose MCP env vars
> (`N8N_MCP_MANAGED_BY_ENV`, `N8N_MCP_ACCESS_ENABLED`) do **not** apply — you'll
> enable MCP in the n8n UI instead (Step 4).

### Q2 — n8n instance base URL / port

- Default: `http://localhost:5678` (the compose file publishes 5678).
- If the instance is remote, on a non-default port, or behind a tunnel, capture
  the exact base URL the MCP client will reach (e.g. a Cloudflare tunnel URL).

### Q3 — How should the MCP auth token be provided?

The token must **never** be written into committed config. This project
standardizes on a **git-ignored project-local token file** that opencode reads
directly via `{file:…}` substitution (no shell/env gymnastics):

- **Option A — project-local `.n8n-mcp-token` file (default/recommended):** put
  the bare token in a git-ignored file `.n8n-mcp-token` at the repo root.
  `opencode.json` reads it with `"Authorization": "Bearer {file:.n8n-mcp-token}"`.
  Keep a copy in git-ignored `.env` (`N8N_MCP_TOKEN=`) as a backup if you like —
  it's not required.
- **Option B — shell env var** (`N8N_MCP_TOKEN`): set in the process env before
  launching the CLI, referenced as `{env:N8N_MCP_TOKEN}`. Requires the var to be
  exported in the launching shell or it becomes an empty string (→ 401).

### Q4 — MCP access token (one-time manual step)

The n8n **access token itself must be created in the n8n UI** — the agent cannot
generate it. Set this expectation up front so the user knows a brief manual step
is required (Step 4.2): **Settings → Instance-level MCP → Connect a client →
API key**, then store the token in `.n8n-mcp-token` (Step 5).

---

## Project-locality rule

**Hard constraint:** all configuration belongs under this project directory.
Never write to global config scopes:

- ❌ `~/.config/opencode/opencode.json` (global)
- ❌ `~/.claude/claude_desktop_config.json`, `claude mcp add` (global)
- ❌ `~/.codex/config.toml` (global)
- ✅ `./opencode.json` (project)
- ✅ `./docker-compose.yml` (n8n container config — project)
- ✅ `./n8n-skills/` (vendored repo clone — project)
- ✅ `./.n8n-mcp-token` (git-ignored token file — project only)
- ✅ `./.env` (git-ignored local token file — project only)

Exception: **jq** is a system tool installed via the OS package manager — that
belongs at the system level, not in the project.

---

## Step-by-step setup

> If the agent's CLI supports it, restart the tool **after** config changes
> (opencode loads config once at startup; it is not hot-reloaded).

### Step 1 — Verify/restore `n8n-skills/` (the skills + plugin source)

The repo should already contain a `n8n-skills/` clone. If it's missing or empty:

```bash
git clone https://github.com/n8n-io/skills.git n8n-skills
# or shallow: git clone --depth 1 https://github.com/n8n-io/skills.git n8n-skills
```

Update it later: `git -C n8n-skills pull`.

### Step 2 — Configure `opencode.json` (project-scoped)

This file drives both the skills plugin and the n8n MCP server. Base template:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["./n8n-skills/opencode/plugin.ts"],
  "mcp": {
    "n8n": {
      "type": "remote",
      "url": "<BASE_URL>/mcp-server/http",
      "headers": { "Authorization": "Bearer {file:.n8n-mcp-token}" },
      "oauth": false,
      "enabled": true
    }
  }
}
```

Rules:
- Replace `<BASE_URL>` with the answer from Q2. For the default Docker setup:
  `http://localhost:5678`. The n8n MCP endpoint is always
  `<base>/mcp-server/http`.
- **Keep the MCP server named `n8n`.** The plugin matches tool names containing
  `n8n` to fire its post-tool reminders; a different name silently disables them.
- Use `{file:.n8n-mcp-token}` (Q3) so the token never lands in the committed
  file — opencode reads it straight from the git-ignored `.n8n-mcp-token` at
  the repo root (no shell env needed). `{env:…}` also works but becomes an empty
  string if the var isn't exported, causing a 401.
- Set `"oauth": false` — n8n authenticates with an API-key header, not OAuth.
  Without this, opencode's OAuth auto-detection fights the header token and
  shows a misleading "needs auth" state. If a previous attempt ran
  `opencode mcp auth n8n`, also delete the stale OAuth record it wrote to
  `~/.local/share/opencode/mcp-auth.json` (see troubleshooting) or it keeps
  confusing the connection status.
- **Restart opencode after every change** to `opencode.json` / the token file —
  MCP config and `{file:...}` values are read **once at startup**, not hot-reloaded.

### Step 3 — Start n8n with Docker (per Q1)

The repo's `docker-compose.yml` runs n8n in a container with **MCP access
already enabled** via environment variables, plus a persistent volume.

1. Copy `.env.example` → `.env` and set a strong `N8N_ENCRYPTION_KEY`:
   ```bash
   # powershell
   Copy-Item .env.example .env   # then edit .env
   # bash/zsh/macOS/Linux
   cp .env.example .env           # then edit .env
   ```
2. Start the container:
   ```bash
   docker compose up -d
   ```
3. Verify it's up: `curl -s http://localhost:5678/healthz` returns `ok`.
   First run takes a moment to boot and initialize the DB.

> **npm alternative (Q1 = npm):**
> ```bash
> npx n8n   # runs on http://localhost:5678
> ```
> There's no compose file injecting the MCP env, so enable MCP manually: in the
> n8n UI open **Settings → Instance-level MCP** → **Enable MCP access**.

### Step 4 — Configure MCP authentication

The MCP server is enabled by the compose file (`N8N_MCP_MANAGED_BY_ENV=true` +
`N8N_MCP_ACCESS_ENABLED=true`, the latter is n8n ≥ 2.20). When managed by env,
the **Access** settings UI is read-only; the MCP server is on and accepts a
bearer token. What remains is creating that token — a **one-time manual UI
step** (n8n ≥ 2.33 shows the API-key dialog; see Q4):

1. Open n8n: `http://localhost:5678`, sign in (owner setup on first run).
2. **Settings → Instance-level MCP**.
3. **Connect a client → API key** tab (n8n ≥ 2.33; older versions: a single
   MCP settings page). If MCP is env-managed, the access toggle is read-only —
   that's expected; the "API key / Connect" action still generates the token.
4. Copy the **access token** and, if shown, the **Server URL**
   (`http://localhost:5678/mcp-server/http`).
5. For each workflow the agent should edit/execute: open it → Workflow menu
   (`…`) → **Settings** → toggle **Available in MCP**. (Execution eligibility
   requires a webhook/form/schedule/chat trigger; building/editing needs
   n8n ≥ 2.13.0.)

### Step 5 — Set the MCP token (per Q3)

Put the token from Step 4 into the project-local, git-ignored token file that
opencode reads directly:

**`.n8n-mcp-token` route (default):**
```
# create the file .n8n-mcp-token in the repo root containing ONLY the token,
# no quotes, no newline. opencode reads it via {file:.n8n-mcp-token}.
```
PowerShell:
```powershell
$tok = "PASTE_TOKEN_FROM_STEP_4"   # your real token
[System.IO.File]::WriteAllText((Join-Path (Get-Location) ".n8n-mcp-token"), $tok)
```
bash/zsh:
```bash
printf '%s' 'PASTE_TOKEN_FROM_STEP_4' > .n8n-mcp-token
```

`.n8n-mcp-token` is git-ignored — **never commit it.**

> **File-format requirements (matter!):**
> - The file must contain **only the token** — no quotes, no trailing newline.
>   opencode `.trim()`s the content, so a trailing newline is harmless, but a
>   stray space, quote, or a UTF-**8 BOM** corrupts the token and causes a 401.
>   The PowerShell `[System.IO.File]::WriteAllText(path, str)` form writes **no
>   BOM and no newline** — use it. (Writing via regular editors can add a BOM.)
> - `{file:...}` paths resolve **relative to the config file's directory**
>   (the repo root for `./opencode.json`). Don't move the token file.
> - The content is JSON-string-escaped during substitution, so a JWT
>   (base64url-only characters) is safe inside the JSON config.

> **Why not `{env:N8N_MCP_TOKEN}`?** opencode substitutes `{env:…}` with an
> **empty string if the var isn't set in the process env**, and it does **not**
> auto-load `.env`. If you use env vars, you must export `N8N_MCP_TOKEN` in the
> shell **before** launching opencode, then restart. The `{file:…}` route avoids
> all of that — opencode reads the file at config load.

### Step 6 — Verify the install

After restarting the CLI agent:

1. **n8n reachable** → `curl -s http://localhost:5678/healthz` returns `ok`.
2. **Skills registered** → in opencode, confirm the 14 skills are discoverable
   (e.g. the `using-n8n-skills-official` meta-skill is injected into the system
   prompt). If not, look for `[n8n-skills]` warnings at startup.
3. **MCP connected** → the `n8n` server connects and n8n MCP tools are
   available (e.g. `search_workflows`, `get_workflow`, `create_workflow`).
4. **Hooks fire** → run any n8n MCP tool and confirm post-tool reminders
   appear. If they don't, check `jq --version` in a fresh **bash** shell
   (jq installed after the shell started won't be on PATH until restart).

**Pre-flight: verify the token + endpoint before blaming opencode.** This proves
whether auth and the MCP endpoint work at all (use it when you see a 401
"SSE error: Non-200 status code (401)" in the MCP status). PowerShell:

```powershell
$tok = (Get-Content .n8n-mcp-token -Raw).Trim()
# health check first
Invoke-WebRequest -Uri "http://localhost:5678/healthz" -UseBasicParsing
# MCP initialize handshake with the exact headers opencode sends
$h = @{ Authorization = "Bearer $tok"; Accept = "application/json, text/event-stream" }
Invoke-WebRequest -Uri "http://localhost:5678/mcp-server/http" -Headers $h -UseBasicParsing -Method Post `
  -ContentType "application/json" `
  -Body '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}'
# Expect HTTP 200 + an "event: message" SSE response. A 401 = bad/empty token.
```

> If the pre-flight returns **200** but opencode still shows 401, the token is
> fine and the problem is how opencode loaded it (see troubleshooting). If it
> returns **401**, the token in `.n8n-mcp-token` is missing/wrong/corrupt (BOM,
> newline, quoting) or MCP isn't actually enabled on the n8n side.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `[n8n-skills] hooks/ or skills/ not found` warning | `n8n-skills/` missing/moved. Re-clone it (Step 1). |
| No post-tool hook reminders | `jq` missing from **git bash** PATH, or MCP server not named `n8n`. Install jq (Q0/prereqs) and confirm `bash -lc "jq --version"`; verify the server name in `opencode.json`. |
| MCP won't connect / `SSE error: Non-200 status code (401)` | Run the Step 6 pre-flight to split symptoms. 401 means the token is missing/wrong/corrupt (BOM, newline, quoting) or MCP isn't enabled (re-check Step 4). If you used `{env:N8N_MCP_TOKEN}` instead of `{file:…}`, an unset var substitutes as **empty** → 401 (opencode doesn't auto-load `.env`). |
| MCP shows "needs auth" even though headers are set / `opencode mcp auth n8n` was run | You mixed OAuth with the header token. n8n uses an API-key header, **not** OAuth. Make sure `"oauth": false` is in `opencode.json`, then **clear the stale OAuth credentials** opencode stored globally: delete `~/.local/share/opencode/mcp-auth.json` (or any `n8n` entry in it), then restart opencode. |
| MCP was working, then shows "needs auth" after a session | Most likely a stale OAuth token in `~/.local/share/opencode/mcp-auth.json`. With `oauth: false` this file shouldn't be involved; if it reappears, delete it / the `n8n` entry and restart opencode. |
| `opencode mcp auth n8n` says "already authenticated" but MCP still fails | That command manages OAuth credentials, which are orthogonal to your header token. With `oauth: false` it's not used; clear `mcp-auth.json` and rely on the header token (`{file:.n8n-mcp-token}`). |
| "Settings → Instance-level MCP" Access UI is read-only | Expected — MCP is `N8N_MCP_MANAGED_BY_ENV=true`. The "Connect a client → API key" action still works to mint the token. |
| Workflow not found / can't edit | Workflow isn't toggled **Available in MCP**; building/editing needs n8n ≥ 2.13.0. |
| Container won't start / port in use | Port 5678 busy or `N8N_ENCRYPTION_KEY` empty. Change the published port or key in `.env` / `docker-compose.yml`. |
| Meta-skill injection stops after opencode upgrade | The plugin uses experimental hooks (`experimental.chat.system.transform`, `experimental.session.compacting`) that can change; check them against your `@opencode-ai/plugin` version. |

---

## Updating

Skills / plugin / hooks: `git -C n8n-skills pull`, then **restart the CLI agent**.

---

## Security notes

- **Never commit** the n8n MCP token or any API key. It lives in the git-ignored
  `.n8n-mcp-token` file (read via `{file:…}`) or a git-ignored `.env` / env var.
- **Never leave the token in a plaintext file in the repo** (e.g. a
  `configuration.md` export from n8n). If the token appears anywhere else,
  delete that file and add it to `.gitignore` — the token is only valid while
  it isn't exposed. If unsure the token leaked, revoke/recreate it in the n8n
  UI (Settings → Instance-level MCP) and update `.n8n-mcp-token`.
- The MCP token grants workflow read/edit/execution to whichever client holds
  it. Treat it like a credential.
- **`N8N_ENCRYPTION_KEY`** (Docker) encrypts stored credentials in n8n. Set a
  long random value in `.env`; keep it stable — losing it makes stored
  credentials unrecoverable. Don't commit the real value.
- Keep the vendored `n8n-skills/` clone's authentication code untouched unless
  you know what you're doing.

## Docker commands

- **Start:** `docker compose up -d`
- **Stop:** `docker compose stop`
- **Logs:** `docker compose logs -f n8n`
- **Recreate after config change:** `docker compose up -d --force-recreate`
- **Full reset (wipes n8n data):** `docker compose down -v`

Compose reads variables from `.env` (source of `N8N_ENCRYPTION_KEY`, `N8N_HOST`,
etc.). After editing compose/.env, recreate the container.

