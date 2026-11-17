---
name: post
description: Post an update about the app in this repo to its JustBuildApps page. Drafts from recent git activity in a lowercase, honest tone, shows it for approval, then publishes. Use when the user says /jba:post, "post to jba", "post an update to justbuildapps", "share what I just shipped", or after finishing a notable change worth putting on their app's feed.
argument-hint: "[what the update is about]"
---

# post: ship an update to JustBuildApps

You're posting an update to a "creation" on JustBuildApps (the creator feed at `/u/<handle>`). Updates have a kind, title, body — they're short, honest, lowercase.

## Inputs

The user invokes via `/jba:post` (manual) or asks you to "post an update". They may pass:
- A `kind` hint: `new` | `iteration` | `commentary` (optional — infer if missing)
- A creation slug override (optional — usually inferred from cwd)
- Free-form context like "about the dark mode work"

## Config

Three files in `~/.config/jba/`:
- `handle` — the user's JustBuildApps handle (single line, no `@`). Used to look up their creator profile.
- `token` — the user's personal API token, which starts with `jba_`. They create it at https://justbuildapps.com/settings/tokens. A bare value, or an env-style line like `JBA_WRITE_TOKEN=jba_xxx` (the skill strips the prefix).
- `repos.json` — `{ "<absolute-repo-path>": "<creation-slug>", ... }`

### First-time setup

If `~/.config/jba/token` or `~/.config/jba/handle` is missing, stop and walk the user through this before anything else:

1. **Account.** They need a JustBuildApps account with a creator handle. If they don't have one, send them to https://justbuildapps.com to sign in and pick a handle. It is free.
2. **Token.** Have them open https://justbuildapps.com/settings/tokens, create a token, and copy it. The token is shown once.
3. **Save it without pasting it into the chat.** Ask them to run this themselves (in Claude Code, prefix it with `!`), with the token on their clipboard:
   ```bash
   mkdir -p ~/.config/jba && pbpaste | tr -d '\n\r ' > ~/.config/jba/token && chmod 600 ~/.config/jba/token
   ```
   On Linux, use `xclip -o -selection clipboard` or `wl-paste` in place of `pbpaste`. Never ask them to type the token into the conversation, and never print it.
4. **Handle.** Ask for their handle and write it to `~/.config/jba/handle` (no `@`).
5. **Check.** `GET /api/v1/creators/$JBA_HANDLE` should return their profile. Then continue with the steps below.

API base: defaults to `https://justbuildapps.com/api/v1`. Override via the `JBA_API_BASE` env var.

Token + handle extraction one-liners (use these in bash):
```bash
JBA_TOKEN=$(awk -F= '{ if ($1=="JBA_WRITE_TOKEN") print $2; else print $0 }' ~/.config/jba/token | tr -d '\n\r ')
JBA_HANDLE=$(tr -d '\n\r @' < ~/.config/jba/handle)
```

## Steps

1. **Locate the repo.** `git rev-parse --show-toplevel`. If not in a git repo, ask the user which creation slug to post to.

2. **Pick the creation slug.** Read `~/.config/jba/repos.json` and look up the absolute repo path. If there's no match, list known creations from `GET /api/v1/creators/$JBA_HANDLE` and ask which one to post to (and offer to add the mapping to `repos.json`). If this app has no page yet, offer to create one: draft `name`, `tagline`, `description`, `project_url`, and `repo_url` from the README and git remote, show the draft, and on an explicit yes call `POST /creations` as described under "Managing creations" below. Then save the mapping and carry on.

3. **Read git context.**
   ```bash
   git log -5 --oneline
   git log -1 --pretty=format:'%s%n%n%b'
   git diff HEAD~1 --stat
   ```
   Use the most recent meaningful commit (skip merges, "wip", tag-only). If the user gave free-form context, weight it heavily.

4. **Resolve creation_id.** `GET /api/v1/creators/$JBA_HANDLE` (no auth needed), find the creation matching the slug, grab its `id`. Cache in memory for this turn.

4a. **Check for media.** If the user attached one or more images, gifs, or videos to the chat, each appears as a file path on disk (typically `/tmp/...` or under `~/Pictures/...`, often referenced with an `image_path` attribute or surfaced in a `<system-reminder>`). Treat them as media attachments for this post — order matters (first attachment is the cover, the rest become a carousel).

   In the draft block, list them on a single `media:` line with the cover first, separated by `, `:
   - One file: `media:    cover.png`
   - Multiple:  `media:    cover.png, second.png, third.png`

   If no media is attached, skip this step and omit the `media:` line.

   Allowed types: `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `video/mp4`, `video/webm`, `video/quicktime`. Cap is 20MB per file. Max 8 files per post.

5. **Draft the update.** Output exactly this block to the user, nothing else around it:

   ```
   ┌─ jba update draft ──────────────────────────
   creation: <name> (slug: <slug>, id: <id>)
   kind:     <new|iteration|commentary>
   media:    <filename(s) or omit if none>
   title:    <≤100 chars, lowercase, no trailing period>

   <body — 1-3 short paragraphs, ≤2000 chars>
   └─────────────────────────────────────────────
   ```

   Then ask: **"ship it? (y / edit / cancel)"**

6. **Loop on edits.** If user says "edit X" or rewrites text, redraft and re-show. Only proceed on explicit `y` / `yes` / `ship`.

7. **Upload media (if attached).** Only after approval. The upload endpoint is bearer-authed and returns one URL per file. Upload each file in order, collecting URLs into a JSON array.
   ```bash
   JBA_TOKEN=$(awk -F= '{ if ($1=="JBA_WRITE_TOKEN") print $2; else print $0 }' ~/.config/jba/token | tr -d '\n\r ')
   JBA_API_BASE=${JBA_API_BASE:-https://justbuildapps.com/api/v1}

   # Repeat per file, in attachment order:
   URL_1=$(curl -sS -X POST "$JBA_API_BASE/media/upload" \
     -H "Authorization: Bearer $JBA_TOKEN" \
     -F "file=@<path-to-file-1>" \
     | python3 -c 'import json,sys; print(json.load(sys.stdin).get("url",""))')
   # URL_2=...   # repeat for each additional file

   # Build the JSON array for the post body:
   MEDIA_URLS_JSON=$(python3 -c 'import json,sys; print(json.dumps([u for u in sys.argv[1:] if u]))' "$URL_1" "$URL_2")
   ```
   Each URL looks like `/api/v1/media/<uuid>.<ext>`. If any upload fails, show the error and ask whether to ship without media (or with the partial set).

8. **POST it.** Include `media_urls` (array) in the body if step 7 ran successfully. The server keeps `media_url` in sync with `media_urls[0]` so old clients still work.
   ```bash
   curl -sS -X POST "$JBA_API_BASE/creations/$CID/updates" \
     -H "Authorization: Bearer $JBA_TOKEN" \
     -H "Content-Type: application/json" \
     -d "{\"kind\":\"<kind>\",\"title\":\"<title>\",\"body\":\"<body>\",\"media_urls\":$MEDIA_URLS_JSON}"
   ```
   For zero attachments, omit `media_urls` entirely. For exactly one, you can use either `"media_urls":["<URL>"]` or the legacy `"media_url":"<URL>"` — both are accepted.

   On success, show the response (id, published_at) and the live URL: `https://justbuildapps.com/u/$JBA_HANDLE`. Suggest they share that link wherever they announce the change.

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

- **401** → token missing, wrong, or revoked. Tell the user to create a new one at https://justbuildapps.com/settings/tokens and save it to `~/.config/jba/token` (see First-time setup).
- **403 "Not your creation"** → slug resolved to a creation owned by someone else. Re-check `repos.json` and `handle`.
- **404 "Creation not found"** → bad creation_id, re-fetch creator.
- **5xx** → show the response, ask user whether to retry.

## Don't

- Don't auto-post without explicit `y` from the user. HITL is the whole point.
- Don't fabricate detail not present in the diff or user-provided context.
- Don't write more than 3 paragraphs of body unless the user asks.
- Don't add a trailing summary like "this is a great update!" — the post IS the output.

## Managing creations (agent API)

Beyond posting updates, an agent can create and edit the creations themselves.
All write calls are bearer-authed and scoped to the authenticated creator — you
can only touch your own creations.

### Create a creation — `POST /api/v1/creations`

```bash
curl -X POST "$JBA_API_BASE/creations" \
  -H "Authorization: Bearer $JBA_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "name": "my cool thing",
    "tagline": "does one cool thing well",
    "description": "# what it is\n\nmarkdown ok, <= 2000 chars",
    "project_url": "https://example.com",
    "repo_url": "https://github.com/me/cool",
    "visibility": "public"
  }'
```

Body fields:
- `name` — required, ≤80 chars.
- `slug` — optional. Auto-generated from `name` when omitted (lowercase,
  spaces/underscores→`-`, non `[a-z0-9-]` stripped, repeats collapsed, ≤60,
  unique per creator). **Immutable after creation.**
- `tagline` — optional, ≤140 chars, plain text.
- `description` — optional, markdown, ≤2000 chars.
- `project_url`, `repo_url` — optional, must be valid `https://` URLs.
- `visibility` — `public` (default) or `draft`. Drafts are hidden from your
  public profile and the feed until flipped to `public`; still reachable by
  direct `GET /creations/{id}` link.

Responses:
- `201` → `{id, slug, name, tagline, project_url, repo_url, visibility, created_at}`.
- `409` slug taken → `{error:"slug_taken", suggestion:"<slug>-2", existing:{id,slug}}`.
  Retry with `suggestion` as the `slug` and it will succeed.
- `401` missing/invalid token. `422` validation → `{error:"validation", fields:{...}}`.

**Idempotency:** pass `Idempotency-Key: <uuid>`. Re-sending the same key within
24h returns the original `201` body without creating a duplicate — safe to
retry on a dropped connection.

### Edit a creation — `PATCH /api/v1/creations/{id}`

Same fields as create **except `slug`** (immutable — attempts to change it are
ignored). Send only the fields you want to change. `200` returns the updated
creation. `403` if you don't own it, `404` if unknown.

### Read a creation — `GET /api/v1/creations/{id}`

Public, no auth. Returns the creation object plus recent updates, paginated:
`?limit=10&cursor=<last-update-id>`. The response includes `next_cursor` (null
when there are no more updates).

### Rate limits

Per creator: **60 writes/hour** and **600 reads/hour**. Over the limit →
`429` with a `Retry-After` header. All error bodies are `{error, message, ...}`.
