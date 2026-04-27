# Omnimatch Plugin

Manage your **Omnimatch** account from any MCP client (Claude Code or Cursor). Browse markets and entities; view matches through inboxes (priority, liked, all, and your own filter inboxes); like / reject / respond / unlock matches; share contact details; manage inbox filters; refine profile criteria with text; view and change subscription plans (via Stripe Billing Portal); check credit balance and top up credits.

This plugin is intended for any **authenticated Omnimatch user** (no admin role required).

## Install

### Claude Code

```bash
/plugin marketplace add talentscore/omnimatch-mcp
/plugin install omnimatch@omnimatch-mcp
```

### Cursor

Open the plugin marketplace in Cursor and install **Omnimatch** from this repository, or add it manually via your `mcp.json`:

```json
{
  "mcpServers": {
    "omnimatch": {
      "url": "https://omnimatch.ai/api/mcp"
    }
  }
}
```

## Configuration

- **MCP URL**: defaults to `https://omnimatch.ai/api/mcp`. Override with the `OMNIMATCH_MCP_URL` environment variable (e.g. for self-hosted dev: `http://localhost:8787/api/mcp`).
- **Authentication**: the Omnimatch MCP server requires user OAuth. In Claude Code, run `/mcp` and authenticate when prompted; the client will then send a Bearer token to the server. In Cursor, follow the OAuth prompt the first time you use a tool.

## How it works: codemode

The Omnimatch MCP server exposes a **single tool** called `codemode` ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)). Codemode lets the model write and run JavaScript in a secure sandbox; underlying operations are invoked via `codemode.<toolName>(args)`.

You don't need to call `codemode` yourself — talk to the assistant in natural language and the included skills will produce the right code. Under the hood, calls look like this:

```js
async () => {
  const markets = await codemode.list_my_markets();
  const entities = await codemode.list_my_entities();
  return { markets, entities };
}
```

## Skills

| Skill | What it does |
|-------|--------------|
| `account-info` | Read-only: list markets, entities, inboxes, and matches; inspect inbox filters, plans, credit balance, and credit packs; show next match-discovery time. |
| `account-briefing` | Read-only overview: markets + entities + top recent matches per entity in one summary. Use for "catch me up" / "what's new". |
| `manage-account` | Write: edit entity title / external link / email frequency, refine criteria with text, manage inbox filters, act on matches (like / reject / respond / view link / unlock / share contact / re-evaluate). Destructive actions require an explicit confirm step. |
| `level-up` | Improve a profile: pick which one, review its inboxes, ask a few targeted questions, then propose and apply concrete profile updates. |
| `subscription` | View and change plans (Stripe Billing Portal), check credit balance, list credit packs, and start a Stripe Checkout to top up credits. |

All skills tell the model to talk to you using **real titles** — market title, entity title, plan name, the other side's name — never raw IDs.

## Asking the user

For skills that need to ask you questions (e.g. `level-up`: which profile, which uncertainties, confirmation before applying), the assistant uses **AskUserQuestion** in Claude Code and **AskQuestion** in Cursor so options are presented clearly. In headless environments, the same questions are asked in chat.

## Example prompts

```
What markets am I in?
Show me my matches for my candidate profile
Why did the AI score that match so low?
Add an inbox filter for senior remote roles
Refine my profile: I'm not interested in series-A startups
Approve match <name>
What plans are available, and what am I on?
How many credits do I have?
Top me up with the 100-credit pack
Level up my candidate profile
```

## Troubleshooting

- **HTTP 401** — not authenticated. Run `/mcp` (Claude Code) or trigger a tool call (Cursor) to complete OAuth so the client sends a Bearer token.
- **HTTP 404 / HTML response** — the MCP URL isn't reaching the Omnimatch worker. Check `OMNIMATCH_MCP_URL`. For local dev, point to the worker port (`http://127.0.0.1:8787/api/mcp`), not the Vite dev port.
- **"Entity not found" or "access denied"** — that entity or match isn't owned by the current user; only your own data is visible/editable.
- **Match insights / suitability gated** — the field is plan-gated; ask about plans (`subscription` skill) for the upgrade link.
- **Score is `null` or insights say `computed: false`** — scoring runs asynchronously and can take up to 5 minutes. Wait, don't poll aggressively.
