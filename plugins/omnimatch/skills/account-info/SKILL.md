---
name: account-info
description: Read-only account information. Use the Omnimatch MCP codemode tool to list the user's markets and entities, view inboxes and matches, inspect inbox filters, read subscription plans, and check credit balance / available credit packs. Call codemode with code that invokes codemode.list_my_markets(), codemode.list_my_entities(), codemode.list_inboxes({ entityId }), codemode.list_inbox_matches({ entityId, inboxKey }), codemode.get_match({ matchId, entityId }), codemode.list_plans({ marketId }), codemode.get_my_credit_balance({ marketId }), codemode.list_credit_packs({ marketId }), codemode.get_next_match_discovery({ entityId }), etc. Use when the user asks to view or list their markets, entities, matches, inbox filters, plans, balance, or when their next search runs. Do not use for editing or purchases—use the manage-account or subscription skill for that. When talking to the user, always use real titles (market title, entity title, match/other side name, type title, plan name)—never show IDs.
---

# Account Info (Read-Only)

Use the **Omnimatch MCP** codemode tool to read the current user's account data ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)). The server exposes only **codemode**: call it with a `code` parameter (async arrow function body) that runs `codemode.list_my_markets()`, `codemode.list_inbox_matches({ ... })`, etc., and returns the result. This skill is **read-only**: list and view only. For updating profiles, filters, plan changes, or match actions, use the **manage-account** skill.

## When to use

- User asks **what markets am I in?** → `codemode.list_my_markets()`.
- User asks **what entities do I have?** → `codemode.list_my_entities()` (across markets) or `codemode.list_market_entities({ marketId })` for one market.
- User asks **show me details for entity X** → `codemode.get_entity({ entityId })` (returns title, type, externalLink, description, criteria, files with summaries, inbox filters, entity query answers if plan allows).
- User asks **what are my matches** or **show matches** → first `codemode.list_inboxes({ entityId })` to see inboxes (priority, liked, all, plus each inbox filter), then `codemode.list_inbox_matches({ entityId, inboxKey })` to load the table for the chosen inbox.
- User asks about **a specific match** → `codemode.get_match({ matchId, entityId })` for full details (insights, scores, suitability, contact). The actual reachable contact (LinkedIn URL, email) lives in `otherEntityContact` on the response, not in the `contact` column of the inbox row. Use `codemode.view_match_link({ matchId, entityId })` for just the externalLink.
- User asks **why did we match** for one inbox filter, or **what does filter X say for match Y** → `codemode.list_match_inbox_filter_answers({ matchId, entityId })`.
- User asks about **inbox filters** they have → `codemode.list_inbox_filters({ entityId })`.
- User asks about **subscription**, **plans**, or **what features do I have** → `codemode.list_plans({ marketId })`.
- User asks **how many credits do I have**, **what's my balance**, or **am I out of credits** → `codemode.get_my_credit_balance({ marketId })`. For markets that don't charge credits, the tool returns `creditsEnabled: false`—say so briefly.
- User asks **what credit packs can I buy** or **how do I top up** → `codemode.list_credit_packs({ marketId })`. Present packs by name and credit amount; never expose Stripe price ids in user-facing copy. Direct the user to the **subscription** skill (or `topup_credits`) when they want to actually buy.
- User asks **when is my next search**, **when do new matches arrive**, or **when does discovery run** → `codemode.get_next_match_discovery({ entityId })`. Returns `{ scheduled: false }` or `{ scheduled: true, nextRunAt }` (UTC ISO-8601).

**When talking to the user:** Always use **real titles**, never IDs. Refer to entities by their **title**, markets by their **market title**, matches by the other side's **title**, types by the **entity type title** (e.g. "job", "candidate"), and plans by **plan name**. Do not show market ID, entity ID, match ID, or Stripe price ID in user-facing messages—use IDs only inside codemode parameters.

## Codemode read-only tools

Call the **codemode** tool with `code` set to an async arrow function body. Inside it, you can use:

- **codemode.list_my_markets()** → `[{ marketId, title, description }]`.
- **codemode.list_my_entities()** → flat `[{ entityId, marketId, marketTitle, title, type, shortDescription }]`.
- **codemode.list_market_entities({ marketId })** → entities in one market.
- **codemode.get_entity({ entityId })** → full profile.
- **codemode.list_inboxes({ entityId })** → built-in inboxes (priority, liked, all) plus filter inboxes (key `query:{filterId}`), each with a count.
- **codemode.list_inbox_matches({ entityId, inboxKey, offset, limit, sortBy, includeRejected, includeLowScores })** → TUI-friendly table: `{ inboxKey, total, offset, columns, rows: [{ matchId, title, yourScore, theirScore, status, contact, updatedAt, locked }] }`. **`contact` is misleading**: it's a tri-state (`"shared" | "pending" | "none"`) that only reflects owner-to-owner mutual contact sharing via `share_my_contact_with_match`. It stays `"none"` even when the matched entity is externally sourced with a public contact (e.g. a `Liaison` carrying a LinkedIn URL). To get the actual reachable contact, call `get_match` and read `otherEntityContact.externalLink` / `.email` / `.name`.
- **codemode.get_match({ matchId, entityId })** → full match detail from your entity's perspective. Score / insight / contact fields are plan-gated; gated values come back as an upgrade-message string. When a match is freshly created or has just been re-evaluated, `yourCriteriaScore`/`theirCriteriaScore` may be `null` and inbox-filter answers may show `computed: false`; **scoring runs asynchronously and can take up to 5 minutes**, so tell the user it's still computing rather than treating null as an error, and don't aggressively poll.
- **codemode.view_match_link({ matchId, entityId })** → just the externalLink (or `{ externalLink: null, locked: true }`).
- **codemode.list_match_inbox_filter_answers({ matchId, entityId })** → per-filter answers with `computed`, `answer`, `outdated`. `computed: false` means the workflow has not produced an answer yet.
- **codemode.list_inbox_filters({ entityId })** → filter list with prompt, strictness, outputFormat.
- **codemode.list_plans({ marketId })** → plans with `planId`, name, description, isFree, isCurrentPlan, price (live from Stripe), features, limits.
- **codemode.get_my_credit_balance({ marketId })** → `{ creditsEnabled: false, message }` when the market doesn't charge credits, else `{ creditsEnabled: true, subscriptionBalance, topupBalance, nextFreeGrantAt }`. `subscriptionBalance` refreshes monthly from the plan; `topupBalance` is purchased credits that don't expire.
- **codemode.list_credit_packs({ marketId })** → `{ creditsEnabled, packs: [{ priceId, name, creditsAmount, price?: { amount, currency } }] }`. `price.amount` is in the smallest unit of the currency (e.g. cents for USD).
- **codemode.get_next_match_discovery({ entityId })** → `{ scheduled: false }` or `{ scheduled: true, nextRunAt }` (UTC ISO-8601 of the next automatic discovery for this entity).

All ids are UUIDs. Do not invent or assume endpoints.
