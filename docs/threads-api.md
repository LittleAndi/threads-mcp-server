# Threads API notes

Researched 2026-11-24 from Meta's docs. Items marked **(unverified)** are
from memory or third-party sources and must be checked before use.

Docs: https://developers.facebook.com/documentation/threads
Changelog: https://developers.facebook.com/documentation/threads/changelog

## Verifying with the Meta MCP

Meta publishes a developer tools MCP server (`meta_devtools`). Use it to
check the claims in this file against Meta's own sources. Sign in with the
Meta account that owns the app.

**What it can do**

| Need | Tools |
|---|---|
| Confirm parameters, limits, permissions and token endpoints in the official docs | `devtools_discovery` (`search_docs`), then `devtools_docs` to read the page |
| Check whether a permission needs App Review, and the review status and requirements | `devtools_app_review` (`status`, `requirements`, `privileges`, `history`) |
| Check the app's use case, settings and compliance | `devtools_app_use_cases`, `devtools_app`, `devtools_compliance` |
| Watch rate limits, call volume and deprecations | `devtools_api_usage` |
| Catch API changes | `devtools_api_changelog` |

**What it cannot do**

It has no tool that calls Graph API endpoints such as `/keyword_search`.
Anything about real responses still has to be tested in the Graph API
Explorer or with this server and a real token:

- response shapes and field names,
- whether `/keyword_search` paginates,
- how sensitive keywords behave (empty array),
- actual quota consumption.

**How to use it**

1. Go through each item marked *unverified* in this file. Search the docs
   for it, read the page, and either clear the marker with a link to the
   page or correct the entry.
2. Answer the App Review open questions with `devtools_app_review` before
   starting the review.
3. For everything it cannot cover, test live and record the result here.
4. Do not copy app IDs, business IDs or tokens from tool output into the repo.

## Basics

- Base URL: `https://graph.threads.net/v1.0` (`graph.threads.com` also works).
- Create a Meta app with the **Threads use case**. Use the *Threads* app
  ID and secret, not the Facebook ones.
- `threads_basic` is required for every call.

## The big limitation: App Review

| Capability | Without approval | After approval |
|---|---|---|
| `/keyword_search` (`threads_keyword_search`) | Only the authenticated user's **own** posts | All public posts |
| `/profile_lookup`, `/profile_posts` (`threads_profile_discovery` + advanced `threads_basic`) | Only Threads **testers'** posts | Public profiles |
| Own posts, replies, insights | Works in development mode | Same |

- App Review needs a screencast, a hosted privacy policy and terms of
  service, and is done per permission. Third-party reports put it at about
  2–7 working days per permission, with first-round rejections common.
- **Open question:** whether an individual developer without business
  verification can get `threads_keyword_search` approved.
- Profile discovery only works for profiles with at least 100 followers
  (lowered from 1,000 in April 2026).

## Keyword search — `GET /keyword_search`

| Param | Notes |
|---|---|
| `q` | Required. Keyword(s). |
| `search_type` | `TOP` (default) or `RECENT` |
| `search_mode` | `KEYWORD` (default) or `TAG` (topic tag) |
| `media_type` | `TEXT`, `IMAGE`, `VIDEO`; omit for all |
| `since` / `until` | Unix ts or strtotime string. `since` ≥ 1688540400, `until` ≤ now |
| `limit` | Default 25, max 100 |
| `author_username` | Exact match, no `@` |
| `fields` | e.g. `id,text,media_type,permalink,timestamp,username,has_replies,is_quote_post,is_reply` |

- `owner` is never returned. No engagement counts in search results.
- Sensitive or offensive keywords return an **empty array**. Report that
  as "possibly filtered", not "no results".
- Pagination is not documented for this endpoint. Check the response for
  a `paging` object **(unverified)**.

### Search quota

- **2,200 queries per user, rolling 24 h**, counted across *all apps*
  (changed 2026-02-13). Older blogs say "500 per 7 days", which is outdated.
- Repeat queries count. Queries with no results do not count.
- No known endpoint reports keyword-search quota usage. Check whether
  `threads_publishing_limit` has a field for it **(unverified)**.
  Otherwise track usage locally.

## Profile discovery

- `GET /profile_lookup?username=` returns basic public profile info.
- `GET /profile_posts?username=` returns a paginated list of a profile's
  posts (cursor `paging.cursors.before/after`).

## Own data (works without App Review)

All **(unverified)**. Confirm names and metrics in the Reference docs.

- `GET /me/threads?fields=...`: own posts, paginated.
- `GET /{media-id}?fields=...`: a single post.
- `GET /{media-id}/insights?metric=views,likes,replies,reposts,quotes,shares`:
  per-post metrics (`threads_manage_insights`).
- `GET /{user-id}/threads_insights?metric=...`: account metrics such as
  views and followers_count (`threads_manage_insights`).
- `GET /{media-id}/replies` and `/{media-id}/conversation`: replies.
- Mentions of your account need `threads_manage_mentions`.

## Tokens

- Short-lived token: 1 h. Exchange it for a long-lived token (60 days).
- A long-lived token can be refreshed between **24 h and 60 days** after
  it was issued. Miss that window and the user has to re-authorise. Most
  integrations break here.
- Exchange: `GET /access_token?grant_type=th_exchange_token&client_secret=...&access_token=...`
  **(unverified)**
- Refresh: `GET /refresh_access_token?grant_type=th_refresh_token&access_token=...`
  **(unverified)**
- Permissions granted for private Threads profiles expire after 90 days,
  whatever the token state.

## Rate limits (general)

- App call count is per app–user pair in a rolling 24 h window.
- Search is the scarce resource. Everything else has plenty of headroom
  for a single-user tool.
