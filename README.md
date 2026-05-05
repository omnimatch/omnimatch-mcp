# Omnimatch MCP plugins & skills

Plugins and agent skills for **[Claude Code](https://docs.claude.com/en/docs/claude-code/overview)**, **[Cursor](https://cursor.com)**, and any agent compatible with the **[Vercel Skills](https://skills.sh)** ecosystem that connect your editor to the **[Omnimatch](https://omnimatch.ai)** MCP server.

Use natural language to browse your Omnimatch markets, view and act on matches through inboxes, refine your profile criteria, manage subscription plans, and top up credits — all without leaving your editor.

## Plugins in this marketplace

| Plugin | Description |
|--------|-------------|
| [`omnimatch`](./plugins/omnimatch) | Manage your Omnimatch account: markets, entities, inboxes, matches, inbox filters, plans, credits. |

## Skills

The same skills are also exposed at the repo root under [`skills/`](./skills) so they're discoverable by the Vercel Skills CLI (`npx skills`) and listed on [skills.sh](https://skills.sh).

| Skill | What it does |
|-------|--------------|
| [`account-info`](./skills/account-info) | Read-only: list markets, entities, inboxes, matches, plans, credit balance. |
| [`account-briefing`](./skills/account-briefing) | Read-only overview: markets + entities + top recent matches in one summary. |
| [`manage-account`](./skills/manage-account) | Edit profile, manage filters, act on matches, change plan. Confirms destructive actions. |
| [`level-up`](./skills/level-up) | Improve a profile: review inboxes, ask targeted questions, propose updates. |
| [`subscription`](./skills/subscription) | View/change plans (Stripe Billing Portal), check balance, top up credits. |

The files under `skills/` are symlinks to the canonical SKILL.md files inside the `omnimatch` plugin, so there's only one source of truth.

## Install

### Claude Code

Run these slash commands in Claude Code (each on its own line, in order):

```
/plugin marketplace add omnimatch/omnimatch-mcp
/plugin install omnimatch@omnimatch-mcp
/reload-plugins
/mcp
```

- `/plugin marketplace add omnimatch/omnimatch-mcp` — register this repo as a plugin marketplace.
- `/plugin install omnimatch@omnimatch-mcp` — install the `omnimatch` plugin from it.
- `/reload-plugins` — load the new plugin, its skills, and the MCP server entry.
- `/mcp` — complete OAuth so Claude Code holds a Bearer token for the Omnimatch MCP server. Re-run any time you need to re-authenticate.

### Cursor

Open the plugin marketplace in Cursor and install from this repository, or add the MCP server manually to your `mcp.json`:

```json
{
  "mcpServers": {
    "omnimatch": {
      "url": "https://omnimatch.ai/api/mcp"
    }
  }
}
```

### Vercel Skills (`npx skills`)

Install all skills with the [Skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add omnimatch/omnimatch-mcp
```

The skills tell your agent how to call Omnimatch via MCP, but they don't install the MCP server itself — for the connection, configure the Omnimatch MCP server (`https://omnimatch.ai/api/mcp`) in your client's MCP config (see Claude Code / Cursor sections above).

## Authentication

The plugin talks to the Omnimatch MCP server at `https://omnimatch.ai/api/mcp` over HTTP and authenticates via OAuth. The first time you invoke a tool the client will prompt you to log in; your Bearer token is reused for the session after that.

## Repo layout

```
omnimatch-mcp/
├── .claude-plugin/marketplace.json     # Claude Code marketplace catalog
├── .cursor-plugin/marketplace.json     # Cursor marketplace catalog
├── skills/                             # Vercel Skills entry point (npx skills, skills.sh)
│   ├── account-info/SKILL.md           #   → symlink to plugins/omnimatch/skills/...
│   ├── account-briefing/SKILL.md
│   ├── manage-account/SKILL.md
│   ├── level-up/SKILL.md
│   └── subscription/SKILL.md
└── plugins/
    └── omnimatch/
        ├── .claude-plugin/plugin.json  # Claude Code manifest
        ├── .cursor-plugin/plugin.json  # Cursor manifest
        ├── mcp.json                    # Cursor MCP server config
        ├── skills/                     # canonical SKILL.md files (symlinked from /skills)
        │   ├── account-info/SKILL.md
        │   ├── account-briefing/SKILL.md
        │   ├── manage-account/SKILL.md
        │   ├── level-up/SKILL.md
        │   └── subscription/SKILL.md
        └── README.md
```

The `omnimatch` plugin's Claude Code MCP config is inlined in `.claude-plugin/plugin.json` (`mcpServers` field). The Cursor MCP config lives in a sibling `mcp.json` at the plugin root. Both reference the same hosted MCP endpoint.

## License

MIT — see [LICENSE](./LICENSE).
