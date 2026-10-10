---
name: writ-signed-in-sites
description: Work on websites where the user is signed in, through Writ personas (saved sign-in identities whose passwords and 2FA stay sealed in Writ). Use when a task needs the user's own account on a site (email, social, shop, bank, work portal), a page behind a login, a two-factor prompt, a CAPTCHA or a "confirm it's you" check, or when the user would otherwise have to paste a password or a card number into the chat.
license: MIT
compatibility: Needs the Writ Cloud MCP server (https://api.usewrit.app/mcp) connected in the client; its tools are named writ_*.
---

# Sites the user is signed in to

A **persona** is a saved sign-in identity in Writ: a site's username, the password and any 2FA secret sealed server-side, and a warm signed-in session. You use one by passing its `persona_id`. You never see its credentials, and no credential passes through the conversation. Never ask the user for a password or a one-time code in chat.

## 1. Find the persona before anything starts

Call `writ_personas` with `action: "list"` and the site's `domain` (for example `"github.com"`; it matches subdomains). Do this before opening a browser, starting a crawl or a build: finding the login wall halfway through wastes the session.

- One fits: pass its `persona_id` to `writ_scrape`, `writ_crawl_site`, `writ_browser_use`, `writ_record_website`, `writ_website_to_api` or `writ_run_workflow`. The work runs signed in, from the persona's own fixed exit IP. `writ_website_to_api` takes a cloud persona's number only.
- Several fit: ask the user which account to use.
- `action: "get"` inspects one; `include_runs: true` adds its recent runs.

Read the list once per turn; do not list again what you just read.

## 2. None fits: the user creates one, then you continue

The `list` answer is then `persona_needed`, with a worded `tell_user`, a `create_url` and a `link_id`.

1. Relay `tell_user` and `create_url` to the user before starting the task. The link opens a small Writ window with only the persona form, pre-filled for that site.
2. Call `writ_personas` with `action: "wait"` and that `link_id`. It is one held call (up to 75 s) that answers the moment the user saves the persona, with its `persona_id`, or says they declined. On `still_waiting`, call it once more the same way.
3. Carry on with the task. The user does not need to come back and tell you.

To ask for another account on a site that already has one: `action: "request"` with `domain` and a one-line `why` the user sees in the window.

This tool cannot create or edit a persona, and no password field exists on it by design.

## 3. Keep the session warm

- A run that reports a sign-in or session error: `writ_personas` with `action: "sign_in"` and the `persona_id` runs its login now. `force: true` re-logs in even when the session looks usable.
- A persona that cannot sign itself in yet: `action: "record_login"` has Writ's server-side AI sign in once and save that flow as the persona's login workflow. Pass `login_url` when you know the exact sign-in page.

## 4. Two-factor codes, CAPTCHAs and decisions

- 2FA codes are minted server-side from the persona. In a browser session, identify the method the page is using before submitting the code; if it is not the persona's method, switch it on the page first ("Try another way").
- The `twofa` action of `writ_browser_act` takes that method as `challenge_method`: `sms` (a phone number, a text message), `email`, `authenticator` (an authentication app) or `other` (approve on the phone, a passkey, a QR code). A `twofa` without it is sent back once with `twofa_method_required`; `twofa_method_mismatch` means switch the method on the page first. Only `twofa_mint_failed` or `twofa_no_persona` means asking the user: `writ_browser_ask_user` with `kind: "twofa"`.
- When `writ_browser_act` reports a `security_check` (CAPTCHA, "confirm it's you"), a code Writ could not mint, or a decision only the account owner can make: call `writ_browser_ask_user` with the `session_id` and the `kind`. The user completes it in the live browser or answers, and the session continues. Never try to click through a CAPTCHA yourself.

## 5. The user's own desktop

Personas of the user's linked Writ desktop appear in the list with `source: "device"` and ids like `device:<agent>:<id>`. Passing one to a tool from step 1 (all but `writ_website_to_api`, which takes a cloud persona's number only) sends the work to that desktop, which signs in from its own vault, on that machine and its IP; the credentials never leave it. `writ_devices` lists the linked desktops.

## 6. Paying with the user's card

A card works like a persona: it stays sealed in Writ, and you never see, type or store its number. Never ask for a card number, expiry date or security code in chat, and never click the button that places an order: browse to the checkout, then hand it over with `writ_payment` (`request` a card the user approves, `checkout` at the order button, `wait` for the outcome). The user confirms the order (email, the Writ app or Writ Desktop) and Writ places it. The whole flow is in the `writ-payments` skill.

## Before acting on an account

Reading is safe. Anything that sends, posts, buys, deletes or changes a setting on the user's account needs their explicit go-ahead for that action, even when a persona makes it possible.
