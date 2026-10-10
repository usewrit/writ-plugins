---
name: writ-payments
description: Buy something on a website for the user with Writ, paid with one of their cards or the card saved on the store, without a card number ever reaching the chat. Use when the user asks to buy, order, pay, check out, book and pay, or reorder something now, or to make an automation that buys (for example when a price drops), or asks which cards Writ can pay with. Writ pauses at the order button until the user confirms (email, the Writ app or Writ Desktop) unless a rule they set covers it, then Writ places the order itself.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Paying for things with Writ

You never see, type, store or ask for a card number, expiry date or security code, and you never click the button that places an order. Writ holds the cards; the user approves which card, and confirms the order. Your job is to get to the checkout and hand it over with `writ_payment`.

Never ask for card details in chat. If the user offers them, do not use them: say Writ keeps cards in the user's own vault, and send the card request link instead.

## Which path

| The ask | Path |
| --- | --- |
| Buy this now ("order two of these", "pay my bill", "buy it") | Live purchase, below |
| Buy later when something happens ("buy it when it drops under $80") | An automation that buys, built as a ladder (the `writ-watch-and-schedule` skill, step "Buy when the price drops"): the monitor, a recorded checkout up to the order page, one rehearsal run, then the monitor wired to that checkout with `buy`; an AI purchase session (`action: "ai_task"` with `buy`) only when the checkout cannot be recorded or its rehearsal fails |
| "Which cards can you use?" | `writ_payment` `action: "list"`: handles only (kind, brand, last four digits, id) |

## Live purchase

1. **Sign in if the store needs the account:** `writ_personas` `action: "list"` with the store's domain first (the `writ-signed-in-sites` skill). The store's saved card and address come with the account.
2. **Choose how it is paid.** The card saved on the store account needs no card request. A Writ card: `writ_payment` `action: "request"` with the store's `site`, a `max_amount` (the most the order may cost, tax and shipping included), the `currency` the store charges in (`"eur"`, `"usd"`, `"gbp"`...), a one-line `purpose` the user reads, the `session` once one is open, and for a card form `fields` (card field to CSS selector: `number`, `exp`, `cvc`, `name`, `zip`; a field inside a payment provider's frame is `{"selector", "frame_url"}`, for example `"js.stripe.com"`). Relay `tell_user` and `open_url`, then `action: "wait"` with the `grant_id` (one held call, up to 60 s; call again on `still_waiting`). You get the card's brand and last four digits, nothing more.
3. **Browse to the final order page** in a Writ browser (`writ_browser_use`, then `writ_browser_act`): options, quantity, cart, shipping. Do not press the order button: it is refused with `use_writ_payment_checkout`, and so is a request to an order endpoint.
4. **Check out:** `writ_payment` `action: "checkout"` with the `session`, `commit_selector` (the place-order button), `total_selector` (where the page shows the order total), and `grant_id`, or `max_amount` and `currency` for the store's saved card, plus a one-line `summary`. With no `currency`, Writ takes the one the page's total shows. For a card page followed by a review page, pass `steps`: `[{"type": "fill_card"}, {"type": "click", "selector": "#continue"}]`; Writ runs them after the confirmation, before the order button.
   - Nothing is typed or clicked yet. The session pauses: until the purchase is settled, every action on it answers `checkout_pending`.
   - Relay `tell_user`. The user confirms from the emailed link, from the Writ app or from Writ Desktop, with their second factor.
   - If a rule the user turned on covers it (their sites, a maximum per purchase, period totals), it goes through at once. Rules apply only when the total could be read, so always pass `total_selector` when the page shows a total.
5. **Read the outcome:** `writ_payment` `action: "wait"` with the `confirmation_id`. Writ typed the card, checked that the total was at most the maximum, and clicked the order button itself. `executed` = placed; `declined` or `expired` = nothing bought; `failed` = nothing bought, unless the result says the order button was clicked (then ask the user to check their orders on the store).
6. **Close the browser** with `writ_browser_cancel` (or save the recording if the purchase will come back as an automation).

`action: "fill"` types an approved card before the checkout: only for a virtual card whose issuer checks every charge in real time (otherwise it answers `fill_at_checkout`; use `steps` in the checkout instead). A grant pays once and expires 15 minutes after it is approved if unused.

## What the user controls

- **Each card pays in one currency** and its limits are counted in it: a purchase in another currency is refused, and nothing is ever converted. Ask for the card in the currency the store charges in.
- **Each card has rules** in Writ (Payment methods): the sites it may pay on, a limit per purchase and per day, week or month, which automations may use it, whether an assistant may buy with it alone and up to how much, and when a purchase needs their approval. Writ checks these before the card is ever used; a refusal names the rule (`card_site_not_allowed`, `card_purchase_limit`, `card_period_limit`, `card_ai_cannot_arm`, `card_automation_not_allowed`). Tell the user which rule and that they can change it in the Writ app; never try another card to get around it.
- **Auto-confirm rules** (Payment methods, Auto-confirm) let some live purchases skip the confirmation, in the rule's currency only. A rule covers an automation's purchases only when the user ticked automations in its assistants, and never when the automation or the card asks for a person's approval. Only the user creates or loosens one, with their second factor. Never suggest turning confirmation off to save time.
- **Automation purchases** stay rehearsals until the user turns purchases on in the Writ app. Once on, each purchase waits for the user's confirmation (the email, the Writ app or Writ Desktop) unless an auto-confirm rule covers it, and the confirmation names the automation and the monitor that asked.
- **Purchases need the Pro plan or higher** (`plan_required`), and an account may have purchases paused (`autobuy_disabled`). Relay it; do not retry.

## Refusals you may see

| Code | Meaning | What to do |
| --- | --- | --- |
| `use_writ_payment_checkout` | You clicked or posted something that places an order | Call `action: "checkout"` instead |
| `checkout_pending` | The session waits for the user's confirmation | `action: "wait"` with the `confirmation_id` |
| `grant_expired`, `grant_not_approved` | The card request is not usable | `action: "request"` again |
| `grant_other_site`, `grant_other_session` | The card was approved for another store or session | Ask again for this store and session |
| `total_above_max`, `total_unreadable` | The total is too high, or Writ could not read it | Nothing was bought. Tell the user the total; pass the right `total_selector` |
| `total_selector_required` | This card needs the total checked | Pass `total_selector` |
| `card_currency_mismatch` | The card pays in another currency than the purchase | Ask the user for a card in the store's currency |
| `total_currency_mismatch`, `currency_mismatch` | The page charges in another currency than the one asked for | Nothing was bought. Request again with the store's `currency` |
| `device_card_live_unsupported` | A card saved on a desktop only pays in automations on that desktop | Ask for another card |
