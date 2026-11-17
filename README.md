# JustBuildApps plugin for Claude Code

[JustBuildApps](https://justbuildapps.com) is a catalog of AI-built apps and indie tools, each with a live link you can try. This plugin connects it to Claude Code.

```
/plugin install jba --marketplace sksareen/justbuildapps-plugin
```

On Claude Code older than 2.1.275: `/plugin marketplace add sksareen/justbuildapps-plugin`, then `/plugin install jba@justbuildapps`.

## What you get

- **Find before you build.** Ask "is there an app for X?" or run `/jba:find <what you need>`. Claude searches the catalog, checks the matches are up, and gives you live links. No account needed.
- **`/jba:post`: post what you ship.** After you finish a change, run `/jba:post`. Claude drafts a short update from your latest commits, shows it to you, and publishes it to your app's page on JustBuildApps once you say yes. Attach screenshots or a video and they go with it. If your app has no page yet, it offers to create one.
- **The JustBuildApps MCP server**, connected for you: search, browse, similar apps, trending, health checks, and the tools for managing your own listings.

## Setup for posting

Searching needs nothing. Posting needs a free account and a personal token:

1. Sign in at [justbuildapps.com](https://justbuildapps.com) and pick a handle.
2. Create a token at [justbuildapps.com/settings/tokens](https://justbuildapps.com/settings/tokens) and copy it.
3. Run `/jba:post`. The first time, it walks you through saving the token to `~/.config/jba/token` on your machine.

Nothing is published without your explicit yes. The token stays on your machine and is only sent to justbuildapps.com.

## Links

- MCP server and per-client install: [justbuildapps.com/mcp](https://justbuildapps.com/mcp)
- API reference: [justbuildapps.com/developers](https://justbuildapps.com/developers)
