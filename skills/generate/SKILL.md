---
name: generate
description: |
  AI content generation for the Ayrshare MCP server — draft post copy and suggest hashtags. Use whenever someone wants help WRITING social content rather than publishing it: "write me a post about our product launch", "draft a tweet announcing the sale", "generate a caption for this image", "give me a LinkedIn post about X", "what should I caption these photos", "suggest hashtags for sustainable fashion", "recommend trending hashtags for ...". Trigger when calling `mcp__ayrshare__generate_post` or `mcp__ayrshare__recommend_hashtags`, and even without the word "Ayrshare" — if the user wants AI-written social copy or hashtag ideas through an assistant, this is the skill. These tools produce DRAFTS ONLY; they never publish. The natural next step is `mcp__ayrshare__validate_post` then `mcp__ayrshare__create_post` (see `../post/SKILL.md`). For the shared auth model and retry rules, see `../getting-started/SKILL.md`.
---

# Ayrshare MCP — Generate (AI)

Two tools that produce content for you to review and publish — neither one posts anything:

- `mcp__ayrshare__generate_post` — AI-drafts post copy from a topic/brief and/or media. **Draft only.**
- `mcp__ayrshare__recommend_hashtags` — returns recommended hashtags for a keyword, sourced from a linked TikTok account.

Unlike most Ayrshare MCP tools, neither of these takes a per-call `profileKey` argument (see Auth). `recommend_hashtags` is still scoped by the connection's `Profile-Key` header; it draws on that profile's linked TikTok account.

## Functions

| Tool | Purpose | Method + Endpoint | Required inputs | Optional inputs |
|------|---------|-------------------|-----------------|-----------------|
| `mcp__ayrshare__generate_post` | AI-draft post copy from a topic/prompt and/or media URLs; returns text only, does NOT publish | `POST /generate/post` | At least one of `text` (topic/brief/prompt string) **or** `mediaUrls` (array of public URLs to caption) | `hashtags` (bool, default `true`), `emojis` (bool, default `false`), `twitter` (bool, default `false` — constrain to X/Twitter length) |
| `mcp__ayrshare__recommend_hashtags` | Recommend hashtags for a keyword (sourced from a linked TikTok account) | `GET /hashtags/recommend` | `keyword` (single keyword string; spaces are stripped before lookup, so pass one word, not a phrase/topic) | `source` (free-form attribution tag) |

## Auth

These two tools are exceptions to the per-call `profileKey` argument that the profile-scoped tools accept: **`generate_post` and `recommend_hashtags` do NOT take a `profileKey` argument.** `recommend_hashtags` is still scoped by the connection's `Profile-Key` header (with the plugin, its Profile-Key setting): it uses that profile's linked TikTok account; omit the header to use the account's primary/Business profile. `generate_post` is account-level AI drafting. See *Which tools accept `profileKey`* in `../getting-started/SKILL.md`.

## Usage guidance

- **`generate_post` produces a DRAFT, not a published post.** It returns suggested copy. To actually publish, hand the result to the posting pipeline: `mcp__ayrshare__validate_post` (dry-run platform-rule check) → `mcp__ayrshare__create_post`. See `../post/SKILL.md`. Never treat a `generate_post` response as "posted."
- **Provide `text`, `mediaUrls`, or both — at least one is required.** Pass `text` to brief the model with a topic, angle, or prompt ("announce our Series B, upbeat tone"). Pass `mediaUrls` (an array of public image/video URLs) to have it caption that media. Provide both to caption media on-topic. With neither, there is nothing to generate from.
- **Tune the draft with the optional flags.** `hashtags` defaults to `true` (set `false` to suppress them); `emojis` adds emoji; `twitter: true` constrains the draft to X/Twitter length when the copy is destined for X. These shape the draft text only.
- **`recommend_hashtags` needs a linked TikTok account.** Recommendations are sourced from TikTok, so the API errors if the profile (per the connection's `Profile-Key`) has no TikTok account linked. If you get an error, confirm a TikTok account is connected for that profile before retrying. Pass `keyword` as a single keyword: spaces are stripped before lookup, so use one word (e.g. `fitness`), not a multi-word phrase or full topic, or you'll get poor or empty results. `source` is just a free-form attribution tag.
- **Pair the two.** A natural flow is `recommend_hashtags` for a keyword, then `generate_post` (with `hashtags: false` if you want to append the recommended set yourself), then validate + create.

## Gotchas

- **Neither tool publishes anything.** `generate_post` and `recommend_hashtags` only return text/suggestions. Publishing is a separate, explicit step via `create_post`. Don't tell the user a post went out off the back of these tools.
- **`generate_post` requires `text` OR `mediaUrls`.** Calling it with neither has nothing to work from. If captioning media, the URLs must be publicly reachable (consider `mcp__ayrshare__validate_media` first).
- **`recommend_hashtags` is TikTok-sourced.** No TikTok account linked on the active profile → the call errors. This is expected, not a bug; link TikTok or scope the connection to a profile that has one.
- **`recommend_hashtags` takes a single keyword, not a phrase.** Spaces in `keyword` are removed before lookup (`sustainable fashion` becomes `sustainablefashion`), which produces poor or empty results. Pass one representative keyword instead of a multi-word topic.
- **`recommend_hashtags` uses the connection's `Profile-Key`, and takes no `profileKey` argument.** Which profile's linked TikTok account it draws on is set by the `Profile-Key` header. Unlike the profile-scoped tools, neither generate tool accepts the per-call `profileKey` argument. (See *Which tools accept `profileKey`* in getting-started.)
- **On failure, call `mcp__ayrshare__explain_error`, then surface it — don't loop.** A 4xx (missing `text`/`mediaUrls`, no linked TikTok, unreachable media) won't fix itself on retry. Translate the error via `mcp__ayrshare__explain_error` and present it; 429 gets at most one retry. (Mirrors the global retry-safety rule.)
