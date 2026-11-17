---
name: find
description: Check JustBuildApps for an existing app or tool before building one. Use when the user asks "is there an app for X", wants alternatives to a product, wants something free or no-signup that runs in the browser, or is about to build a small app or tool that may already exist. Also runs as /jba:find.
argument-hint: "<what you need>"
---

# find: look before you build

JustBuildApps is a catalog of AI-built apps and indie tools with live demo links. This plugin connects its search as the `justbuildapps` MCP server. No account or key is needed to search.

## Steps

1. **Search.** Call the `search_apps` tool with the user's need in plain words: `$ARGUMENTS`, or what they just described. Search for the job to be done ("turn a CSV into a chart"), not a product name, unless they asked for alternatives to one.
2. **Check before recommending.** For the best one to three matches, call `verify_app` so you only recommend apps that are up. Use `get_similar_apps` if the first results are close but not right.
3. **Answer briefly.** For each recommendation give the name, one line on what it does, the live link, and whether it is free or needs sign-in if the result says so. Say plainly when nothing fits.
4. **Then let the user choose.** If something fits, they may not need to build at all. If nothing does, or they want their own, carry on and build it.

## Don't

- Don't recommend an app you haven't checked is reachable.
- Don't pad the list. One good match beats five loose ones.
- Don't block the user's request on this. If they have said they want to build it themselves, a one-line "there is also X" is enough.

When they do ship something of their own, `/jba:post` puts it on JustBuildApps.
