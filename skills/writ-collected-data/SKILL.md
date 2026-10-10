---
name: writ-collected-data
description: Answer questions from data Writ has already collected, without running anything. Search across every workflow's and crawl's results, read a dataset as a table, fetch full records, shape rows for a program, and export CSV or JSON. Use when the user asks about data a Writ workflow, crawl or monitor already gathered ("what did the scraper find", "the listings from this morning", "export that to CSV"), or before re-running a job whose results may already be there.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Data Writ already collected

Reading stored data costs no run, no browser and no AI. Check it before running a workflow or crawl again.

## Find it

- **Don't know where it is:** `writ_search_data` with `q` searches everything the account's workflows collected, grouped by workflow. Add `workflow` or `workflow_id` to scope it. Matches are preview-sized.
- **Know the workflow:** `writ_workflow_data` with `workflow_id` (or `workflow` by name) returns a table (columns and rows).
  - `q` filters rows by a substring across fields.
  - `view`: `latest` (the last run), `all` (every run), or `run` with `run_id` (one run).
  - `limit` caps the rows.
- **Which workflows exist:** `writ_list_workflows` (with `search`).
- **Recent runs, their status and errors:** `writ_workflow_runs` (`status`: `success`, `failed`, `running` and so on).

## Workflows and crawls are different ids

A crawl id is not a workflow id.

- A crawl's rows: `writ_crawl_status` with the `crawl_id` (and `wait: true` if it is still running). Its answer carries `data_workflow_id`, the dataset holding its rows, which `writ_workflow_data` then reads.
- A saved crawl's latest rows: `writ_saved_crawl_data` with the crawl's name. It never starts a crawl.
- Passing a crawl id to a workflow tool gets a redirect to the right tool, not data.

## Full records

Long text cells arrive cut (`_truncated` lists the cut fields). Every row carries its `run_id` and `record_index`. Fetch the full records with `writ_workflow_data` and `refs: ["<run_id>:<record_index>", ...]` (up to 100), or set `preview_chars: 0` for whole cells.

## Shape it for a program

When the result goes to code, a spreadsheet or another agent, pass `output` instead of reshaping it yourself:

- `{"shape": "records"}`: a bare list of records.
- `{"shape": "record"}`: the newest record alone (one entity, a usage meter).
- `{"shape": "table"}`: `{columns, rows, total}`.
- `fields`: pick, order and rename, with dotted paths (`["title", "price as amount", "seller.name"]`). A missing field comes back `null`, so keys stay stable.
- `exclude`: drop fields. `key`: wrap the answer under a name. Page metadata is stripped unless `include_meta: true`.

## Export

`writ_export_data` returns a workflow's whole table, un-paginated, as `format: "csv"` (default) or `"json"`, with `q` and `view` applied.

## The user's own desktop

Data collected on a linked Writ desktop stays on it. Pass `device` (an agent id from `writ_devices`) to read it, or scope the connection with `writ_devices` `action: "use"` (see `writ-desktop-devices`).

## Fresh or stored?

When the user asks for "the latest" and the stored run is recent enough, answer from it and say when it was collected. When they need it fresh, run the workflow; `max_age` (seconds) on `writ_run_workflow` or `writ_run_saved_crawl` reuses a result younger than that instead of running again.
