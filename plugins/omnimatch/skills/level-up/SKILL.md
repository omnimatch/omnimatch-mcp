---
name: level-up
description: Level up a profile by finding the entity to improve, reviewing its inboxes, and either gathering more info (if no decent matches), clarifying uncertainties (if score disparities), or discussing unapproved strong matches—then proposing profile updates and applying them. Use AskUserQuestion (Claude Code) or AskQuestion (Cursor) for all questions to the user. Use when the user wants to "level up", improve their profile, get better matches, or refine what the AI knows about them. When talking to the user, always use real titles—never show IDs.
---

# Level Up (Improve Your Profile)

Use this skill when the user wants to **improve their profile** so they get better matches. You will: (1) find which profile to level up, (2) review the inboxes and matches, (3) have a short, friendly conversation to gather information, then (4) propose concrete profile changes, **get explicit confirmation**, and only then apply updates. Use **entity type titles** (e.g. "job", "candidate") and **profile titles** in all messages — never raw IDs.

**Asking the user:** Use **AskUserQuestion** (Claude Code) / **AskQuestion** (Cursor) for all questions. Ask one or a few at a time.

## When to use

- User says **level up**, **improve my profile**, **get better matches**, **refine my profile**, or **help the AI understand what I want**.

## Workflow

### 1. Choose which profile to level up

- `codemode.list_my_markets()` and `codemode.list_my_entities()` to enumerate the user's profiles.
- If multiple, **AskUserQuestion** to ask which one — offer options using their **type** and **title**.
- `codemode.get_entity({ entityId })` for the chosen profile (criteria, title, type, description, files, inbox filters).

### 2. Load matches via inboxes

- `codemode.list_inboxes({ entityId })` to see counts in priority / liked / all and any filter inboxes.
- `codemode.list_inbox_matches({ entityId, inboxKey: "all", limit: 50 })` to see the spread of matches.
- Scores are 0–100 in both directions: **yourScore** (how well the match fits what you want) and **theirScore** (how well you fit what they want).
- Classify:
  - **Decent**: both scores > 40.
  - **Good**: both scores > 70.

### 3a. No decent matches (no row with both scores > 40)

- Tell them in plain language that there aren't any matches yet where both sides score above 40%, so the system needs **more information** to find better fits.
- **AskUserQuestion** for a few concrete details — must-haves, deal-breakers, preferences, what's missing. Offer options + custom input. One or two questions per call.
- Use answers to propose **additions**: a free-text criteria refinement (`codemode.update_entity_criteria`), or a new inbox filter (`codemode.add_inbox_filter`), or title/link/email preference updates. **Do not apply yet** — confirm first.

### 3b. There are matches but want to reduce uncertainty (big score gaps)

- Focus on rows where there's a **big gap** between yourScore and theirScore. Prioritise rows where **theirScore is lower** than yours.
- For one or two of these, run `codemode.get_match({ matchId, entityId })` and `codemode.list_match_inbox_filter_answers({ matchId, entityId })` to see what the system is uncertain about.
- **AskUserQuestion** about those uncertainties — friendly, structured, one or two at a time.
- Use answers to propose refinements (criteria text, new inbox filter, etc.). Do not apply yet.

### 3c. Strong matches (both > 70) not yet approved

- Identify rows where both scores > 70 and `status` is `"pending"` (not approved, not rejected).
- **AskUserQuestion**: "You have strong matches not yet acted on — do you want to like any of them? Or do you think the score is wrong?" Offer options + custom input.
- Only call `codemode.like_match` or `codemode.respond_to_match` when the user explicitly asks.

### 4. Propose profile updates and confirm before applying

- Summarize the proposed changes (new or edited filters, criteria refinements, title / link / email preference updates, file removals).
- **AskUserQuestion** to confirm explicitly before writing.
- Only after the user agrees:
  - `codemode.update_entity_criteria({ entityId, text })` for criteria refinements.
  - `codemode.add_inbox_filter({ entityId, prompt, strictness })` / `codemode.update_inbox_filter` / `codemode.remove_inbox_filter`.
  - `codemode.update_entity_title` / `codemode.update_entity_external_link` / `codemode.update_entity_email_notifications`.
  - `codemode.remove_entity_files({ entityId, fileKeys, confirmed: true })` if removing files.
- Destructive ops require `confirmed: true`; the first call returns a preview describing the action — show it to the user, then re-call with `confirmed: true`.

### 5. Closing message

- Tell them updating their profile will improve future searches.
- Suggest checking their email for match updates.
- Encourage coming back to level up again when they have new info.

## Tool usage

- **AskUserQuestion** / **AskQuestion**: every question to the user.
- **Read** via codemode: `list_my_markets`, `list_my_entities`, `get_entity`, `list_inboxes`, `list_inbox_matches`, `get_match`, `list_match_inbox_filter_answers`, `list_inbox_filters`, `list_plans`.
- **Write** via codemode (only after confirmation): `update_entity_criteria`, `add_entity_note`, `update_entity_title` (confirmed), `update_entity_external_link` (confirmed), `update_entity_email_notifications`, `remove_entity_files` (confirmed), `add_inbox_filter`, `update_inbox_filter`, `remove_inbox_filter` (confirmed), and only when the user explicitly asks: `like_match`, `reject_match` (confirmed), `respond_to_match` (confirmed if direction="less"), `share_my_contact_with_match` (confirmed), `unlock_match` (confirmed), `change_plan`.

When talking to the user, always use real titles — never IDs.
