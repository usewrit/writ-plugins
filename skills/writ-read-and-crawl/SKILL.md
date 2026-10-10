---
name: writ-read-and-crawl
description: Read web pages and collect whole sites or sections into datasets with Writ. Use when the user wants what a page says, a summary of a URL, the top N items of a listing and what each one says, several links fetched together, a page that a plain fetch could not open (403, bot wall, JavaScript-rendered, behind their sign-in), or every page of a site or section collected as rows to query, export or re-run later.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Read pages and collect sites with Writ

Two tools, two jobs:

| The user wants | Call | What comes back |
| --- | --- | --- |
| What a page (or a few pages) says, right now | `writ_scrape` | Clean markdown in this call (2-10 s), nothing stored |
| A site or section as a dataset | `writ_crawl_site` | Rows stored as a queryable, change-tracked dataset |

Neither opens a browser session for you to drive. Clicking, typing and submitting are the `writ-browser-tasks` skill.

## Read pages: `writ_scrape`

One call answers each of these shapes:

- One page: `{"url": "https://example.com/pricing"}`
- Several known pages, fetched in parallel (up to 20): `{"urls": ["https://a.test/1", "https://a.test/2"]}`
- The top N of a listing and each item page (N up to 20): `{"url": "https://news.ycombinator.com/", "top_n": 3}`. The listing's link order is the ranking. Add `include_paths` (for example `["item\\?id="]`) when you know the item-link shape.

Then read the markdown and answer. Do not split a top-N ask into one call per page, and do not crawl it.

When the page does not come back usable:

- Blocked or a bot wall: `"use_residential": true` (and `residential_country`, for example `"ca"`, when the site serves a different page per country).
- Needs JavaScript: `"render_mode": "browser"`.
- Behind the user's sign-in: `persona_id` (see the `writ-signed-in-sites` skill). Never sign in yourself and never ask for a password.
- Long pages are cut at 12,000 characters (`_truncated`); `preview_chars: 0` returns the whole page.
- Need the raw HTML (selectors, embedded JSON): `"format": "html"` or `"both"`, single `url` only.

Discussion pages keep their threads, each comment tagged `[top-level]` or `[reply · depth N]`.

## Collect a site: `writ_crawl_site`

Use it when the result will be queried, exported, monitored or re-run: "crawl the docs", "every product in this category", "build a dataset of this site".

### Scope it first

An unscoped crawl of a real site collects hundreds of navigation and pagination pages and bills for each one. Pick the shape that matches the ask:

- A section ("the docs", "the pricing and blog pages"): `intent` in plain language (the server derives include and exclude paths from the site's real URLs), plus `relevance_threshold` around `0.3` to drop off-goal pages.
- Known pages as a dataset: `seed_urls`. For an immediate answer, `writ_scrape` with `urls` is cheaper.
- The top N of a listing as a dataset: `rank_cap` = N.
- The whole site: the defaults, with `page_budget` to cap the spend.

### Choose who reads each page

- **Markdown** (default, `extract_mode: "markdown"`): every page as clean text. No AI, fastest. Right for docs, articles and discussions.
- **Schema** (`extract_mode: "schema"` with `extract_schema`): every page holds the same record (a product, a listing) and you want rows. Deterministic extraction with a row selector and fields, no AI.
- **AI-assisted** (`executor: "ai"` with `extract_prompt`): the fields need understanding and vary per page, across many pages. Each page waits on a model call and bills 5x the page rate. Put the fields you want in `extract_prompt`; do not combine it with `extract_mode: "schema"`. Never use it for a few pages you could read yourself.

If the ask mentions comments or replies, pass `content_spec: {"preset": "full", "include_comments": true}` or they are stripped.

### Wait for it once

- A bounded crawl (`seed_urls` or `rank_cap`) waits and returns its rows in `data.rows` in the same call.
- An open crawl returns a `crawl_id`. Call `writ_crawl_status` with `crawl_id` and `wait: true` once: it holds up to 75 s and returns the rows when the crawl converges. If it is still running at the ceiling, call it once more the same way. Never poll in a loop.
- Rows longer than the preview arrive cut; fetch full records with `writ_workflow_data` using the crawl's `data_workflow_id` and `refs: ["<run_id>:<record_index>"]`.
- Original documents the crawl captured (PDFs, office files, images): `writ_crawl_files`.

### Save it only when it will be re-run

`save_as` stores the crawl settings as a named, callable crawl, listed to every future session as data already collected. Use it only when the user will re-run it (a recurring pull, an API they asked for), never for a one-off question.

- `writ_saved_crawls` lists them; `writ_run_saved_crawl` with `max_age` returns recent data without re-crawling; `writ_saved_crawl_data` reads the last run and never starts one; `writ_update_saved_crawl` edits the settings.
- A saved crawl is already an API: its answer carries `api.rest_endpoint`.
- When a program will consume the result, pass `output` (`{"shape": "records"}`, `{"shape": "record"}` for one entity, `fields` to pick and rename, `exclude` to drop). Saved with `save_as`, it becomes the endpoint's default shape.

## Data already collected

Before fetching again, check what the account already has: `writ_search_data` searches across everything collected, `writ_workflow_data` reads one dataset, `writ_export_data` returns CSV or JSON.

## Respect the site

Crawls honor `robots.txt` by default (`respect_robots: true`). Turn it off only when the user vouches for the target and a crawl reported a robots refusal, and then scope the crawl to exactly the pages they asked for.
