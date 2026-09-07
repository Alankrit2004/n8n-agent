# n8n-agent

An **opencode assistant** for working with a self-hosted **n8n** workflow
automation instance. It loads n8n's official skills and connects to the n8n
built-in MCP server, so a CLI coding agent can search, build, edit, and execute
workflows directly.

## What's in here

| Path | Purpose |
| --- | --- |
| `opencode.json` | Project config: loads the n8n-skills plugin + the `n8n` MCP server. |
| `n8n-skills/` | Vendored clone of the official [`n8n-io/skills`](https://github.com/n8n-io/skills) repo (plugin + 13 capability skills + meta-skill). |
| `docker-compose.yml` | Runs n8n in Docker with **MCP access enabled** via env vars. |
| `setup.md` | **Portable one-shot runbook** — read this to stand this project up on a fresh machine. |
| `AGENTS.md` | Instructions loaded by opencode describing how this project is wired. |
| `.env` / `.n8n-mcp-token` | Git-ignored local secrets (n8n MCP access token). Never committed. |
| `.env.example` | Template for the local env file. |

## Quick start

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

> **Want the full walkthrough, including troubleshooting and the exact token
> file format?** Read [`setup.md`](setup.md). It's written to be handed to a
> coding agent so the whole environment can be reproduced on a different
> machine in one pass.

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
- Update the vendored skills with `git -C n8n-skills pull`.