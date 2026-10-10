---
name: writ-watch-and-schedule
description: Watch web pages for changes with Writ monitors and run saved workflows or crawls on a schedule. Use when the user wants to be told when a price, listing, stock level or status changes, wants a page checked every few minutes, or wants data pulled every morning or every hour. What happens after a change or a scheduled run (notifications, chained workflows, digests, an AI agent) is the writ-automations skill.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Watch pages and run jobs on a schedule

| The user wants | Build |
| --- | --- |
| "Tell me when this page, price or status changes" | a monitor: `writ_create_monitor`, then `writ_wire_monitor` |
| "Every morning, get me X" | a saved workflow, then `writ_set_schedule` (a crawl: a scheduled automation, below) |
| "…and email it to me", "when A finishes, run B", "alert me when a run fails" | `writ_create_automation` (the `writ-automations` skill) |

## Watch a page: monitors

1. **Prove the selector on the live page.** Open it with `writ_browser_use` and pass that `session_id` to `writ_create_monitor`: the selector is checked before saving. A selector that matches several elements is pinned to the one shown; one that matches nothing becomes a visual-zone watch.
2. **Create it:** `writ_create_monitor` with `url`, `selector` and `interval` (`"15m"`, `"1h"`, or seconds; `interval_minutes` still works).
   - No interval given: nothing is created. The answer is `needs_input` with the plan's options (checks per day, how long the allowance lasts, which need a higher plan) and a `tell_user` line: relay it, let the user pick, and call again with their `interval`. Don't pick one for them.
   - The page is checked first. A public page needs no account. `needs_persona` means the value only shows after sign-in: follow the persona flow in `writ-signed-in-sites`. A bot check answers `needs_input` with options (a residential connection when the plan pays for it; otherwise Writ Desktop on the user's own computer, an upgrade, credits, or `try_anyway`): relay them, don't choose.
   - A price: `watch: "price"` (rejects a selector whose text holds no number).
   - No selector possible: `mode: "visual"` with `zone_text` exactly as the page prints the value (for example `"51,77 EUR"`). A zone fires on any visual change, so wire a change alert, not a threshold.
   - A value that lives in a JSON or XHR response: `extract` (for example `{"from": "json", "path": "data.price"}`) with `request_url`, and no browser at all.
   - JavaScript-rendered pages: `requires_browser: true`. Behind a login: `persona_id`. Datacenter-blocked sites: `use_residential: true`.
   - An interval the plan doesn't allow is refused (`status: "refused"`, with the allowed options), not clamped: offer those to the user. One that is allowed but uses up the allowance early is created with a `warning`: pass it on.
3. **Make it do something:** `writ_wire_monitor` with the `monitor_id` and an `action`:
   - `notify`: `channels` (for example `["email"]`) and a `message` template (`{{event.url}}`). Omitting `recipients` reaches every enabled recipient on the channel; the answer names who it reaches.
   - `workflow`: run a saved workflow on each change.
   - `ai_task`: wake an AI agent with a `prompt` ("check whether the price dropped below $500 and summarize"). It opens the page with the change and its values. `cooldown_minutes` limits how often it wakes.
   - A price watch takes `threshold`: the action then runs once, when a check reads a price at or below it, not on every change. `threshold_op` picks the side: `lte` (at or below) is the default, `lt` is strictly below ("under", "below", "less than"), `gte` and `gt` watch for a rise. Pass what the user's words say. Omit `threshold` for an action on any change.
4. **Buy when the price drops (auto-buy).** Build it as a ladder, in this order:
   1. **The monitor** that reads the price (step 2, with `watch: "price"`).
   2. **Record the checkout** up to the final order page: `writ_record_website` (or `writ_browser_use`), then `writ_browser_save`. Make the product, quantity and shipping choices inputs (`{{name}}` + `inputs`) where they vary, add an `extract` of the order total, and never press the order button: Writ refuses it (`use_writ_payment_checkout`) because only Writ places an order. Note that button's CSS selector instead: it becomes `commit_selector` in step 4. Behind a login, use the store's persona (the `writ-signed-in-sites` skill).
   3. **Rehearse it once:** `writ_run_workflow` with `mutation_mode: "dry_run"`. The run must end on the order page and return the total. Rehearse only a checkout recorded this way: one recorded elsewhere may include the order click.
   4. **Wire it:** `writ_wire_monitor` with `action: "workflow"`, that workflow, the `threshold` (and `threshold_op`), and `buy`:
      - `payment`: `{"kind", "ref"}`. A card's handle comes from `writ_payment` (`action: "list"`, or the `payment` of an approved grant); `{"kind": "merchant_saved"}` uses the card saved on the store account. Never ask for a card number in chat.
      - `total_selector`: where the checkout shows its total, so an order above the allowed amount is stopped.
      - `commit_selector`: the place-order button you noted while recording. Writ clicks it itself, only in a live buy, right after checking the total; a rehearsal stops before it. A live buy whose recording holds no order step and has no `commit_selector` is refused, unless `fallback_ai_session` is on.
      - `fallback_ai_session: true`: if the recorded checkout fails when it fires, Writ wakes one AI purchase session with the same limits.
      - Optional: `require_confirmation` (default on), `confirm_over`, `spend_cap`, `price_buffer_pct`.
   5. **Only if step 2 or 3 fails** (a bot wall, a checkout that will not replay, a login the persona cannot keep): `writ_wire_monitor` with `action: "ai_task"`, a `prompt` saying exactly what to buy (product, quantity, options, shipping), the `threshold`, and the same `buy` with `max_amount` set inside it (`buy.max_amount`: the most one order may cost). When the price is reached, an AI purchase session browses to the order page and hands over to Writ's confirmed checkout. It cannot pay with a desktop card (`device_card`).
   6. **Tell the user** which path you took and why (a recorded checkout, or an AI purchase session because the recording or its rehearsal failed), that it is saved as a rehearsal until they turn purchases on in the Writ app, and that a person confirms each purchase unless an auto-confirm rule they set covers automation purchases.
   - Purchases run on Writ Cloud only: a buy on a desktop monitor (`device`) is refused. The card's own rules (sites, limits, whether an automation may buy with it) still apply when it fires. A buy the user wants made now is a live purchase instead (the `writ-payments` skill).

## Run on a clock: schedules

`writ_set_schedule` on a saved workflow:

- An interval: `every_minutes`.
- Daily or weekly: `kind: "daily"` or `"weekly"`, `time` (`"08:00"`), `days` (1 = Monday to 7 = Sunday) and `tz` (an IANA zone such as `"America/Toronto"`).
- A multi-function workflow: `functions` picks which functions run, with saved `inputs`. The answer lists `also_runs` (what they depend on) and any `missing_inputs` to fill.

A saved crawl (see `writ-read-and-crawl`) has no schedule of its own, and `writ_set_schedule` turns crawl ids and names away. To crawl on a clock, call `writ_create_automation` with `when: "scheduled"` and a raw `blocks` tree: a `scheduled` root and one `crawl` action block whose `config` carries the crawl settings (`seed_url`, `extract_mode`, `extract_schema`, `intent`, `include_paths`, `page_budget`, `persona_id`):

```json
{"name": "Docs every morning", "when": "scheduled", "blocks": [
  {"id": "clock", "type": "event", "blockType": "scheduled", "config": {"mode": "daily", "time": "08:00", "tz": "America/Toronto"}},
  {"id": "crawl", "type": "action", "blockType": "crawl", "parentId": "clock", "config": {"seed_url": "https://example.com/docs", "intent": "the API reference pages", "page_budget": 200}}
]}
```

Each fire starts a fresh crawl with those settings; it does not refresh the saved crawl, so read its rows from the `crawl_completed` event or the crawl's own data, not from `writ_saved_crawl_data`.

## Deliver the result

A schedule only runs the job. To send its data somewhere (an email digest, a Slack message), chain another workflow or wake an agent after it, add an automation with `when: "workflow_completed"` or `"crawl_completed"`: see the `writ-automations` skill.

## Before you create one

Monitors, schedules and automations keep running and use the user's plan after the chat ends. State what will run, how often and who gets notified, and confirm with the user before creating it.
