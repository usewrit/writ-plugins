---
name: writ-browser-tasks
description: Do a task on a website in a real Writ cloud browser (click, fill a form, submit, search inside an app, change a setting, buy, book or post) and record it as a reusable workflow when it will come back. Use when the user asks for an action on a site rather than just its content, says "every time", "again", "weekly" or "record this", or wants a task on a site they are signed in to done for them.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Tasks in a Writ cloud browser

Writ opens a real browser in the cloud; you drive it. No Writ model acts in the session: the `goal` you pass is a label for the run log, and every step is yours.

## First, is it already saved?

Call `writ_list_workflows` (with `search`) before opening any browser. A saved workflow replays with `writ_run_workflow` at a fraction of the cost and without a browser. Only open one when nothing saved fits.

Reading a page is not a browser task: use `writ_scrape` (the `writ-read-and-crawl` skill).

## One-off task or reusable one?

| The ask | Tool | Afterwards |
| --- | --- | --- |
| Once, now | `writ_browser_use` (url, optional `persona_id`) | `writ_browser_cancel` when done |
| Will come again ("every", "each time", "weekly", "record it"), or the same steps differ only by a value | `writ_record_website` (url, goal) | `writ_browser_save`, then prove it |

Every browser turn returns the live page (DOM, fields, buttons), roughly 20k tokens. A saved workflow replays for about 1k tokens with no AI. Recording pays off the second time the task comes back, but do not record a genuine one-off: a library of one-shot workflows is noise for every future session.

Before either, if the site needs the user's account, get a `persona_id` first (the `writ-signed-in-sites` skill).

Both tools also take:

- `auth_mode: "fresh_login"` to record a new login: the browser starts without the persona's old cookies and storage but keeps its device and network identity. The default, `reuse`, checks a saved session before adopting it.
- `human_layer`: real keyboard input, mouse paths and a pause before each click. A fresh login turns it on; `false` turns it off.
- `use_residential: true` (with an optional `residential_country`) for a site that blocks datacenter IPs. A browser's exit IP is fixed once it opens, so on a bot wall or a CAPTCHA, `writ_browser_cancel` it and open a new one with `use_residential: true`. `fresh_exit: true` draws a new residential address after the site refused the usual one (an IP rate limit, a correct sign-in rejected).

## Drive it

Call `writ_browser_use` or `writ_record_website` once per task, then `writ_browser_act` with the `session_id` and a batch of `actions`:

```json
{"session_id": "<id>", "actions": [
  {"action": "fill", "selector": "input[name=q]", "value": "{{city}}"},
  {"action": "press_key", "key": "Enter"}
], "inputs": {"city": "Montreal"}}
```

- After a navigate, a click or a select that changes the page, end the batch and look at the new page before acting on it.
- Look before you click: `query_dom`, `find_text`, `read_text` and `list_candidates` (repeating rows) are cheaper than `get_dom`. `writ_browser_context` re-reads the live page; `section: "lists"` returns its lists.
- A site's real API: `capture_network`, then search it with `writ_browser_network`.
- Signing in with a persona: the session is already signed in, or `twofa` mints the code server-side. Read the challenge first and pass the method the page uses as `challenge_method` (`sms`, `email`, `authenticator` or `other`). A CAPTCHA or a check only the user can pass: `writ_browser_ask_user` (see `writ-signed-in-sites`).

## Make the recording reusable

- **Inputs:** write a value the user will change as `{{name}}` in the action and pass the real value in `inputs`. The page gets the real value, the saved step keeps `{{name}}`, and `name` becomes a workflow input.
- **Data:** record an `extract` action (`variable` plus a read-only script) at the point the data shows. Its result comes back in the same answer; check it before saving.
- **Secrets:** fill a password with `data_key`, never a literal value.
- **Named functions, input descriptions, explicit steps:** `writ_browser_compose` (`define_function`, `set_inputs`, `add_steps`, `test_function`).

## Save, then prove it

1. Only once the task worked on the live page: `writ_browser_save` with the `session_id` and a short `name`. It closes the browser and the workflow is active immediately.
2. Run it with `writ_run_workflow` on **two different input values** and check the answers differ. Only then report it done.
3. A run that returns nothing: `writ_diagnose_http_workflow` with the run's `task_id`, then fix it with `writ_update_workflow` (the `writ-fix-workflows` skill).

Optional next steps: `writ_pin_workflow_tool` gives it its own `run_<name>` tool, `writ_set_schedule` runs it on a clock (`writ-watch-and-schedule`), `writ_expose_workflow_api` gives it a REST endpoint. To make it run without a browser, see `writ-http-functions`.

## Close what you open

An open cloud browser keeps billing. When the task is done and nothing is to be saved, call `writ_browser_cancel` (unsaved work is auto-saved unless `discard: true`). Before opening a new browser, `writ_browser_sessions` lists sessions you can resume instead.

## Actions with consequences

Clicking "send", "post", "delete", "submit" or changing a setting acts on the user's real account. Do it only when the user asked for that action, and confirm first when the ask is ambiguous. A saved workflow with such a step sends it again on every run.

Buying is different: you never press the order or pay button yourself (it is refused with `use_writ_payment_checkout`). Browse to the final order page, then call `writ_payment` `action: "checkout"`; the user confirms and Writ places the order. See the `writ-payments` skill.
