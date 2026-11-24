# threads-mcp-server

An MCP server for the Meta Threads API, focused on **search and reading**.
Posting is out of scope.

Owner: LittleAndi (https://github.com/LittleAndi). This is a public repo, a sibling of
`smhi-mcp-server` and `trafikverket-mcp-server`.

## Read first

- `docs/threads-api.md`: what the Threads API allows, its limits, and what
  is still unverified. Check it before writing any endpoint code.
- `docs/design.md`: the tool list and design decisions. Build work and open
  questions are GitHub issues in the v0.1.0 milestone.

## Conventions

- **Mirror `smhi-mcp-server`** (github.com/LittleAndi/smhi-mcp-server):
  same language, MCP SDK, project layout, config style, README structure,
  test setup and CI. Clone or read it before scaffolding. Do not introduce
  a new stack.
- Tool names are snake_case verbs: `search_posts`, `get_profile`, ...
- Tool descriptions are written for an LLM. State the limits that change
  how a result should be read, for example "results limited to your own
  posts until App Review is approved".
- Return compact JSON. Include `permalink` on every post so results can
  be clicked through.

## Hard rules

- **Read-only.** Do not call publish, reply, delete, hide or repost
  endpoints. Do not request `threads_content_publish` or
  `threads_manage_replies`.
- **Official API only.** No scraping, no session cookies, no third-party
  scraper services (Apify, ScrapeCreators, ...). These break Meta's terms
  and would not belong in a public repo.
- **Secrets:** the app secret and tokens come from env/config only. Never
  log them, never echo them in tool output or errors, never commit them.
  `.env` goes in `.gitignore` from the first commit. Ship `.env.example`.
- **Quota awareness:** keyword search is limited per user across *all*
  apps (see the API notes). Never loop or fan out searches automatically.
- Do not invent Threads API fields or metrics. If something is marked
  *unverified* in `docs/threads-api.md`, check the official docs or test it
  in the Graph API Explorer first, then update the doc.
- **Verification tooling:** the Meta developer tools MCP server
  (`meta_devtools`) can search and read the official Threads docs and check
  the app's review status. It cannot call Threads endpoints. See
  "Verifying with the Meta MCP" in `docs/threads-api.md` for what it covers.

## Environment

```
THREADS_APP_ID=
THREADS_APP_SECRET=
THREADS_ACCESS_TOKEN=        # long-lived user token (60 days)
THREADS_API_BASE=https://graph.threads.net/v1.0
```

## Done means

- Tools tested against the live API with the maintainer's own account.
- The README explains the App Review limitation up front, in plain words.
- `docs/threads-api.md` is updated with anything learned while building.
