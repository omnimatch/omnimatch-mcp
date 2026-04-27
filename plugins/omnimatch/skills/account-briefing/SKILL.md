---
name: account-briefing
description: Give a read-only overview of the user's markets, entities, and top recent matches. Use the Omnimatch MCP codemode tool with code that calls codemode.list_my_markets(), codemode.list_my_entities(), then for each entity codemode.list_inboxes() and codemode.list_inbox_matches({ inboxKey: 'priority', limit }) to produce a briefing. Use when the user asks for a briefing, overview, summary, "what's new", "catch me up", or "my recent matches". Do not mutate data—use manage-account for that. When talking to the user, use real titles (market title, entity title, match title)—never show IDs.
---

# Account Briefing (Overview)

Use the **Omnimatch MCP** codemode tool to produce a **read-only overview** of the user's markets, entities, and top recent matches ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)). The server exposes only **codemode**: call it with a `code` parameter (async arrow function body) that runs `codemode.list_my_markets()`, `codemode.list_my_entities()`, then per entity `codemode.list_inboxes({ entityId })` and `codemode.list_inbox_matches({ entityId, inboxKey, limit })`, and returns the combined summary. This skill only reads data; for actions use **manage-account**.

## When to use

- User asks for a **briefing**, **overview**, **summary**, or **catch me up** on their account.
- User asks **what's new**, **how are my matches**, **my recent matches**, or **how are my entities doing**.
- User wants to see **their markets, entities, and top matches** in one place.

**When talking to the user:** Use **real titles**, never IDs. For each market use its **market title**; for each entity use its **title** and **type** (e.g. "job", "candidate"); for each match use the row's **title** field (the other side's name). Do not show market ID, entity ID, or match ID in the summary.

## Workflow

1. **List markets and entities** — Run `codemode.list_my_markets()` and `codemode.list_my_entities()` to get the top-level shape.
2. **For each entity (or the first N if many)** — Run `codemode.list_inboxes({ entityId })` to see counts, then `codemode.list_inbox_matches({ entityId, inboxKey: "priority", limit: 5 })` to get top priority matches; if priority is empty, fall back to `inboxKey: "all"`.
3. **For each top match** (optional, when the user wants more depth) — `codemode.get_match({ matchId, entityId })` for one or two top matches to ground the briefing.
4. **Summarize** — Present a concise overview per market → per entity → top matches. Use scores, status (mutual / sharedWithYou / etc), and updatedAt to decide what's interesting. Mention inbox totals (priority count, liked count, all count) where useful. Do not call any write tools.

If the user has many entities, summarize the first few and offer to dig deeper into one.

## Codemode operations (read-only)

- `codemode.list_my_markets()`
- `codemode.list_my_entities()`
- `codemode.list_market_entities({ marketId })`
- `codemode.list_inboxes({ entityId })` — returns priority/liked/all plus any inbox filters with counts.
- `codemode.list_inbox_matches({ entityId, inboxKey, offset, limit, sortBy })` — TUI table; use `inboxKey: "priority"` (or "liked" / "all" / `"query:{filterId}"`).
- `codemode.get_match({ matchId, entityId })` — full match detail; some fields are plan-gated and come back as an upgrade-message string.
- `codemode.list_inbox_filters({ entityId })` — when the user asks about their saved filters.
- `codemode.list_plans({ marketId })` — when the user asks about their plan.

This skill only reads; do not call write tools (update_entity_*, like_match, reject_match, respond_to_match, unlock_match, share_my_contact_with_match, add_inbox_filter, update_inbox_filter, remove_inbox_filter, change_plan).
