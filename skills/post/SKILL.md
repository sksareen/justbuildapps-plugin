---
name: post
description: Post an update about the app in this repo to its JustBuildApps page. Drafts from recent git activity in a lowercase, honest tone, shows it for approval, then publishes. Use when the user says /justbuildapps:post, "post to jba", "post an update to justbuildapps", "share what I just shipped", or after finishing a notable change worth putting on their app's feed.
argument-hint: "[what the update is about]"
---

# post: ship an update to JustBuildApps

You're posting an update to one of the user's projects ("creations") on JustBuildApps. Updates have a kind, a title, and a body. They're short, honest, lowercase.

Everything goes through the `justbuildapps` MCP server that this plugin connects. You never handle the user's token: Claude Code sends it to the server for you.

## Setup

Posting needs two things. Check them before drafting.

1. **A token.** The tools `create_creation` and `post_update` return `auth_required` when no token is set. If that happens, tell the user:
   - sign in at https://justbuildapps.com and pick a handle (free),
   - create a token at https://justbuildapps.com/settings/tokens and copy it,
   - run `/plugin`, open the JustBuildApps plugin, and paste the token into its **JustBuildApps token** setting, then run `/reload-plugins`.
   Never ask them to paste the token into the conversation, and never read it from a file.
2. **Their handle.** Read `~/.config/jba/handle` (one line, no `@`). If it is missing, ask for their handle and write it there. It is public, not a secret.

`~/.config/jba/repos.json` maps repos to projects: `{ "<absolute-repo-path>": "<creation-slug>", ... }`. Create it when you first need it.

## Inputs

The user runs `/justbuildapps:post` or asks you to "post an update". They may pass:
- A `kind` hint: `new` | `iteration` | `commentary` (optional; infer if missing)
- A project slug (optional; usually inferred from the repo)
- Free-form context like "about the dark mode work": `$ARGUMENTS`

## Steps

1. **Locate the repo.** `git rev-parse --show-toplevel`. If not in a git repo, ask which project to post to.

2. **Pick the project.** Call `get_creator` with the user's handle to list their creations (each has an `id` and a `slug`). Look up the repo path in `repos.json`. If there is no match, ask which creation to post to and offer to save the mapping.

   If this app has no page yet, offer to create one: draft `name`, `tagline`, `description`, `project_url`, and `repo_url` from the README and git remote, show the draft, and on an explicit yes call `create_creation`. Save the mapping and carry on with the returned id.

3. **Read git context.**
   ```bash
   git log -5 --oneline
   git log -1 --pretty=format:'%s%n%n%b'
   git diff HEAD~1 --stat
   ```
   Use the most recent meaningful commit (skip merges, "wip", tag-only). If the user gave free-form context, weight it heavily.

4. **Draft the update.** Output exactly this block, nothing else around it:

   ```
   ┌─ jba update draft ──────────────────────────
   creation: <name> (slug: <slug>, id: <id>)
   kind:     <new|iteration|commentary>
   title:    <≤100 chars, lowercase, no trailing period>

   <body: 1-3 short paragraphs, ≤2000 chars>
   └─────────────────────────────────────────────
   ```

   Then ask: **"ship it? (y / edit / cancel)"**

5. **Loop on edits.** If the user says "edit X" or rewrites text, redraft and show it again. Only proceed on an explicit `y` / `yes` / `ship`.

6. **Post it.** Call `post_update` with `creation_id`, `kind`, `title`, and `body`. On success, show the live URL, `https://justbuildapps.com/u/<handle>/<slug>`, and suggest they share that link wherever they announce the change.

**Media.** This plugin cannot upload local screenshots or videos yet. If the user wants media on the post, tell them to add it from the post's page on justbuildapps.com after it is live. If they give a public `https://` image or video URL, pass it in `media_urls`.

## Tone

JustBuildApps posts read like indie shipping logs. Match this style by default; the user can override.

- **lowercase everywhere**, including titles. No caps unless quoting a name (Android, iOS, GitHub).
- short sentences. one idea per line.
- direct and honest. acknowledge what doesn't work. no marketing speak.
- no emojis. no hashtags. no exclamation points except when natural.
- words to avoid: delve, holistic, robust, leverage, journey, thrilled, excited to announce.

**Good titles:** "fixed the accelerometer on android", "stripped like/skip from /explore", "high scores save now", "switched the llm to gemini 2.5 flash"

**Good bodies:** Set the change in one line. Add why or what was surprising. End with the truth (what's left, what still doesn't work). Like: *"localstorage. not even sql. you score, it sticks. that's the whole feature."*

## Kind inference

- `new` — the creation itself just launched / first time it's live. Rare.
- `iteration` — a feature, fix, refactor that ships. Default for most commits.
- `commentary` — a thought, retrospective, decision-not-yet-built. No code change required.

If the diff is meaningful code, it's `iteration`. If the message is reflective with no code, it's `commentary`.

## Errors

- **`auth_required` or 401** → no token, or it is wrong or revoked. See Setup.
- **403 "Not your creation"** → slug resolved to a creation owned by someone else. Re-check `repos.json` and `handle`.
- **404 "Creation not found"** → bad creation_id, re-fetch creator.
- **5xx** → show the response, ask user whether to retry.

## Don't

- Don't post without an explicit `y` from the user. Approval is the whole point.
- Don't invent detail that isn't in the diff or the user's context.
- Don't write more than 3 paragraphs of body unless the user asks.
- Don't add a trailing summary like "this is a great update!". The post is the output.
- Don't ask for, read, print, or store the user's token.
