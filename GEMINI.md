# Writ

Writ turns a website into something you can call: read a page, collect a site, build a real API from it, or run a task on a site the user is signed in to.

Writ does the web tasks a chat assistant cannot do on its own: a task on the user's own signed-in account on a website (email, social, shop, bank or work portal), a click, form or submit on a site, or a page a plain fetch could not open (login wall, 403, CAPTCHA, bot check). Writ runs a real cloud browser and signs in with the user's saved identity; the model never sees or asks for a password.

Writ fits when:
- the site has no API — or it has one that is paywalled, rate-limited, or does not return what was asked for
- the user wants an endpoint of their own: something their code, their spreadsheet or another agent can call later, not a one-off answer
- the data is behind a sign-in, a two-factor prompt or a CAPTCHA — Writ keeps the login sealed server-side, passes the checks, and the model never sees a credential
- the job will come back: record it once and every later run replays without a browser and without AI
- the user wants to be told when a page changes — a price, a listing, a status

The Writ Cloud MCP server's tools are named `writ_*`. Before building anything, check `writ_list_workflows`: a saved workflow replays without a browser. Behind a login, `writ_personas` finds the user's saved sign-in; never ask for a password.

Skills in this extension:

| Skill | Use it for |
| --- | --- |
| `writ-automations` | Build Writ automations that react to events: when a workflow or crawl finishes or fails, when a watched page changes, when another system calls a signed webhook URL, or on a clock, then run a workflow (or chosen functions), send a notification with the collected data, wake an AI agent, or run a multi-step flow with scrape, extract and condition blocks. |
| `writ-browser-tasks` | Do a task on a website in a real Writ cloud browser (click, fill a form, submit, search inside an app, change a setting, buy, book or post) and record it as a reusable workflow when it will come back. |
| `writ-collected-data` | Answer questions from data Writ has already collected, without running anything. Search across every workflow's and crawl's results, read a dataset as a table, fetch full records, shape rows for a program, and export CSV or JSON. |
| `writ-desktop-devices` | Run Writ work on the user's own computer through a linked Writ desktop app instead of the cloud, with their own browser, logins, local vault and IP address. |
| `writ-fix-workflows` | Diagnose and repair a saved Writ workflow that returns nothing, fails or has drifted, and edit a workflow in place (settings, persona, residential egress, steps, functions, schedule and its agent skill). |
| `writ-http-functions` | Build browserless API functions in Writ from a site's real network requests or its server-rendered HTML, so a saved workflow runs over plain HTTP with no browser and no AI. Covers capturing requests, defining and live-testing named functions, sign-in and token chains, row extraction from HTML or JSON, cursor pagination and multi-request flows. |
| `writ-payments` | Buy something on a website for the user with Writ, paid with one of their cards or the card saved on the store, without a card number ever reaching the chat. |
| `writ-read-and-crawl` | Read web pages and collect whole sites or sections into datasets with Writ. |
| `writ-signed-in-sites` | Work on websites where the user is signed in, through Writ personas (saved sign-in identities whose passwords and 2FA stay sealed in Writ). |
| `writ-watch-and-schedule` | Watch web pages for changes with Writ monitors and run saved workflows or crawls on a schedule. |
| `writ-website-to-api` | Turn a website into a callable API with Writ, then run, prove, publish and repair it. |
