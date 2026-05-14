# Omnimatch Plugin

A Claude Code plugin that connects to the **Omnimatch MCP** server so you can manage your account via MCP. List and view your entities and matches; update entity profiles and files; approve or reject matches; and check subscription plans. Requires a logged-in user (no admin role). Intended for public use so users can manage their Omnimatch account from any MCP client.

## Installation

### Claude Code (Omnimatch plugin marketplace)

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
- `/mcp` — complete OAuth so Claude Code holds a Bearer token for the Omnimatch MCP server.

### Claude Code (TalentScore plugins catalog)

```bash
claude plugin install omnimatch@talentscore-plugins --scope project
```

### Cursor

Open the plugin marketplace in Cursor and install **Omnimatch** from this repository, or add an MCP server manually to your `mcp.json`. Production default:

```json
{
  "mcpServers": {
    "omnimatch": {
      "url": "https://omnimatch.ai/api/mcp"
    }
  }
}
```

For local development against a running worker, set `"url": "http://localhost:8787/api/mcp"` (or the same host and port your Worker uses).

## Configuration

- **MCP URL**: By default the plugin connects to `http://localhost:8787/api/mcp`. Override with the `OMNIMATCH_MCP_URL` environment variable (e.g. for a remote or different port).
- **Authentication**: The Omnimatch MCP requires user OAuth. In Claude Code, use `/mcp` and authenticate when prompted so the client can send a Bearer token to the server. Any authenticated user can use this server (unlike the Market Manager MCP, which is admin-only).

## Prerequisites

1. **Worker running**: Start the Omnimatch worker so the MCP endpoint is available:

   ```bash
   npm run dev
   ```

   `npm run dev` starts the worker and Vite together. MCP is served at `http://localhost:8787/api/mcp`.

2. **User account**: You must be logged in (Better Auth). The MCP uses your session/token to scope all tools to your entities and matches.

## MCP: Codemode only

The Omnimatch MCP server exposes **only the codemode tool** ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)). Codemode lets the LLM write and run JavaScript in a secure sandbox; underlying operations are invoked inside that sandbox via `codemode.<toolName>(args)`.

**How to use:** Call the **codemode** tool with a `code` parameter: a string that is the **body** of an async arrow function. Inside the function, call `codemode.list_my_entities()`, `codemode.get_entity({ entityId })`, etc., and return the result. Example:

```js
async () => {
  const list = await codemode.list_my_entities();
  return list;
}
```

Chaining multiple calls in one round-trip:

```js
async () => {
  const list = await codemode.list_my_entities();
  if (list.length === 0) return { message: "No entities" };
  const first = list[0]?.entities?.[0];
  if (!first) return list;
  const details = await codemode.get_entity({ entityId: first.id });
  return { list, details };
}
```

**Operations available inside codemode** (all scoped to the authenticated user):

| Operation | Description |
|-----------|-------------|
| `codemode.list_my_entities()` | List the current user's entities across markets. |
| `codemode.get_entity({ entityId })` | Get entity details: **criteria**, title, full description, type, attached files (key, name, summary), match insight questions, and entity query answers if the plan allows. |
| `codemode.create_entity({ marketId, entityTypeTitle, title, externalLink?, initialCriteriaText?, confirmed? })` | Create a new public entity the caller will own. If the caller already has a User Profile on file, it is **auto-pinned** as a scoring file so scoring + match-insight answers see their LinkedIn/CV. Returns `userProfilePinned: boolean` — if `false`, drive the LinkedIn flow (next two rows) for entity types whose onboarding asks for a personal profile. Requires confirmation. |
| `codemode.collect_user_profile_from_linkedin({ linkedinUrl })` | First-time User Profile setup over MCP. Ask the user for their LinkedIn URL, then call this — it pulls their public profile from CoreSignal, stages the JSON in R2, and returns a preview (name, headline, currentRole, locationCity) plus `stagedR2Key`. Does **not** persist yet. Rate-limited to 3 attempts per 24h. |
| `codemode.confirm_user_profile({ stagedR2Key, contentType, linkedinSlug?, coresignalEmployeeId?, coresignalChangedAt?, entityId? })` | After the user confirms the LinkedIn preview is them, call this to persist + (optionally) pin to the entity in one shot. Pass the fields back from `collect_user_profile_from_linkedin` plus the `entityId` you're onboarding. From then on every new `create_entity` auto-pins. |
| `codemode.create_private_entity({ marketId, entityTypeTitle, title, externalLink?, initialCriteriaText?, confirmed? })` | Create a private scoring entity from a link or text. Requires confirmation. |
| `codemode.get_private_entity_file_upload_url({ marketId, contentType, name })` | Get a one-time upload URL for creating a private scoring entity from a file. After PUT to `uploadUrl`, call `codemode.enqueue_private_entity_file_upload`. |
| `codemode.enqueue_private_entity_file_upload({ marketId, entityTypeTitle, title, stagedKey, contentType, name, confirmed? })` | Create a private scoring entity from a staged uploaded file. Requires confirmation; attaches the file privately and queues profile/scoring workflows. |
| `codemode.update_entity_title({ entityId, title, confirmed? })` / `codemode.update_entity_external_link({ entityId, externalLink, confirmed? })` / `codemode.update_entity_email_notifications({ entityId, frequency })` / `codemode.update_entity_criteria({ entityId, text })` | Update allowed profile fields or add free-text criteria. |
| `codemode.list_inboxes({ entityId })` / `codemode.list_inbox_matches({ entityId, inboxKey, ... })` | List inboxes and matches. Match rows include scores, status, contact state, and locked state. |
| `codemode.get_match({ matchId, entityId })` | Full details for a match: other entity name/type, match insights, suitability reports, scores, contact, and credit-lock state. |
| `codemode.list_plans({ marketId })` | Subscription plans for the market. Use when the user needs to upgrade for gated features. |
| `codemode.like_match({ matchId, entityId })` / `codemode.reject_match({ matchId, entityId, confirmed? })` / `codemode.respond_to_match({ matchId, entityId, direction, textFeedback?, confirmed? })` | Approve, reject, or give feedback on a match. |

Use natural language (e.g. “list my entities”, “update my job profile”, “approve match X”) ; the assistant will call the codemode tool with the right code.

## User Profile (LinkedIn / CV) onboarding

The **User Profile** is a single LinkedIn/CV blob stored per user that the scoring pipeline reads alongside the entity's own files. It's how the matcher sees who the caller is — without it, every "do we share an X?" insight question comes back blank for the user's side.

**`create_entity` is the entry point.** It returns `userProfilePinned: boolean`:
- `true` → the user already has a profile on file and it's been pinned to this new entity. Skip any "tell us about yourself" step in the returned onboarding questions.
- `false` → first time setup. If the entity type asks for a personal profile, drive the LinkedIn flow before falling back to text:
  1. Ask the user for their LinkedIn URL (or just their slug).
  2. Call `collect_user_profile_from_linkedin({ linkedinUrl })`. Show the returned preview (`name`, `headline`, `currentRole`, `locationCity`) to the user.
  3. On confirmation, call `confirm_user_profile({ ...staging fields, entityId })` — this persists the profile and pins it to the entity in one shot.
  4. If the user has no LinkedIn, the lookup returns no match, or the rate limit (3 collects per 24h) hits, fall back to `update_entity_criteria({ entityId, text })` with whatever description they paste.

After the first `confirm_user_profile`, every subsequent `create_entity` returns `userProfilePinned: true` automatically — no need to re-ask. The MCP cannot accept binary file uploads (no CV file path), so LinkedIn-or-pasted-text are the only two paths over MCP.

Backfilling pre-existing entities (created before this flow shipped) is done by an admin from the web app's admin Entity overview page, not over MCP.

## Match data gotcha: `contact` vs `otherEntityContact`

The `contact` column on `list_inbox_matches` rows is **not** the contact info. It is a tri-state — `"shared" | "pending" | "none"` — that only tracks **owner-to-owner mutual contact sharing** on the platform: it goes to `"shared"` once both sides have called `share_my_contact_with_match`. It stays at `"none"` whenever the other side is an externally-sourced entity (system-created, no human owner to share back), even though that entity's public contact is already embedded in the entity itself.

To get the actual reachable contact, call `get_match({ matchId, entityId })` and read `otherEntityContact`:

```ts
{
  otherEntityContact: {
    name: string | null,
    email: string | null,
    externalLink: string | null, // public profile URL for externally-sourced entities
  }
}
```

For externally-sourced entity types, `externalLink` is the operational contact (public profile / homepage) regardless of the `contact` column. Do **not** report "no contact available" based on `contact: "none"` alone — fetch `get_match.otherEntityContact` first.

`share_my_contact_with_match` is only meaningful for owner-to-owner matches. Calling it on an externally-sourced match is pointless: there's no owner on the other side to receive the share, and the public contact is already available.

## Linking back to the Omnimatch web app

The MCP returns IDs only — it does **not** return on-platform URLs for entities, matches, or markets. If a user asks for a clickable link (e.g. "open this match in the web app", "give me a URL I can paste into a doc"), construct it from the user's `ownerId` for that market plus the entity/match ID. The `ownerId` for a given market comes back from `codemode.join_market({ marketId })` (also when already joined — it returns the existing ownerId).

Base URL: `https://omnimatch.ai` (production). For non-prod, swap the host but keep the path.

| Resource | URL pattern |
|---|---|
| Entity you own | `/user/owners/{ownerId}/entities/{entityId}` |
| Match (connection) on your entity | `/user/owners/{ownerId}/entities/{entityId}/connections/{matchId}` |
| Owner home (your owner record in a market) | `/user/owners/{ownerId}` |

Do **not** synthesize `/admin/...` URLs — those work only for admin users and 404 (or redirect) for everyone else. The MCP currently has no admin scope, so any caller of this plugin should be treated as a non-admin and routed to the `/user/...` paths above.

If the user has not yet joined the market, you don't have an `ownerId` and cannot produce a per-entity URL — call `join_market` first or send them to `https://omnimatch.ai` to log in and join.

## Discovering a market's entity types

A market's entity types are not exposed as a dedicated MCP tool. To discover them without spamming `create_entity` with guesses, call `codemode.join_market({ marketId })` — the response includes an `onboarding.entityTypes` array (or, if already joined, `onboarding.instructions` enumerates the available types). Use that to pick the right `entityTypeTitle` for `create_entity`. Each type has an `externalOnly` flag: `true` means only the matching system can create that type (you'll see them on the other side of your matches but cannot create one yourself).

## Asking the user

For skills that ask the user questions (e.g. level-up: which profile, extra info, confirmation before applying), use the **AskUserQuestion** tool in Claude Code or the **AskQuestion** tool in Cursor so the user gets clear options and can respond in a structured way. If the tool is not available (e.g. headless environment), ask in chat with the same structure.

## Skills

| Skill | Description |
|-------|--------------|
| `account-info` | Read-only: list your entities, get entity details, list files, list matches, get match details, get subscription plans. Use when the user asks to view their data. Use entity type titles and instance names from the app, not the word "entity". |
| `account-briefing` | Read-only overview: your entities plus their top recent matches in one summary. Use when the user asks for a briefing, "what's new", "catch me up", or "how are my matches". Does not mutate data. |
| `manage-account` | Write: update entity profile, create private scoring entities from text/link/file uploads, approve or reject matches. Use when the user wants to edit their profile, upload private opportunities/profiles, manage filters, or respond to matches. |
| `level-up` | Improve a profile: pick which one to level up, review matches (scores 0–100). If no decent matches (both >40%), ask for more info; if disparities, ask a few questions about uncertainties; if strong matches (both >70%) unapproved, ask if they want to approve or what we got wrong. Propose profile updates, confirm with the user, then apply. Use when the user wants to "level up", get better matches, or refine what the AI knows. Use type titles and instance names, not "entity". |
| `subscription` | View and change plans (Stripe Billing Portal), check credit balance, list credit packs, and start Stripe Checkout to top up credits. |

## Example Workflows

### List my entities

```
List my entities
```

### View an entity

```
Show details for my entity <entity-id>
```

### List matches

```
List matches for my entity <entity-id>
```

### Update profile

You can only update title, external link, email notifications, and match insight questions. Description and criteria are regenerated when files are added or removed.

```
Update entity <entity-id>: set title to "Senior React Developer"
```

```
Update entity <entity-id>: set link to https://example.com/profile and email notifications to best_only
```

### Upload a Private Scoring Entity from a File

1. Call `get_private_entity_file_upload_url` with `marketId`, `contentType`, and `name`.
2. Upload the file with PUT to the returned `uploadUrl`.
3. Call `enqueue_private_entity_file_upload` with `marketId`, `entityTypeTitle`, `title`, and the returned `stagedKey`, `contentType`, and `name`.

### Respond to a match

```
Approve match <match-id>
```

```
Reject match <match-id>
```

### Check plans

```
What subscription plans are available for my entity <entity-id>?
```

## Troubleshooting

- **HTTP 401** — Not authenticated. Use `/mcp` in Claude Code and complete OAuth so the client sends a Bearer token.
- **HTTP 404 / HTML response** — Request is not reaching the Worker. Start `npm run dev` and ensure the MCP URL points at the Worker (e.g. `http://127.0.0.1:8787/api/mcp`).
- **Entity not found or access denied** — The entity or match is not owned by the current user. Only your own entities and matches are visible and editable.
- **Match insights / suitability gated** — Use the `subscription` skill or `codemode.list_plans({ marketId })` for the upgrade link and plan details.
