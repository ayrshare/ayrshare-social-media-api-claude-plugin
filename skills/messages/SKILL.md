---
name: messages
description: |
  Direct messages (DMs) for the Ayrshare MCP server: read conversations and messages, send a DM, and manage the account's DM auto-responder. Use whenever someone wants private/direct messages on Facebook, Instagram, or Twitter/X: "read my Instagram DMs", "show our Facebook conversations", "what messages came in on X", "reply to that DM", "send a direct message", "DM them this link/photo", "answer the latest message". Also for auto-reply settings: "set up an auto-reply for DMs", "turn on the DM auto-responder", "change the auto-reply message", "what is our auto-responder set to", "turn off auto-replies". Trigger when calling `mcp__ayrshare__get_messages`, `mcp__ayrshare__send_message`, `mcp__ayrshare__get_auto_response`, or `mcp__ayrshare__set_auto_response`, and even without "Ayrshare": if the user wants to read or send social DMs, or configure a DM auto-reply, through an AI assistant, this is the skill. DMs exist ONLY on Facebook, Instagram, and Twitter/X.
---

# Ayrshare MCP — Messages (DMs)

Four tools, split into two jobs — moving messages, and the account-level auto-responder:

- `mcp__ayrshare__get_messages` — read conversations and messages for one platform. Its conversation/message ids feed `send_message`.
- `mcp__ayrshare__send_message` — send a DM (text and/or media) to a recipient or into an existing conversation.
- `mcp__ayrshare__get_auto_response` — read the account's DM auto-reply settings.
- `mcp__ayrshare__set_auto_response` — change the account's DM auto-reply settings.

**DMs are ONLY on Facebook, Instagram, and Twitter/X.** No other network (LinkedIn, TikTok, YouTube, etc.) has a messaging surface here. `get_messages` and `send_message` are platform-scoped and accept exactly one of `facebook`, `instagram`, `twitter`. The two auto-response tools take no `platform`: they govern the selected profile's DM auto-reply across FB/IG/X at once (network-agnostic, but still scoped to the profile the `profileKey` argument or `Profile-Key` header selects).

All four are profile-scoped: choose the profile with the `profileKey` argument or the `Profile-Key` header (see Auth; the argument wins when both are set).

## Functions

| Tool | Purpose | Method + Endpoint | Required inputs | Optional inputs |
|------|---------|-------------------|-----------------|-----------------|
| `mcp__ayrshare__get_messages` | Read conversations / messages for one platform | `GET /messages/:platform` | `platform` (`facebook`\|`instagram`\|`twitter`) | `conversationsOnly` (bool), `conversationId`, `status` (`active`\|`archived`), `limit` (1-100, **X/Twitter only**), `next` (cursor, **X only**) |
| `mcp__ayrshare__send_message` | Send a DM (text and/or media) | `POST /messages/:platform` | `platform` (`facebook`\|`instagram`\|`twitter`); **`message` OR `mediaUrls`**; **`recipientId` OR `conversationId`** | — (none beyond the above) |
| `mcp__ayrshare__get_auto_response` | Read the account's DM auto-reply settings | `GET /messages/autoresponse` | — (no params) | — |
| `mcp__ayrshare__set_auto_response` | Update the account's DM auto-reply settings | `POST /messages/autoresponse` | at least one of `autoResponseActive` (bool), `autoResponseMessage` (string), `autoResponseWaitSeconds` (int) | — (none beyond the above) |

Full input schemas and example payloads/responses are in [`references/schemas.md`](references/schemas.md).

## Auth

All four tools are **profile-scoped**: choose the profile with an optional `profileKey` tool argument or the `Profile-Key` connection header (the argument wins when both are set). Pass `profileKey` on the call to act as one client for that call, or set the plugin's Profile-Key setting (`/plugin configure ayrshare@ayrshare`, then `/reload-plugins`; it is sent as the `Profile-Key` header) to default the whole connection to it; with neither, calls act on the account's primary/Business profile. Full model: `../getting-started/SKILL.md`.

This applies to the auto-response tools too: they take no `platform`, but still act on whichever profile the `profileKey` argument or `Profile-Key` header selects. Different profiles → different DM auto-reply settings.

## Usage guidance

- **Pick the platform first.** `get_messages` and `send_message` are `:/platform` calls; pass exactly one of `facebook`, `instagram`, `twitter`. There is no "all platforms" mode — to scan three networks, make three calls.
- **Read before you reply.** Call `get_messages` to pull conversations/messages, find the right conversation, then reply with `send_message`. Use `conversationsOnly: true` for a lightweight list of threads (no message bodies) when you just need to pick a conversation; drop it (or pass a `conversationId`) to read the messages within one thread.
- **`send_message` needs two things: content AND a target.**
  - **Content:** supply `message` (text), `mediaUrls` (an array of URLs), or both. At least one is required — an empty message with no media is rejected. **On Twitter/X, `message` text is required even when you send `mediaUrls`** — a media-only DM is rejected on X (FB/IG accept media-only). `mediaUrls` count also differs: **Facebook and Instagram accept multiple URLs; X/Twitter accepts only one** — sending more than one to X errors.
  - **Target:** supply `recipientId` (the person/page-scoped user id) **or** `conversationId` (an existing thread). At least one is required.
- **Facebook and Instagram REQUIRE `recipientId`.** On FB/IG you must address the message to a `recipientId`; a bare `conversationId` is not enough. On Twitter/X you can reply into a `conversationId` or send to a `recipientId`. When in doubt, include `recipientId` — it's the universally accepted target.
- **Where ids come from.** `recipientId` and `conversationId` come out of `get_messages`. Read first to obtain them; don't guess or hand-construct ids.
- **Messaging windows are real.** Each platform enforces a window in which you may DM a user; on Facebook/Instagram this follows Meta's standard messaging-window policy (tied to the user's last message to you). Outside the window the API returns a clear error. **Don't retry it** — translate it via `mcp__ayrshare__explain_error` and tell the user the window has closed (the customer must message first to reopen it). See the global retry rule in `../getting-started/SKILL.md`.
- **`limit` and `next` are Twitter/X-only.** `limit` (1-100) caps results and `next` is the pagination cursor — both apply to X only on `get_messages`. Don't pass them for Facebook or Instagram.
- **Auto-responder is account-level.** `get_auto_response` takes no params and returns `autoResponseActive`, `autoResponseMessage`, `autoResponseWaitSeconds`. `set_auto_response` accepts any subset of those three; send the field(s) you're changing (at least one of the three is required). Sending `autoResponseMessage: ""` (empty string) **resets the message to the Ayrshare default**, it does not blank it out.

## Gotchas

- **DMs are only FB / IG / X.** `get_messages` and `send_message` reject any other platform. Don't try LinkedIn/TikTok/YouTube/etc. DMs through these tools — there is no messaging surface for them.
- **Be deliberate about scope.** Whose DMs you read/send, and whose auto-responder you read/set, is set by the `profileKey` argument (per call; it wins) or the connection's `Profile-Key` header; with neither it's the primary profile. Confirm you're targeting the right profile before reading or sending.
- **`send_message` needs content AND target.** Missing either side fails: you must give `message` OR `mediaUrls`, **and** `recipientId` OR `conversationId`. FB/IG additionally require `recipientId` specifically — a `conversationId` alone won't do. **On Twitter/X, `message` text is required even with `mediaUrls`** — a media-only DM (the generic "`mediaUrls` alone is fine" case) is rejected on X; it works only on FB/IG.
- **`mediaUrls` is an array of URLs.** Media is referenced by reachable URL, same as posts — there is no upload step here. Validate a media URL with `mcp__ayrshare__validate_media` (see the Media skill) before sending if you're unsure it's reachable.
- **Auto-response empty string resets, not clears.** `autoResponseMessage: ""` reverts to the Ayrshare default reply; it does not produce an empty auto-reply. To stop auto-replying entirely, set `autoResponseActive: false`.
- **Auto-response is not platform-scoped.** `get_auto_response`/`set_auto_response` hit `/messages/autoresponse` with no `platform`; they govern the whole account's (profile's) DM auto-reply across FB/IG/X at once. Don't expect a per-platform toggle here.
- **On failure, call `mcp__ayrshare__explain_error`, then surface it — don't loop.** `send_message` is a write: never auto-retry it on a 4xx. A closed messaging window, a bad/expired `recipientId`, or an unsupported platform won't fix itself on retry, and a blind resend can double-send. Translate the error via `mcp__ayrshare__explain_error` and present it; 429 gets at most one retry. (Mirrors the global retry-safety rule.)
