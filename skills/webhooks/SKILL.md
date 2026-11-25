---
name: webhooks
description: |
  Webhook subscriptions for the Ayrshare MCP server: register, unregister, and list HTTPS push notifications so an agent or app is notified on events instead of polling. Use whenever someone wants event push: "notify me when a post publishes", "let me know when a scheduled post goes out", "set up a webhook for new comments", "send mentions to my endpoint", "list our webhooks", "what webhooks are registered", "stop sending webhooks for comments", "unsubscribe from the messages webhook". Trigger when calling `mcp__ayrshare__register_webhook`, `mcp__ayrshare__unregister_webhook`, or `mcp__ayrshare__list_webhooks`, and even without "Ayrshare": if the user wants push notifications for social-post or social-account events through an assistant, this is the skill. To confirm a queued post actually published, prefer the `scheduled` webhook over polling (then optionally `get_post` / `get_post_history`).
---

# Ayrshare MCP — Webhooks

Three tools for subscribing your endpoint to Ayrshare events so you receive **push notifications instead of polling**:

- `mcp__ayrshare__register_webhook` — subscribe an HTTPS URL to one event action.
- `mcp__ayrshare__unregister_webhook` — cancel a subscription by action.
- `mcp__ayrshare__list_webhooks` — list current subscriptions.

All three are profile-scoped: choose the profile with the `profileKey` argument or the `Profile-Key` header (see Auth; the argument wins when both are set).

## Functions

| Tool | Purpose | Method + Endpoint | Required inputs | Optional inputs |
|------|---------|-------------------|-----------------|-----------------|
| `mcp__ayrshare__register_webhook` | Subscribe an HTTPS endpoint to a webhook action | `POST /hook` | `url` (**must be `https://`**), `action` (one webhook action) | `source` (string), `secret` (string — for signature verification) |
| `mcp__ayrshare__unregister_webhook` | Cancel a webhook subscription | `DELETE /hook` | `action` (one webhook action) | `source` |
| `mcp__ayrshare__list_webhooks` | List current webhook subscriptions | `GET /hook` | — | `allWebhooks` (bool), `source` |

**Webhook actions** (the `action` enum): `feed`, `social`, `scheduled`, `batch`, `messages`, `mentions`, `comments`, `automations`, `accountActivity`.

## Auth

All three tools are **profile-scoped**: choose the profile with an optional `profileKey` tool argument or the `Profile-Key` connection header (the argument wins when both are set). Pass `profileKey` on the call to manage one client profile's webhooks, or set the plugin's Profile-Key setting (`/plugin configure`; it is sent as the `Profile-Key` header) to default the whole connection to it; with neither, calls manage the account's primary/Business profile. Beyond `profileKey`, these webhook tools take no `passthrough` object; their inputs are exactly the fields in the table above. Full model: `../getting-started/SKILL.md`.

## Usage guidance

- **`url` must be HTTPS.** `register_webhook` rejects non-`https://` URLs. The endpoint must be reachable so Ayrshare can deliver events to it.
- **Pick the right `action`.** One subscription = one action. The actions are `feed`, `social`, `scheduled`, `batch`, `messages`, `mentions`, `comments`, `automations`, `accountActivity`.
- **Confirming a queued post published — use `scheduled`, don't poll.** `scheduled` is the **platform-agnostic post-completion webhook**: it fires when a queued/scheduled post finishes processing. Register a `scheduled` webhook to be told when the post goes out, instead of repeatedly calling history. Once notified (or if you can't run an endpoint), you can confirm details with `mcp__ayrshare__get_post` (Ayrshare Post ID) or `mcp__ayrshare__get_post_history` — but the webhook is the push-based, no-polling path. See `../history/SKILL.md`.
- **Use `secret` to verify deliveries.** Pass a `secret` at registration so your endpoint can verify the signature on incoming payloads and reject spoofed calls.
- **`source` is a free-form tag** you can attach at register time and then filter on with `unregister_webhook` / `list_webhooks`. Use it to distinguish multiple subscriptions for the same action or different downstream consumers.
- **Unregister by `action`.** To stop notifications, call `unregister_webhook` with the same `action` (and `source` if you registered with one). To audit what's active, call `list_webhooks` (`allWebhooks: true` to widen the listing).

## Gotchas

- **Non-HTTPS URLs are rejected.** `register_webhook` requires `https://`. A plain `http://` or hostless URL fails.
- **One action per call.** `action` is a single value, not an array. To subscribe to multiple event types, register each separately.
- **`scheduled` is the post-completion signal, not `social`.** For "tell me when my queued post published," use `scheduled` (platform-agnostic, fires on post completion). Don't reach for polling loops when a `scheduled` webhook does it push-style.
- **Be deliberate about scope.** Which profile a webhook is registered/listed under is set by the `profileKey` argument (per call; it wins) or the connection's `Profile-Key` header; with neither it's the primary profile. Confirm you're targeting the intended profile before registering.
- **Unregister must match how you registered.** Cancel with the same `action` (and `source` if used). A mismatched `source` may leave a subscription active — verify with `list_webhooks`.
- **On failure, call `mcp__ayrshare__explain_error`, then surface it — don't loop.** `register_webhook`/`unregister_webhook` are writes: never auto-retry on a 4xx (bad URL, unknown action, wrong scope). Translate the error via `mcp__ayrshare__explain_error` and present it; 429 gets at most one retry. (Mirrors the global retry-safety rule.)
