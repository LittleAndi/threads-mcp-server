# Plan

## Why

This server lets an MCP client work with Threads. It has two goals:

1. **Search** Threads for relevant conversations and people (main goal).
2. **Track your own posts** and their insights straight from the API.

## Tools

| Tool | Endpoint | Works pre-approval? |
|---|---|---|
| `search_posts` | `/keyword_search` | Own posts only |
| `get_profile` | `/profile_lookup` | Testers only |
| `get_profile_posts` | `/profile_posts` | Testers only |
| `get_my_posts` | `/me/threads` | Yes |
| `get_post` | `/{media-id}` | Yes |
| `get_post_insights` | `/{media-id}/insights` | Yes |
| `get_account_insights` | `/{user-id}/threads_insights` | Yes |
| `get_replies` | `/{media-id}/replies` or `/conversation` | Yes |
| `get_search_quota` | local counter (+ API if available) | Yes |

### `search_posts` details

- Inputs: `query` (required), `mode` (keyword|tag), `sort` (top|recent),
  `media_type`, `since`, `until`, `author`, `limit` (≤100).
- Accept `author` with or without a leading `@` and strip it.
- Accept ISO dates and convert them to Unix time. Validate `since` ≥ 1688540400.
- Output: list of `{id, username, text, timestamp, permalink, media_type,
  is_reply, is_quote_post, has_replies}` plus a `meta` block: `scope`
  (`own_posts` | `public`), `quota_used_24h`, `possibly_filtered` (true on
  an empty result).
- Detect the scope from config (`THREADS_SEARCH_SCOPE=own|public`, default
  `own`) so the tool description and output stay honest.

### Quota tracking

- Log a timestamp for each search that returned results. Report the count
  over the last 24 h against 2,200.
- Store the log in a small local file (path configurable). No database.

### Token handling

- Load the long-lived token from env/config.
- On startup and once a day: if the token is more than 24 h old and close
  to expiry (for example 7 days left), refresh it and persist the new one.
- Error clearly when the token is expired and needs re-authorisation.
  Never print the token.

## Phases

0. **Docs check (Meta MCP):** work through the *unverified* items in
   `docs/threads-api.md` using the Meta developer tools MCP server, and
   check the App Review requirements. See "Verifying with the Meta MCP"
   there for what it can and cannot cover.
1. **Spike (manual):** create the Meta app (Threads use case), add
   your account as a tester, get a token in the Graph API Explorer,
   call `/keyword_search` and `/me/threads`. Record the actual response
   shapes in `docs/threads-api.md` and clear the *unverified* markers.
2. **Scaffold:** copy the structure and tooling from `smhi-mcp-server`.
   Add config, an HTTP client with error mapping (Graph API error object
   → readable MCP error) and the token loader.
3. **Own-data tools:** `get_my_posts`, `get_post`, `get_post_insights`,
   `get_account_insights`, `get_replies`.
4. **Search:** `search_posts` and `get_search_quota`.
5. **Discovery:** `get_profile` and `get_profile_posts`. Test them against
   a tester account until approval.
6. **Docs and release:** README with setup (Meta app creation, token flow),
   the App Review limitation stated up front, example prompts.
7. **App Review (parallel):** privacy policy and ToS pages (e.g. GitHub
   Pages), screencast script, submit `threads_keyword_search` and
   `threads_profile_discovery`.

## Open questions

- Can an individual (no business verification) pass App Review for
  `threads_keyword_search`?
- Does `/keyword_search` paginate?
- Is there an API that reports keyword-search quota?
- Exact insights metric names, and whether follower vs non-follower views
  are exposed.

## Out of scope

- Publishing, replying, deleting, moderating.
- Webhooks.
- Multi-user support. This is a single-user tool for now.
