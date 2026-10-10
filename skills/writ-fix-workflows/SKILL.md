---
name: writ-fix-workflows
description: Diagnose and repair a saved Writ workflow that returns nothing, fails or has drifted, and edit a workflow in place (settings, persona, steps, functions, schedule and its agent skill). Use when a Writ run comes back empty or errors, a site changed, a workflow was signed out, or the user wants to change how a saved workflow behaves.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Fix and edit saved workflows

## An empty answer is not "no results"

Before telling the user nothing matched, find out what the site actually answered.

1. `writ_workflow_runs` with the `workflow_id` (`status: "failed"` to narrow it): the run's status, timing and error.
2. `writ_diagnose_http_workflow` with the `workflow_id` and the run's `task_id`: whether it really ran over HTTP, whether it fell back to a browser, and what **every request** was answered with (status, size, a response sample). Read it first when a run returned nothing.
   - A 200 with an empty feed means the site stopped serving this session: signed out, blocked, or a request variable overwritten with null. It does not mean "no matches".
   - Without `task_id` it reports readiness: browser-only dependencies, invalid flow expressions, unsafe query parameters, and the next repair action.

## Common causes and their fix

| Symptom | Fix |
| --- | --- |
| Signed out, session error, login page in the response | `writ_personas` `action: "sign_in"` with the persona (`force: true` if the session looks fine but is not) |
| No saved login for the site | the persona flow in `writ-signed-in-sites` |
| Bot wall, 403, empty page from a datacenter IP | `writ_update_workflow` `patch` turning on residential egress (or pass `use_residential: true` on the run) |
| Wrong country's results | `residential_country` on the run, or in the workflow's `patch` |
| A selector no longer matches | `function_updates` on the function's step, or `step_updates` on a step that belongs to no function (read the outline first) |
| A request the site changed | redefine the function (`writ-http-functions`) or `writ_create_http_extraction` |
| A missing input | `writ_list_workflows` shows the declared inputs; pass it in `inputs` |

## Edit a workflow: `writ_update_workflow`

- **Read it:** call it with only `workflow_id` (or `workflow`). It returns a compact outline: each step's zero-based index, stable step id, type, function and source hash, plus `updated_at`. `verbose: true` returns the full definition, with its agent skill (`skill_md`).
- **Read one function:** `function_name` (plus `step_id` when the function has several steps, and a JSON Pointer `path` such as `/config/flow`) returns that source only, windowed by `offset` and `max_chars`. `section: "flow"` with `function_name` lists the nested flow nodes and their paths.
- **Operators:** `section: "contract"` lists every HTTP flow operator with its operands and what it does; `operator` picks one. It needs no workflow.
- **Repair a function:** prefer `function_updates`, a list of `{function_name, step_id?, expected_hash?, patch?, json_edits?, script_edits?}`:
  - `patch` merges the step's type, config and enabled flag (objects merge, arrays replace).
  - `json_edits` change one nested value with `test`, `add`, `replace` or `remove` and a `path`, without rewriting the program.
  - `script_edits` replace `old` with `new` at a `path`; `old` must match exactly once.
- **Check first:** `validate_only: true` previews the edits and validates the HTTP grammar without saving, running JavaScript or sending requests. Pass the `expected_updated_at` you read: a 409 means someone else changed it, so read it again.
- **Settings:** `patch` is a sparse settings patch: default persona, headless or fast mode, device routing, timeouts, retries, schedule, functions, sessions. Credentials are never set here; they belong to personas.
- **Network settings:** residential egress and its country, and human behavior, are patched the same way.
- **Steps by index:** `step_updates` merges into indexed steps (selectors, scripts, URLs, waits, extraction config). `replace_steps` replaces the whole list; use it only on purpose.
- An `api_call` step's `config.function_name` binds it to the function that `writ_run_workflow` `function_name` selects.
- A source edit (`function_updates`, `json_edits`, `script_edits`) answers with a receipt: `source_changes` and the new `updated_at`. Any other edit answers with the fields it changed; `verbose: true` returns the full definition.

## Keep its agent skill current

Each workflow carries a SKILL.md that teaches an agent when and how to call it.

- `regenerate_skill: true` rebuilds it from the workflow's current functions, inputs and sign-in (after any `patch` in the same call).
- `patch.skill_md` writes an edited version (YAML frontmatter `name` and `description`, then Markdown); `patch.skill_md: null` removes it.

## Prove the fix

Run it again with `writ_run_workflow` on **two different inputs** and check the answers differ and are not empty. For an HTTP workflow, confirm with `writ_diagnose_http_workflow` and the new `task_id` that it ran as `engine=http` and returned records. Only then tell the user it is fixed.

## Before editing

`writ_update_workflow` overwrites the saved definition, and scheduled runs pick up the change. Say what you are about to change. A workflow installed from the Writ marketplace cannot be edited.
