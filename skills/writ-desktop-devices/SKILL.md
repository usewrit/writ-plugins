---
name: writ-desktop-devices
description: Run Writ work on the user's own computer through a linked Writ desktop app instead of the cloud, with their own browser, logins, local vault and IP address. Use when the user says "on my laptop", "on my work computer", "from my own IP", "use my own browser", wants credentials to never leave their machine, or asks about workflows, personas, data or monitors that live on their desktop.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client and a Writ desktop app linked to the same account; tools are named writ_*.
---

# The user's own desktop

A **linked Writ desktop** is the Writ desktop app, signed in to the same account. Work sent to it runs on that machine, from its IP address, and signs in from its own vault: the credentials never leave the computer.

## See what is linked

`writ_devices` (`action: "list"`, the default) shows each linked desktop, whether it is connected right now, and how many local workflows and personas it offers. A desktop with workflows that reach its local network carries a `local_network` line saying how many run there, and the answer adds a `local_network_note`.

## Scope the connection or target one call

- **Everything, for this connection:** `writ_devices` `action: "use"` with `device` (the agent id or exact name). Runs, browser sessions, desktop workflows and personas, run history, data and monitors then target that desktop. `action: "clear"` removes the scope.
- **One call:** pass `device` (an agent id from the list) on the tools that take it: `writ_run_workflow`, `writ_scrape`, `writ_crawl_site`, `writ_workflow_data`, `writ_search_data`, `writ_workflow_runs`, `writ_create_monitor`, `writ_wire_monitor`, `writ_website_to_api` (`execution_target`), `writ_record_website`.
- **Browser sessions:** `writ_browser_use` with `execution_target`: `"local"` requires the desktop, `"auto"` prefers it when online and falls back to the cloud, `"cloud"` is the fleet.

## Desktop workflows and personas

- `writ_list_workflows` includes the desktop's own workflows (`runs_on: "desktop"`, ids like `local:<id>`). Run one with `writ_run_workflow` and that id; it runs on the desktop.
- A workflow that reaches a desktop's local network (for example an intranet page or a shop computer) runs only on that desktop. Its `writ_list_workflows` row says so: `runs_on: "desktop"`, `agent_id`, `reaches_local_network: true` and a `venue` line naming the machine.
- `writ_personas` lists the desktop's personas with `source: "device"` and ids like `device:<agent>:<id>`, name and site only. Passing one as `persona_id` sends the work to that desktop, which signs in only on the persona's own site.
- `writ_devices` `action: "monitors"` lists the desktop's monitors; `action: "datasets"` lists what its exposed workflows collected.

## When the desktop is offline

A run with `execution_target: "local"` fails when the app is not running or not connected. Tell the user; fall back to the cloud only with their agreement, because the cloud runs from Writ's IPs and needs a cloud persona for any login.

A workflow that reaches the desktop's local network has no cloud fallback: it runs on that desktop even with `execution_target: "cloud"`, and fails with `device_offline` while the desktop is offline. Ask the user to open the app, then run it again.

## Which Writ server is this?

The user may also have the desktop app's own local MCP server connected (it reports itself as `writ-local` and is usually configured under the key `writ`) or a self-hosted coordinator. This skill is about the cloud connector ("Writ Cloud") reaching a linked desktop. On any surface, prefer an existing saved workflow over building a new one.
