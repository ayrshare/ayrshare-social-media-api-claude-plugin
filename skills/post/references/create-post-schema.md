# `mcp__ayrshare__create_post` / `mcp__ayrshare__validate_post` — full input schema

`POST /post` (create) and `POST /validate/post` (validate). `validate_post` takes the **same** inputs as `create_post` and reports platform-specific issues (character limits, unsupported media, duplicate content) as a dry-run — it does NOT publish. Always validate before creating.

Only `post` and `platforms` are required; everything else is optional. There is **no generic `passthrough`/escape-hatch field** — set advanced options directly as the typed fields below. Unknown keys are dropped on validation.

## Required

| Field | Type | Description |
|-------|------|-------------|
| `post` | string | The text content of the post (min length 1). |
| `platforms` | array of enum | Networks to publish to. Enum: `twitter, facebook, instagram, linkedin, tiktok, youtube, pinterest, reddit, telegram, gmb, bluesky, snapchat, threads`. Always an array. Include each platform's required fields (e.g. `youTubeOptions.title` for YouTube). See `platforms.md`. |

## Media

| Field | Type | Description |
|-------|------|-------------|
| `mediaUrls` | array of string (https) | Image/video URLs to attach. URLs must be secure (`https://`). Videos require a paid plan. Validate reachability first (see `../../media/SKILL.md`). |
| `isVideo` | boolean | Force the media to be treated as video when the URL has no known video extension (e.g. animated GIFs). Default `false`. |
| `isLandscapeVideo` | boolean | Test publishing only: with `isVideo` + `randomMediaUrl`, send a landscape random video. Default `false`. |
| `isPortraitVideo` | boolean | Test publishing only: with `isVideo` + `randomMediaUrl`, send a portrait random video (TikTok / Reels shape). Default `false`. |

## Scheduling

| Field | Type | Description |
|-------|------|-------------|
| `scheduleDate` | string (ISO 8601) | UTC datetime to schedule a future post, e.g. `2026-07-08T12:30:00Z`. Omit to publish immediately. |
| `validateScheduled` | boolean | Pre-validate scheduled posts at submission so errors surface immediately. Default `true`; set `false` to defer validation to publish time. |
| `autoSchedule` | object | `{ schedule: true (required), title? }` — publish into a named auto-schedule's next open slot. Cannot be combined with `scheduleDate`. |
| `autoRepost` | object | `{ repeat: 1-10 (required), days: >=2 (required), startDate? }` — repost on a recurring interval (evergreen). Paid plan. Cannot be combined with `scheduleDate`. |

## Content enhancement

| Field | Type | Description |
|-------|------|-------------|
| `firstComment` | object | `{ comment (required), mediaUrls? }` — automatically add a first comment after publishing (comment media on FB/LinkedIn/X only). |
| `disableComments` | boolean | Disable comments on the post. Instagram, LinkedIn, and TikTok only. Default `false`. |
| `shortenLinks` | boolean | Shorten `https` links via Ayrshare's shortener. Max Pack required. Default `false`. |
| `autoHashtag` | object \| `true` | `{ max?: 1-10, position?: "auto"\|"end" }` or the Boolean `true` for defaults. Auto-adds relevant hashtags. Paid plan. |
| `unsplash` | string \| array | `"random"`, a search-term string, or an array of Unsplash image IDs. |

## Workflow / metadata

| Field | Type | Description |
|-------|------|-------------|
| `requiresApproval` | boolean | Hold the post in `awaiting approval` until approved (via `update_post` with `approved: true`). |
| `notes` | string | Free-form notes attached to the post, retrievable via `get_post_history`. Does not affect the post. |
| `idempotencyKey` | string | Unique ID so retries do not create duplicates. Duplicate keys are rejected. |

## Test publishing

| Field | Type | Description |
|-------|------|-------------|
| `randomPost` | boolean | Generate random post text for testing. When `true`, `post` is ignored. |
| `randomMediaUrl` | boolean | Generate a random media image for testing. When `true`, `mediaUrls` is ignored. |

## Top-level per-platform fallbacks

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Post/video title fallback for platforms that require one (Reddit, Snapchat Saved Stories). Prefer the platform-options object (e.g. `youTubeOptions.title`, `redditOptions.title`). |
| `subreddit` | string | Subreddit (no leading `/r/`), top-level fallback for Reddit. Prefer `redditOptions.subreddit`. |

## Per-platform options (12 objects; Telegram has none)

Each is an optional object applied only when its platform is in `platforms`: `instagramOptions`, `facebookOptions`, `xOptions`, `youTubeOptions` (**`title` required for YouTube**), `tikTokOptions`, `linkedInOptions`, `pinterestOptions`, `redditOptions` (**`title` + `subreddit` required for Reddit**), `threadsOptions`, `snapChatOptions`, `gmbOptions`, `blueSkyOptions`. Each carries the network's specific fields (Reels/Stories, alt text, polls, visibility, targeting, thumbnails, etc.).

`create_post` is synchronous and returns per-platform `postIds`. To recover a failed send, use `retry_post` — NOT a second `create_post`.

## Profile scoping (not a post-body field)

The profile is selected by the optional `profileKey` tool argument or the `Profile-Key` connection header (with the plugin, its Profile-Key setting, entered with `/plugin configure ayrshare@ayrshare` and loaded with `/reload-plugins`); the argument wins when both are set. With neither, calls act under the account's primary/Business profile. See `../../getting-started/SKILL.md`.

## Lifecycle tools (key off the returned `id`)

| Tool | Endpoint | Inputs |
|------|----------|--------|
| `mcp__ayrshare__get_post` | GET `/post/:id` | `id` (required) — the ONLY input |
| `mcp__ayrshare__update_post` | PATCH `/post` | `id` (required) + at least one of: `scheduleDate`, `approved` (bool), `notes` (string), `disableComments` (bool; Instagram/LinkedIn only on update, and disabling on LinkedIn deletes existing comments), `scheduledPause` (bool), `youTubeOptions` (object; reuses the create-side YouTube schema, so `title` is required — re-send the existing title even when changing only visibility/description/categoryId). Primarily for editing/approving/rescheduling a PENDING post. |
| `mcp__ayrshare__retry_post` | PUT `/post/retry` | `id` (required, must be a post in status `error`). Retries ONCE, only if the error was retryable. |

## Reminders

- Validate (`mcp__ayrshare__validate_post`) → confirm with the user → create (`mcp__ayrshare__create_post`). On per-platform failure, call `mcp__ayrshare__explain_error`.
- `create_post` returns an `id`. Capture it — every other tool here needs it.
- `scheduleDate` omitted = post now. Never pass an empty string.
- `update_post` requires at least one editable field alongside `id`; it is primarily for PENDING posts.
- `retry_post` only applies to posts in status `error`, retries once, and only if retryable.
