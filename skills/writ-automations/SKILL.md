---
name: writ-automations
description: Build Writ automations that react to events: when a workflow or crawl finishes or fails, when a watched page changes, when another system calls a signed webhook URL, or on a clock, then run a workflow (or chosen functions), send a notification with the collected data, wake an AI agent, or run a multi-step flow with scrape, extract and condition blocks. Use when the user wants "when X happens, do Y", a daily digest emailed, workflows chained together, a URL another app can call to start a workflow, an alert only when a value crosses a threshold, or an agent that acts on a change.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Automations

An automation is an **event** plus one or more **actions**. `writ_create_automation` builds it. It keeps running after the chat ends and uses the user's plan, so before creating one, state what will run, when, and who gets notified, and confirm with the user.

## Pick the shape

| The user wants | Call |
| --- | --- |
| A notification, a workflow or an agent when a watched page changes | `writ_wire_monitor` on the monitor (see `writ-watch-and-schedule`) |
| Something when a workflow or crawl finishes, starts or fails | `writ_create_automation` with `when` and the action arguments |
| Actions at a time of day or on an interval | `writ_create_automation` with `when: "scheduled"` and `schedule` |
| Something when another system calls a URL (a form, a CRM, a script) | `writ_create_automation` with `when: "webhook_received"`, which mints the URL |
| Conditions, a scrape, an extraction or branches in between | `writ_create_automation` with a raw `blocks` tree |

Every automation needs a `name` and at least one of `run_workflow`, `notify` or `ai_prompt` (or action blocks).

## Events (`when`)

- `workflow_completed`, `workflow_started`: name the source with `on_workflow` (or `on_workflow_id`).
- `crawl_completed`, `crawl_failed`: after any crawl on the account (saved, scheduled or one-off). Narrow it to one site with the root event block's `seed_host` in a `blocks` tree, or a condition on `{{seed_host}}`.
- `ai_session_completed`, `ai_session_started`.
- `scheduled`: a clock. `schedule` is `{"kind": "interval", "interval_minutes": 60}`, `{"kind": "daily", "time": "08:00", "tz": "America/Toronto"}` or `{"kind": "weekly", "time": "08:00", "days": [1, 3, 5], "tz": "Europe/Paris"}` (1 = Monday).
- `change_detected`: name the monitor in `target_id` (the `monitor_id` from `writ_create_monitor`), and `target_selector_id` for one of its selectors. `writ_wire_monitor` is the simpler path.
- `webhook_received`: another system POSTs to a signed URL that Writ mints. See [On a webhook](#on-a-webhook).

## Actions

### Run a workflow

`run_workflow` (name) or `run_workflow_id`. On a multi-function workflow, `run_functions` picks the functions; whatever they depend on (sign-in, a token) runs too. `inputs` fills its inputs, each a literal or a `{{template}}` over the event, for example `{"max_price": "{{extracted.price}}"}`. Saved values fill the rest. Secrets never go in `inputs`. The tool refuses an automation whose functions would miss a required input.

### Notify with the data, not just that it ran

`notify` is a message template over the event, sent on `channels`:

- after a workflow: `{{result.extracted_data.0.title}}`, `{{result.extracted_data.<variable>.0.url}}`
- after a crawl: `{{rows.0.<field>}}`, `{{row_count}}`, `{{seed_host}}`, `{{pages_done}}`
- after a monitor change: `{{extracted.price}}`, `{{event.url}}`

A missing path renders empty, so a digest of N rows can be N numbered lines. Add `title` for the subject.

`channels`: `email`, `pushover`, `twilio` (SMS), `whatsapp`, `signal`, `webhook`, and the connected `slack`, `discord`, `telegram`. Omitting `recipients` reaches every enabled recipient on the channel. The answer names who the alert actually reaches and warns when nobody is configured. Email and phone recipients only receive once they have confirmed their code. The user adds recipients in the Writ app, under Integrations, Recipients.

### Wake an AI agent

`ai_prompt` is the task, for example "Compare the new listings with yesterday's and message me the three best". The agent gets the event context and works in a cloud browser. `ai_entry_url` is the page it starts on (needed for workflow and webhook events). `cooldown_minutes` (default 10) limits how often it wakes.

## On a webhook

`when: "webhook_received"` mints a new webhook URL with its own signing secret. The answer's `webhook` holds what the user needs:

- `url`: where the other system POSTs JSON.
- `signing_secret`: shown only in this answer, because Writ stores it encrypted. Give it to the user right away so they keep it with the sender. A lost secret is replaced in the Writ app, on the automation's webhook block.
- `sign_each_call`: every call sends `X-Writ-Timestamp` (Unix seconds, within 5 minutes) and `X-Writ-Signature`, which is `sha256=` plus the hex HMAC-SHA256 of `<timestamp>.<raw body>` keyed with the secret. Unsigned, stale and replayed calls are refused: each call needs a fresh timestamp and signature.
- `example_curl` and `example_python`: calls that sign correctly, reading the secret from `WRIT_WEBHOOK_SECRET`.
- `example_body`: the fields this automation uses.

Where the body goes:

- Each top-level JSON field (and URL query parameter) fills the workflow input of the same name, over its saved value. The automation's own `inputs` win over the body.
- `inputs`, `notify` and `ai_prompt` read any field, nested ones too, as `{{payload.<path>}}`, for example `{{payload.order.id}}`.

A call is answered at once with `data.trigger_rules_fired`, the number of automations it started; they run in the background. To get the result back in the same request, add `?wait=true` to the URL (and `&timeout=` 10 to 300 seconds, default 120): the call is held until the workflow runs the automation starts have finished, and the answer adds `status` (`success`, `failed` or `timeout`), `extracted_data` (a Return block's data, else the last run's rows) and `runs`. On a timeout the automation keeps running; read it later with `writ_workflow_runs`.

The sender signs each request itself: a script, a server, or a code step in n8n or Zapier. A service that signs its own way (Stripe, GitHub) cannot call the URL directly.

`writ_list_webhooks` lists the account's webhook URLs, the automations each one fires, and whether each is signed. To hang a new automation on a URL a sender already uses, pass its `webhook_trigger_id`: the sender keeps its secret. A webhook listed with `signed: false` refuses every call until it has a secret; reusing it mints one.

## Recipes

**Daily digest by email:**

1. `writ_set_schedule` on the workflow, for example daily at 08:00.
2. `writ_create_automation`:

```json
{"name": "Morning listings digest", "when": "workflow_completed", "on_workflow": "Kijiji apartments",
 "notify": "1. {{result.extracted_data.0.title}} {{result.extracted_data.0.price}}\n2. {{result.extracted_data.1.title}} {{result.extracted_data.1.price}}\n3. {{result.extracted_data.2.title}} {{result.extracted_data.2.price}}",
 "title": "New apartments", "channels": ["email"]}
```

**Start a workflow from another system:**

```json
{"name": "Lead from the website form", "when": "webhook_received", "run_workflow": "Create CRM lead",
 "inputs": {"company": "{{payload.org.name}}"},
 "notify": "New lead: {{payload.email}}", "title": "New lead", "channels": ["email"]}
```

The form's backend then POSTs `{"email": "...", "org": {"name": "..."}}` to `webhook.url`, signed as in `webhook.example_curl`. `email` fills the workflow's `email` input by name.

**Chain two workflows:** `when: "workflow_completed"`, `on_workflow: "A"`, `run_workflow: "B"`, with `inputs` templated from A's result.

**Run functions on a clock, then send what they returned:** `when: "scheduled"` with `schedule`, `run_workflow` plus `run_functions`, and `notify`. The notification waits for that run, so `{{result.extracted_data...}}` is filled.

## Multi-step flows: `blocks`

Use `blocks` for anything the arguments above cannot express. It is a list of at most 50 blocks, each `{id, type, blockType, config, parentId}`:

- The **first** block is the root event: `type: "event"`, `blockType` is one of the events above. A scheduled root takes `{"mode": "daily", "time": "08:00", "tz": "..."}` or `{"mode": "interval", "interval_minutes": 30}`.
- Every other block names an **earlier** block as its `parentId`.
- Action blocks (`type: "action"`):
  - `workflow`: `{workflow_id, function_name | function_names, input_mapping}`
  - `notification`: `{template, title, channels, recipients}`
  - `ai_session`: `{goal, entry_url}`
  - `scrape`: `{urls (up to 5, templates allowed), format: markdown | html | both, on_error}`, which publishes `{{scraped.content}}` and `{{scraped.pages}}`
  - `extract`: `{source: "{{scraped.content}}", fields: [{key, from: html_css | json | regex | embedded_json | body, selector, attribute, all, path, pattern, group, number, required}]}`, which publishes `{{extracted.<key>}}`
- Condition blocks (`type: "condition"`): `{field, operator, value}`. `field` is a dotted path such as `extracted.price` (no braces). Operators: `equals`, `not_equals`, `contains`, `not_contains`, `matches` (regex), `exists`, `gt`, `gte`, `lt`, `lte`, `changed` (differs from the previous run). A condition that fails stops its branch.
- To wait for a run started by a workflow block: an event block `workflow_completed` with `{"linked_to_block": "<workflow block id>"}`.
- Every string setting takes `{{placeholders}}`.

Example: check a price every morning, notify only when it drops below 500:

```json
[
  {"id": "clock", "type": "event", "blockType": "scheduled", "config": {"mode": "daily", "time": "08:00", "tz": "America/Toronto"}},
  {"id": "page", "type": "action", "blockType": "scrape", "parentId": "clock", "config": {"urls": ["https://shop.example/item/42"], "format": "html"}},
  {"id": "read", "type": "action", "blockType": "extract", "parentId": "page", "config": {"source": "{{scraped.content}}", "fields": [{"key": "price", "from": "html_css", "selector": ".price", "number": true, "required": true}]}},
  {"id": "cheap", "type": "condition", "blockType": "condition", "parentId": "read", "config": {"field": "extracted.price", "operator": "lt", "value": 500}},
  {"id": "tell", "type": "action", "blockType": "notification", "parentId": "cheap", "config": {"template": "Price is now {{extracted.price}}", "title": "Price drop", "channels": ["email"]}}
]
```

`blocks` cannot be mixed with `run_workflow`, `run_functions`, `inputs`, `notify`, `ai_prompt` or `schedule` in the same call: with blocks, the schedule lives on the root block.
