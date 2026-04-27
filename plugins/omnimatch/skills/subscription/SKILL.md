---
name: subscription
description: View and change the user's subscription plan or credit balance in a market. Use the Omnimatch MCP codemode tool to list plans, switch plans via Stripe Billing Portal, check the owner's credit balance, list one-time credit packs, and start a Stripe checkout to top up credits. Call codemode with code that invokes codemode.list_plans({ marketId }), codemode.change_plan({ marketId, targetPlanId }), codemode.get_my_credit_balance({ marketId }), codemode.list_credit_packs({ marketId }), and codemode.topup_credits({ marketId, priceId }). Use when the user asks about their plan, pricing, upgrading, downgrading, cancelling, their credit balance, or buying more credits. When talking to the user, always use plan name and pack name with plain pricing—never show Stripe price ids.
---

# Subscription

Use this skill when the user asks about their **subscription plan**, **pricing**, **features**, **upgrading**, **downgrading**, **cancelling**, their **credit balance**, or **buying more credits**. The Omnimatch MCP server exposes a single **codemode** tool ([Cloudflare Codemode](https://developers.cloudflare.com/agents/api-reference/codemode/)). Call it with a `code` parameter (async arrow function body) that runs the appropriate `codemode.<tool>(args)` calls.

Both plan changes and credit top-ups go through Stripe in the user's browser — `change_plan` returns a **Billing Portal** URL, and `topup_credits` returns a one-time **Checkout Session** URL. Stripe is the confirmation surface for both; do not add our own confirmation prompt.

## When to use

- User asks **what plans are there?** / **show me my plan options** → `codemode.list_plans({ marketId })`.
- User asks **what plan am I on?** → `codemode.list_plans({ marketId })` and look for `isCurrentPlan: true`.
- User asks **how much does X cost?** / **what features do I get with the Pro plan?** → `codemode.list_plans({ marketId })` and read the `price`, `features`, `limits` for that plan.
- User asks **upgrade to Pro** / **downgrade to free** / **cancel my subscription** → `codemode.list_plans({ marketId })` to find the targetPlanId, then `codemode.change_plan({ marketId, targetPlanId })` to get the Stripe portal link, and hand the URL to the user.
- User asks **how many credits do I have?** / **what's my balance?** → `codemode.get_my_credit_balance({ marketId })`. For markets that don't charge credits, the tool returns `creditsEnabled: false`—say so briefly.
- User asks **what credit packs can I buy?** / **how do I top up?** → `codemode.list_credit_packs({ marketId })`. Present packs by name and credit amount.
- User asks **buy {pack name}** / **top me up** → `codemode.topup_credits({ marketId, priceId })` and hand the user the `checkoutUrl`.

## Workflow

1. Pick the marketId. If you don't have one, call `codemode.list_my_markets()` and ask the user (via AskUserQuestion / chat) which market.
2. `codemode.list_plans({ marketId })` returns:
   ```
   {
     plans: [
       {
         planId: "free" | "price_xxxx",
         name: "Free" | "Pro" | ...,
         description: "...",
         isFree: true | false,
         isCurrentPlan: true | false,
         price: { amount, currency } | null,
         freeTrial: { days } | null,
         features: ["..."],
         ctaText, badge, footnote,
         limits: { ... }
       }
     ],
     message: "..."
   }
   ```
   Present plans **by name** (e.g. "Free", "Pro", "Enterprise"). Convert `price.amount` from the smallest unit to a normal currency string (e.g. `1999` `usd` → `$19.99`). Mention the current plan with a note like "(your current plan)".
3. To change plans, find the `planId` for the chosen plan and call:
   ```
   codemode.change_plan({ marketId, targetPlanId })
   ```
   Returns `{ billingPortalUrl, currentPlanId, targetPlanId, message }`.
4. Hand the `billingPortalUrl` to the user. Tell them to open it in a browser and confirm there. Stripe handles upgrades, downgrades, and cancellation in one portal UI.

## Credit balance and top-ups

Some markets charge credits per match-view, manual search, or re-evaluation; a plan grants a monthly subscription allotment, and owners can buy one-time top-up packs that don't expire.

1. **Check balance** — `codemode.get_my_credit_balance({ marketId })`:
   ```
   { creditsEnabled: false, message }
   // or
   { creditsEnabled: true, subscriptionBalance, topupBalance, nextFreeGrantAt }
   ```
   `subscriptionBalance` resets monthly from the plan; `topupBalance` is purchased credits that never expire. Subscription credits are spent before top-ups. If `creditsEnabled: false`, tell the user briefly that this market doesn't charge credits and stop.

2. **List packs** — `codemode.list_credit_packs({ marketId })`:
   ```
   {
     creditsEnabled: true,
     packs: [{ priceId, name, creditsAmount, price: { amount, currency } | null }]
   }
   ```
   Convert `price.amount` from the smallest unit to a normal currency string. Always present packs by **name and creditsAmount**, never the priceId.

3. **Buy a pack** — `codemode.topup_credits({ marketId, priceId })` returns `{ checkoutUrl, packName, creditsAmount, message }`. Hand `checkoutUrl` to the user; once they pay, Stripe redirects back and the app credits their `topupBalance` automatically. Stripe is the confirmation surface — don't add our own confirmation prompt.

## Important

- **Never expose Stripe price ids in user-facing copy.** Use plan names ("Pro", "Free") and pack names instead.
- The `planId` is what you pass to `change_plan` — for free, it's the literal string `"free"`; for paid, it's the Stripe price id from `list_plans`. The `priceId` you pass to `topup_credits` is the Stripe price id from `list_credit_packs`. The user only sees plan/pack names.
- Live prices come from Stripe; small differences may exist if Stripe is unreachable. If `price` is null for a paid plan or pack, say so and recommend retrying or checking the billing portal.
- Don't try to "upgrade silently" — always present the plan or pack and let the user open the Stripe surface.
