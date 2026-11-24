# Design

Build work and open questions are tracked in the [v0.1.0 milestone](https://github.com/LittleAndi/threads-mcp-server/milestone/1). This file holds the design decisions only.

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

## Out of scope

- Publishing, replying, deleting, moderating.
- Webhooks.
- Multi-user support. This is a single-user tool for now.
