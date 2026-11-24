# Threads MCP Server

Read-only MCP server for the [Meta Threads API](https://developers.facebook.com/documentation/threads): search posts, look up profiles, and read your own posts, replies and insights.

> **Status: early development.** Nothing is implemented yet. The tools below are planned. See [docs/plan.md](docs/plan.md).

## Read this first: App Review limits what search can see

Threads only lets an app search other people's posts after Meta approves it in **App Review**. Until then:

| Capability | Before approval | After approval |
|---|---|---|
| Keyword search (`threads_keyword_search`) | Your **own** posts only | All public posts |
| Profile lookup and profile posts (`threads_profile_discovery`) | Posts of accounts added as **testers** only | Public profiles |
| Your own posts, replies and insights | Works | Works |

So if you run this server with your own Meta app and have not been through App Review, `search_posts` will not find anyone else's posts. This is a Threads platform rule, not a bug in the server. The tool output states which scope was used.

Keyword search is also rate limited to **2,200 queries per user in a rolling 24 hours**, shared across all apps that use the same account. The server never searches in a loop and keeps a local count so you can see how much you have used.

## Planned tools

| Tool | Description | Works before App Review? |
|---|---|---|
| `search_posts` | Keyword or topic-tag search | Own posts only |
| `get_search_quota` | Search queries used in the last 24 hours | Yes |
| `get_profile` | Public profile info | Testers only |
| `get_profile_posts` | Posts of a profile | Testers only |
| `get_my_posts` | Your own posts | Yes |
| `get_post` | A single post | Yes |
| `get_replies` | Replies to a post | Yes |
| `get_post_insights` | Metrics for one of your posts | Yes |
| `get_account_insights` | Metrics for your account | Yes |

Every post in a result includes its `permalink`.

## What it will not do

- Publish, reply, delete, hide or repost. The server is read-only and never requests the publishing or reply-management permissions.
- Scrape Threads. It only uses the official API.

## Setup

1. Create a Meta app with the **Threads use case** at [developers.facebook.com](https://developers.facebook.com/).
2. Add your Threads account as a tester and generate a long-lived access token.
3. Copy `.env.example` to `.env` and fill in the values. Keep `.env` out of version control.

Tokens, the app secret and other secrets are only read from the environment. They are never logged or included in tool output.

Detailed setup, token refresh and example prompts will be added as the tools are built.

## Documentation

- [docs/threads-api.md](docs/threads-api.md): what the Threads API allows, its limits, and what is still unverified.
- [docs/plan.md](docs/plan.md): tool list, build phases and open questions.

## License

[MIT](LICENSE)
