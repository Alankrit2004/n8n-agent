# n8n-agent

An **opencode assistant** for working with a self-hosted **n8n** workflow
automation instance. It loads n8n's official skills and connects to the n8n
built-in MCP server, so a CLI coding agent can search, build, edit, and execute
workflows directly.

## Get started (first run)

This repo is a **project**, not a stand-alone app: you open it in opencode and
let the agent set everything up. There is no separate install step.

1. **Clone the repo**, then launch opencode from inside it:

   ```bash
   git clone https://github.com/Alankrit2004/n8n-agent.git
   cd n8n-agent
   opencode
   ```

2. **Paste this prompt as your first message:**

   > Read `setup.md` and set up this project, asking me the setup questions as you
   > go.

   The agent will:
   - ask how n8n should be hosted (default: **Docker**) and where,
   - copy `.env.example` → `.env` and start n8n,
   - prompt you for the two one-time manual bits it can't do for you: creating
     the MCP access token in the n8n UI, and toggling a workflow **Available in MCP**,
   - write the token to the git-ignored `.n8n-mcp-token` file,
   - and finish by verifying the skills, MCP connection, and hooks all work.

3. **Restart opencode** once told to (config is read at startup).

That's it. You'll be talking to n8n through opencode from there.

> **Keep one opencode session per workflow.** Session context (decisions, naming,
> preferences) is where a workflow lives; starting a new session mid-workflow
> forces you to re-explain. The agent also keeps a git-ignored
> `.n8n-user-profile.md` at the repo root that it reads each session and appends
> to at the end of each workflow, so it gradually adapts to how you work. That
> file is local to you and never committed.

> **Already have n8n running, or hitting trouble?** The full decision questions,
> manual steps, troubleshooting, and token-file gotchas are in
> [`setup.md`](setup.md).

## What's in here

| Path | Purpose |
| --- | --- |
| `opencode.json` | Project config: loads the n8n-skills plugin + the `n8n` MCP server. |
| `n8n-skills/` | Vendored in-tree copy of the official [`n8n-io/skills`](https://github.com/n8n-io/skills) repo (plugin + 13 capability skills + meta-skill). |
| `docker-compose.yml` | Runs n8n in Docker with **MCP access enabled** via env vars. |
| `setup.md` | **Portable one-shot runbook** — read this to stand this project up on a fresh machine. |
| `AGENTS.md` | Instructions loaded by opencode describing how this project is wired. |
| `.env` / `.n8n-mcp-token` | Git-ignored local secrets (n8n MCP access token). Never committed. |
| `.env.example` | Template for the local env file. |

## Quick start (manual, if you'd rather not use an agent)

n8n runs in **Docker**, MCP is enabled out of the box, and the agent
authenticates with the n8n MCP server through a header token.

```bash
# 1. Copy and fill in local secrets
cp .env.example .env        # set a strong N8N_ENCRYPTION_KEY
# 2. Start n8n
docker compose up -d
# 3. In the n8n UI, create an MCP access token:
#    Settings → Instance-level MCP → Connect a client → API key
#    Put the token (bare, no newline/BOM) in the git-ignored file .n8n-mcp-token
# 4. Launch opencode from a shell in this directory
```

> **Prefer the guided path?** See the [Get started (first run)](#get-started-first-run)
> section above, or read [`setup.md`](setup.md) for the full walkthrough with
> troubleshooting and the exact token file format.

## How auth works (in short)

The n8n MCP server is hosted at `http://localhost:5678/mcp-server/http` and is
configured as a **remote** server in `opencode.json`:

```jsonc
"mcp": {
  "n8n": {
    "type": "remote",
    "url": "http://localhost:5678/mcp-server/http",
    "headers": { "Authorization": "Bearer {file:.n8n-mcp-token}" },
    "oauth": false,
    "enabled": true
  }
}
```

- `{file:.n8n-mcp-token}` makes opencode read the token straight from the
  git-ignored project file at config load — no shell env needed.
- `"oauth": false` is required: n8n uses an API-key header, **not** OAuth.
  Without it, opencode's OAuth auto-detection fights the header and shows a
  misleading "needs auth".

## Credits

The `n8n-skills/` directory is vendored from the official
**[n8n-io/skills](https://github.com/n8n-io/skills)** repository. It contains
the n8n skills plugin plus the capability skills and meta-skill used by this
project. See `n8n-skills/LICENSE` for the license terms.

## Notes

- **Everything is project-scoped.** No opencode/n8n config is written to global
  (`~/.config/opencode`) scope.
- **Secrets**: the n8n MCP token and `N8N_ENCRYPTION_KEY` must never be
  committed. They live in the git-ignored `.env` / `.n8n-mcp-token`.
- **Restart opencode** after changing `opencode.json` / the token file — MCP
  config is read once at startup.
- **Build/test speed** is tuned for fast iteration: batched node-type lookups,
  one-shot builds, and verify-once-at-the-end (`AGENTS.md` "Build & test
  speed") instead of the official skill's verify-after-every-update default.
- Update the vendored skills by re-vendoring from upstream (see `setup.md`
  Step 1) — do not run `git -C n8n-skills pull` (no nested `.git` there; it
  resolves to this repo).