---
name: manage-account
description: Manage the user's Omnimatch account. Use the Omnimatch MCP codemode tool to update entity profile (title, external link, email notifications), update entity criteria with free-text (stored as a scoring file behind the scenes), manage inbox filters (add/update/remove), perform match actions (like, reject, respond, view link, unlock, share contact), and change subscription plan via Stripe billing portal. Destructive actions require confirmed=true. Call codemode with code that invokes codemode.update_entity_*, codemode.add_inbox_filter, codemode.like_match, codemode.reject_match, codemode.respond_to_match, codemode.unlock_match, codemode.share_my_contact_with_match, codemode.change_plan, etc. Use when the user wants to edit their profile, manage filters, act on matches, or change plan. When talking to the user, always use real titles—never show IDs.
---

# Manage Account

Use this skill when the user wants to **change something** in their Omnimatch account: edit a profile, refine search criteria, manage inbox filters, act on a match (like / reject / respond / view link / unlock / share contact), or upgrade/downgrade their plan. The Omnimatch MCP server exposes only a single **codemode** tool ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)) — call it with a `code` parameter (async arrow function body) that runs the appropriate `codemode.<tool>(args)` calls.

**Important:** All entity edits are **text only**. To refine the entity's description or search criteria, use `codemode.update_entity_criteria({ entityId, text })` — the server stores the text as a scoring file internally and the system regenerates the profile from all scoring files. There is no presigned-URL upload flow over MCP. To attach an arbitrary text note (e.g. transcript, kickoff-call notes), use `codemode.add_entity_note({ entityId, name, text })`.

**Confirmation pattern:** Destructive or credit-spending tools (`reject_match`, `unlock_match`, `reevaluate_match`, `remove_entity_files`, `remove_inbox_filter`, `update_entity_title`, `update_entity_external_link`, `respond_to_match` with `direction: "less"`) require `confirmed: true`. The first call returns `{ requiresConfirmation: true, action, details, message }` — show the action to the user and only re-call with `confirmed: true` once they confirm.

**When talking to the user:** Always use **real titles**, never IDs. Refer to entities by their **title**, markets by **market title**, matches by the row's **title** (the other side's name), types by the **entity type title** (e.g. "job", "candidate"), and plans by **plan name**.

## Entity edits

- Title → `codemode.update_entity_title({ entityId, title, confirmed })`.
- External link → `codemode.update_entity_external_link({ entityId, externalLink, confirmed })` (pass `null` to clear).
- Email frequency → `codemode.update_entity_email_notifications({ entityId, frequency: "never" | "best_only" | "all" })`.
- Search criteria refinement → `codemode.update_entity_criteria({ entityId, text })`.
- Add a note → `codemode.add_entity_note({ entityId, name, text })`.
- Remove files → `codemode.remove_entity_files({ entityId, fileKeys, confirmed })` (keys from `get_entity` files[]).

## Inbox filters

Inbox filters are saved free-text questions answered per match. Each filter becomes its own inbox keyed `query:{filterId}`.

- Create → `codemode.add_inbox_filter({ entityId, prompt, strictness: "lenient" | "normal" | "strict" })`. Plan-gated (Match insights). Returns `{ filterId, inboxKey }`.
- Update → `codemode.update_inbox_filter({ entityId, filterId, prompt?, strictness? })`.
- Remove → `codemode.remove_inbox_filter({ entityId, filterId, confirmed })`.

## Match actions

Match actions all take `{ matchId, entityId }` (the match id and your entity id):

- Approve / "more like this" → `codemode.like_match({ matchId, entityId })`.
- Reject / "less like this" → `codemode.reject_match({ matchId, entityId, confirmed })` — confirm before calling.
- Free-text feedback → `codemode.respond_to_match({ matchId, entityId, direction: "more" | "less", textFeedback?, confirmed })` — `direction: "less"` requires confirmation.
- Get just the external link → `codemode.view_match_link({ matchId, entityId })`. Returns `{ externalLink, locked, message? }`.
- Unlock a credit-gated match → `codemode.unlock_match({ matchId, entityId, confirmed })` — deducts one credit; confirm before calling.
- Share contact details with an on-platform match → `codemode.share_my_contact_with_match({ matchId, entityId, name?, email?, externalLink?, confirmed })` — at least one of email/externalLink required.
- Re-evaluate a match's score → `codemode.reevaluate_match({ matchId, entityId, confirmed })` — confirm before calling. Useful after the user updates their profile and wants the score to reflect the new context. Costs one re-evaluate credit on credit-enabled markets; fails if either side's profile isn't ready (missing scoring files, no extracted criteria, or marked outdated). The re-evaluation runs asynchronously and **can take up to 5 minutes** to complete; while it's pending, `get_match` may show `yourCriteriaScore`/`theirCriteriaScore` as `null` and inbox-filter answers as `computed: false`. Tell the user up front so they aren't surprised, and don't aggressively poll — wait at least a few minutes between checks.

## Subscription / plan changes

- List plans → `codemode.list_plans({ marketId })` for plan name, description, isFree, isCurrentPlan, price (live from Stripe), features, limits.
- Change plan → `codemode.change_plan({ marketId, targetPlanId })` returns `{ billingPortalUrl, currentPlanId, targetPlanId, message }`. Open the URL — Stripe handles upgrade/downgrade/cancellation. No confirmed flag needed; Stripe is the confirmation surface.

## Workflow patterns

- **"Reject this match"** — call `reject_match` first without `confirmed`; show the preview to the user; if they confirm, call again with `confirmed: true`.
- **"Unlock that match"** — same pattern; warn that it costs one credit before re-calling with `confirmed: true`.
- **"Add a filter for senior roles"** — `codemode.add_inbox_filter({ entityId, prompt: "Is this a senior role?", strictness: "normal" })`. After it returns the filterId, tell the user the filter has been queued for output-format inference and will appear in the inbox once matches are evaluated.
- **"What does my filter say about this match"** — `codemode.list_match_inbox_filter_answers({ matchId, entityId })` (read-only — also available in account-info).
- **"Upgrade to the Pro plan"** — `codemode.list_plans({ marketId })` to find the plan with name "Pro" and its planId; then `codemode.change_plan({ marketId, targetPlanId })`. Hand the user the `billingPortalUrl`.
