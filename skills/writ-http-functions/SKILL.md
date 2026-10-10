---
name: writ-http-functions
description: Build browserless API functions in Writ from a site's real network requests or its server-rendered HTML, so a saved workflow runs over plain HTTP with no browser and no AI. Covers capturing requests, defining and live-testing named functions, sign-in and token chains, row extraction from HTML or JSON, cursor pagination and multi-request flows. Use when turning a site into a fast, reliable API by hand, when a website-to-API build parks as needs_guidance, or when a recorded workflow should stop depending on a browser.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Browserless functions

A **function** is a named, callable piece of a saved workflow (`orders.list`, `quotes.search`), run with `writ_run_workflow` and `function_name`. An `api` function replays one HTTP request and extracts fields from the response, with no browser at run time.

Try `writ_website_to_api` first (the `writ-website-to-api` skill). Build functions by hand when it parks as `needs_guidance`, or when you are improving a recording.

## 1. Find the real request

In a browser session (`writ_browser_use`, `writ_record_website`, or `writ_website_to_api` with `mode: "guided"`):

1. Reach the state that shows the data (a search, a filter, a page 2).
2. `writ_browser_act` with `{"action": "capture_network"}`. To catch a POST (login, search, submit), perform the action that triggers it, then capture.
3. `writ_browser_network` with `operation: "search"` (filter by `query` or `method`) lists the calls; `operation: "detail"` with `index` returns one in full: method, URL, headers, request body, response body.

The server-rendered shortcut: when the rows are already in the page's HTML, `writ_scrape` with `format: "html"` shows them. No capture is needed.

## 2. Define it: `writ_browser_compose` `define_function`

```json
{"session_id": "<id>", "operation": "define_function", "payload": {
  "name": "quotes.list", "fn_type": "api", "from_index": 3,
  "request": {"url": "https://site.example/api/quotes?page={{page}}"},
  "input_variables": [{"name": "page", "example": "1"}],
  "response_extractions": {"quotes": {"from": "json", "path": "quotes"},
                           "has_next": {"from": "json", "path": "has_next"}},
  "then_save": {"name": "Quotes API"}}}
```

- `from_index` seeds method, URL, headers and body from the captured call; override `request` fields to put `{{inputs}}` in them.
- Every function is **live-tested** as you define it (`sample_inputs`, or each input's `example`). A failed test keeps nothing: fix it and define it again with the same name.
- `then_save` defines, tests and saves in one call.
- Name functions `<surface>.<verb>` so they group: `orders.list`, `orders.get`, `orders.create`.
- Deterministic alternative for an authenticated GraphQL/RPC call or a form submit: `compile_function` with `from_index`. It traces session tokens to a sign-in function and generates per-call ids (`{{uuid()}}`, `{{timestamp_ms()}}`).

### Extraction sources (`response_extractions`)

- JSON: `{"from": "json", "path": "data.items"}`
- JSON embedded in HTML: `{"from": "embedded_json", "kind": "array", "has": ["id"]}`
- Server HTML rows: `{"from": "html_css", "selector": "tr.athing", "all": true, "base_url": "<page>", "fields": {"title": {"selector": ".titleline > a"}, "url": {"selector": ".titleline > a", "attribute": "href"}}}`. With `fields` it returns row objects, which is how a server-rendered list becomes a browserless function. Prefer it over a `list` or `script` function whenever the rows are in the served HTML.
- Also: `regex` (`pattern`, `group`), `header` (`name`), `body`.

Each extraction is returned as an output field and published to later steps as `{{extracted:<name>}}`.

### Other function types

`list` (a row selector plus `fields` on the live page; `writ_browser_context` `section: "lists"` hands you a ready-made payload), `script` (a read-only JS function returning data), `extraction` (one selector's text). These read the page, so they need a browser at run time.

## 3. Sign-in and tokens

Use an ordered, named graph of functions:

- The sign-in or bootstrap function has `is_auth: true`. It runs first on every replay, and its `response_extractions` publish tokens, ids or hosts.
- Data functions consume them as `{{extracted:<name>}}` and stay independently callable.
- Credentials come from the persona as `{{secret:<name>}}`, never as inputs. Anti-CSRF cookie echoes are `{{cookie:<NAME>}}`.

## 4. Inputs and pagination

- `set_inputs` declares each caller parameter (`default`, `description`, `required`, `example`). Every `{{name}}` must be a declared input, a credential, a `{{cookie:}}` or `{{extracted:}}` reference, or produced by an earlier step, or the save is refused.
- Declare search, filter, limit and page, offset or cursor inputs, and return `next_cursor`, `next_offset` or `has_more`.
- Never append Writ-only controls to the site's URL.

## 5. When one request is not enough

Request loops, GraphQL descriptor discovery, recursive mapping, cross-page dedupe, sorting or cursor walks need a flow (`api_call.config.flow`). On a saved workflow, `writ_create_http_extraction` generates one from a plain-language `goal`, `request_samples` (calls from `writ_browser_network`) and `desired_inputs`:

1. Call it with `apply: false` and review the draft and its validation.
2. Call it again with `apply: true`.

Never hide `fetch` inside `evaluate_js`.

## 6. Writes

A request that creates, posts, sends or deletes is marked as a write. Building and testing never send it; a normal run does (`mutation_mode: "live"`). `writ_run_workflow` with `mutation_mode: "dry_run"` previews it. Confirm with the user before a run that writes.

## 7. Prove it over HTTP

After saving, run the workflow with its intended persona and egress on two different inputs, then `writ_diagnose_http_workflow` with the `task_id`. Ship it only when it ran as `engine=http`, returned non-empty records, and pagination matches what the browser showed. Otherwise name the measured browser-only blocker.
