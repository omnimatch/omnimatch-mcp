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

```bash
/plugin marketplace add talentscore/omnimatch-mcp
/plugin install omnimatch@omnimatch-mcp
```

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

Install one or more skills with the [Skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add talentscore/omnimatch-mcp@account-info
npx skills add talentscore/omnimatch-mcp@manage-account
# ...or all of them
npx skills add talentscore/omnimatch-mcp
```

The skills tell your agent how to call Omnimatch via MCP, but they don't install the MCP server itself — for the connection, configure the Omnimatch MCP server (`https://omnimatch.ai/api/mcp`) in your client's MCP config (see Claude Code / Cursor sections above). There's no manual submission to skills.sh — once the repo is public and someone runs `npx skills add`, install telemetry surfaces it on the directory automatically.

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

## Adding a plugin to this marketplace

1. Create a new folder under `plugins/<your-plugin-name>/`.
2. Add `.claude-plugin/plugin.json` (Claude Code manifest) and `.cursor-plugin/plugin.json` (Cursor manifest).
3. If the plugin connects to an MCP server, declare it in `plugin.json` (`mcpServers` field for Claude Code) and in a separate `mcp.json` at the plugin root (for Cursor).
4. Add `skills/<skill-name>/SKILL.md` files describing what the plugin can do.
5. Register the plugin in both `.claude-plugin/marketplace.json` and `.cursor-plugin/marketplace.json`.

See [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference) and [github.com/cursor/plugin-template](https://github.com/cursor/plugin-template) for the full plugin schemas.

## License

MIT — see [LICENSE](./LICENSE).
