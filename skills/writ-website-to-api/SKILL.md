---
name: writ-website-to-api
description: Turn a website into a callable API with Writ, then run, prove, publish and repair it. Use when a site has no official or practical API but the user wants its data or actions programmatically ("turn this site into an API", "an endpoint for", "map the API of"), wants something their code, spreadsheet or another agent can call later, or wants to run a saved Writ workflow and get its data.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Website to API

## 0. Already built?

`writ_list_workflows` lists the account's saved workflows, each with its functions and inputs. When one covers the ask, run it (step 3) and stop.

## 1. Start the build: `writ_website_to_api`

Three things decide the result:

- **`url`: the page that already shows the rows**, such as the search-results, category or listing URL, not the app's home page. A build seeded at an empty shell produces nothing.
- **`goal`: the inputs and fields in plain language**, for example "page number in; quotes with text, author and tags, and has_next out".
- **`save_as`**: a name for the saved workflow.

Add `persona_id` when the site needs the user's account (see `writ-signed-in-sites`). It takes a cloud persona's number only, never a `device:` persona; to build on a linked desktop, pass its agent id as `execution_target`.

The default mode (`intelligent`) runs the whole ladder itself: the user's own matching workflows and ready-made marketplace APIs first, then the AI fast path (one real browser, a few AI steps, every function live-tested and reads proven on a second input), then Writ's own AI browser rung only when the fast path cannot prove the functions. Building never sends a write (create, update, delete) to the site.

The crawl rungs run only when you ask for them: `mode: "crawl"` (static) or `mode: "browser"` (rendered) maps a whole server-rendered site instead. Those maps stay unverified (`verified: false`) until a run proves them.

The first answer may be a **proposal** instead of a build:

- `existing_workflows`: the user already has a match. Run it, or call again with `skip_existing: true` if they declined.
- `marketplace_candidates`: a ready-made API. Offer it, or call again with `skip_marketplace: true`.

## 2. Wait for it once: `writ_discovery_status`

Call it with the `build_id` and `wait: true`: one held call (up to 75 s) that follows every rung. If it is still running at the ceiling, call it once more the same way. Never poll in a loop, and do not open a browser or start a second build for the same site meanwhile.

- `succeeded`: it returns the `workflow_id` and a `run_example`. `verified: false` means the functions are candidates until a real run proves them; say so.
- `needs_guidance`: the mechanical rungs are done and the build is yours to finish. Call `writ_website_to_api` with `mode: "guided"` and the `build_id`; it opens a browser bound to the build's map (`writ_browser_context` with `section: "map"`). Drive it with `writ_browser_act`, define and test functions with `writ_browser_compose` (the `writ-http-functions` skill), then `writ_browser_save`.
- `escalations` says why each earlier rung handed over; quote it when you explain why a browser was needed. When a crawl rung reports that `robots.txt` refused it and the user vouches for the site, start again with `respect_robots: false`. It only affects the crawl rungs (`mode: "crawl"` or `"browser"`), never the AI or guided browser rungs.

## 3. Run it and prove it

Run exactly as `run_example` shows:

```json
{"workflow_id": 123, "function_name": "quotes.list", "inputs": {"page": 2}}
```

- `function_name` runs one function and only the sign-in it depends on; `function_names` runs several in one run.
- Run it with **two different inputs** and check the answers differ before reporting success.
- `max_age` (seconds) reuses a recent result instead of running again.
- `output` shapes the answer for a program: `{"shape": "records"}`, `{"shape": "record"}`, `fields` to pick and rename, `exclude`.

### Functions that write

A function that creates, posts, sends or deletes is sent on a normal run (`mutation_mode` defaults to `live`). Confirm with the user before calling one. `mutation_mode: "dry_run"` previews the request without sending it; `"private_test"` sends it with the function's safe overrides.

## 4. Hand it over

- **REST endpoint:** `writ_expose_workflow_api` returns a POST URL that runs the workflow and returns its JSON. Repeat calls reuse the endpoint.
- **API docs:** OpenAPI 3, Markdown or Postman at `GET /api/v1/workflows/{workflow_id}/api-docs`.
- **Its own MCP tool:** `writ_pin_workflow_tool` adds a `run_<name>` tool.
- **On a clock:** `writ_set_schedule` (see `writ-watch-and-schedule`).

## 5. When a run returns nothing

The full repair loop is the `writ-fix-workflows` skill. In short:

An empty answer is not proof that nothing matched.

1. `writ_workflow_runs` shows the run's status and error.
2. `writ_diagnose_http_workflow` with the `task_id` shows what every request was answered with. A 200 with an empty feed usually means the session was signed out or blocked: refresh the persona (`writ_personas` `sign_in`).
3. Fix the workflow in place with `writ_update_workflow` (`patch`, `step_updates`), or `writ_create_http_extraction` for multi-request extractions, then run it again on two inputs.

Answer questions about data already collected from `writ_search_data` or `writ_workflow_data` before running anything (the `writ-collected-data` skill).
