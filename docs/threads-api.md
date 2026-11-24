# Threads API notes

Researched 2026-11-24 from Meta's docs. Endpoint names, parameters, limits
and permissions were checked against the official pages (linked inline) with
the `meta_devtools` MCP. Anything still marked **(unverified)** needs a live
call. Response shapes and pagination behaviour are not covered by the docs.

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
| `/profile_lookup`, `/profile_posts` (`threads_profile_discovery`) | Only @meta, @threads, @instagram and @facebook | Public profiles with 100+ followers |
| `/{media-id}` for other users' posts (advanced `threads_basic`) | Only posts by Threads **testers** on the app | Public posts |
| `/{user-id}/mentions` (`threads_manage_mentions`) | Only mentions made by a Threads tester | Public mentions |
| Own posts, replies, insights | Works in development mode | Same |

- App Review needs a screencast, a hosted privacy policy and terms of
  service, and is done per permission. Third-party reports put it at about
  2–7 working days per permission, with first-round rejections common.
- **Open question:** whether an individual developer without business
  verification can get `threads_keyword_search` approved.
- Profile discovery only works for public profiles with at least 100 followers
  (lowered from 1,000 in April 2026), and is limited to 1,000 requests per
  user per rolling 24 h. See [Threads profiles](https://developers.facebook.com/documentation/threads/threads-profiles).

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
- Pagination is not documented for this endpoint **(unverified live)**: the
  [keyword search page](https://developers.facebook.com/documentation/threads/keyword-search) shows no `paging` object in
  any example response and lists no `before`/`after` parameters. Treat it
  as a single page of at most `limit` (max 100) results. Whether a
  `paging` object ever appears is a live-response question for the spike.
- `GET /me?fields=recently_searched_keywords` returns the user's recent
  queries as `[{query, timestamp}]` (same page).

### Search quota

- **2,200 queries per user, rolling 24 h**, counted across *all apps*
  (changed 2026-02-13). Older blogs say "500 per 7 days", which is outdated.
- Repeat queries count. Queries with no results do not count.
- No endpoint reports keyword-search quota usage.
  `GET /{user-id}/threads_publishing_limit` has `fields` `quota_usage`,
  `config`, `reply_quota_usage`, `reply_config`, `delete_quota_usage`,
  `delete_config`, `location_search_quota_usage`, `location_search_config`
  and nothing for keyword search
  ([User reference](https://developers.facebook.com/documentation/threads/reference/user)). Track usage locally.
  `recently_searched_keywords` might help as a cross-app count
  **(unverified)**: test whether it is complete and how far back it goes.

## Profile discovery

- `GET /profile_lookup?username=` returns basic public profile info.
- `GET /profile_posts?username=` returns a paginated list of a profile's
  posts (cursor `paging.cursors.before/after`; also `since`, `until`,
  `limit`). Same 100-follower and 1,000/24 h limits.
  Verified in the [User reference](https://developers.facebook.com/documentation/threads/reference/user) and
  [Retrieve user posts](https://developers.facebook.com/documentation/threads/retrieve-and-discover-posts/retrieve-posts).
- `profile_lookup` fields: `username`, `name`, `profile_picture_url`,
  `biography`, `follower_count`, `likes_count`, `quotes_count`,
  `reposts_count`, `views_count`, `is_verified`. The counts cover the
  past 7 days. The doc's example response also shows `replies_count`,
  which its field table omits.

## Own data (works without App Review)

Verified against the official docs on 2026-11-24. Response shapes still
need a live check.

- `GET /{user-id}/threads?fields=...`: own posts, paginated
  (`paging.cursors.before/after`, `since`, `until`, `limit`). `me` works
  as the user ID ([Retrieve user posts](https://developers.facebook.com/documentation/threads/retrieve-and-discover-posts/retrieve-posts)).
- `GET /{media-id}?fields=...`: a single post. Without advanced
  `threads_basic`, only posts by Threads testers are retrievable.
- Post fields: `id`, `media_product_type`, `media_type` (`TEXT_POST`,
  `IMAGE`, `VIDEO`, `CAROUSEL_ALBUM`, `AUDIO`, `REPOST_FACADE`),
  `media_url`, `permalink`, `text`, `username`, `timestamp`, `shortcode`,
  `is_quote_post`, `has_replies`, `link_attachment_url`, and more. See the
  page above for the full list.
- `GET /{media-id}/insights?metric=...` (`threads_manage_insights`):
  `views`, `likes`, `replies`, `reposts`, `quotes`, `shares`. `views` and
  `shares` are labelled "in development". Returns an empty array for
  `REPOST_FACADE` posts. Replies counts exclude nested replies' metrics
  ([Insights](https://developers.facebook.com/documentation/threads/insights)).
- `GET /{user-id}/threads_insights?metric=...` (`threads_manage_insights`):
  `views` (time series), `likes`, `replies`, `reposts`, `quotes`,
  `followers_count`, `clicks` (per link), `follower_demographics` (100+
  followers; needs one `breakdown`: `country`, `city`, `age` or `gender`).
  `since`/`until` must be on or after 1712991600. `followers_count` and
  `follower_demographics` ignore `since`/`until`.
- `GET /{media-id}/replies`: top-level replies only. `GET /{media-id}/conversation`:
  all replies flattened, root posts only. Both paginated, with a `reverse`
  parameter (default `true`)
  ([Replies](https://developers.facebook.com/documentation/threads/retrieve-and-manage-replies/replies-and-conversations)).
  `GET /{user-id}/replies` lists replies a user wrote.
- `GET /{user-id}/mentions` needs `threads_manage_mentions`. Private
  users' posts are not returned ([Mentions](https://developers.facebook.com/documentation/threads/threads-mentions)).

## Tokens

Verified against [Long-lived tokens](https://developers.facebook.com/documentation/threads/get-started/long-lived-tokens).

- Short-lived token: 1 h. Exchange it for a long-lived token (60 days).
  Expired short-lived tokens cannot be exchanged.
- A long-lived token can be refreshed once it is at least **24 h old** and
  before it expires. A token not refreshed within 60 days expires for good
  and the user has to re-authorise. Most integrations break here.
- Exchange (server-side only, it carries the app secret):
  `GET https://graph.threads.com/access_token?grant_type=th_exchange_token&client_secret=...&access_token=...`
  Response: `{access_token, token_type: "bearer", expires_in}` in seconds.
- Refresh:
  `GET https://graph.threads.com/refresh_access_token?grant_type=th_refresh_token&access_token=...`
  Same response shape. Both endpoints are unversioned, with no `/v1.0`.
- Permission grants last 90 days. Refreshing extends the grant only for
  public profiles. For private profiles the user must re-authorise
  when it expires.

## Rate limits (general)

- App call count is per app–user pair in a rolling 24 h window.
- Search is the scarce resource. Everything else has plenty of headroom
  for a single-user tool.
