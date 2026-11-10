---
name: getting-started
description: |
  Foundation skill for the Ayrshare MCP server — auth model, installation, client onboarding, and retry safety. Use this whenever someone is installing, configuring, or wiring up the Ayrshare MCP plugin/server in Claude Code; whenever an Ayrshare MCP tool call returns 401/403 or "unauthorized"; whenever there's confusion about WHICH credential to use ("which key do I use", "how do I act as a specific client", Business key vs Profile-Key); when onboarding a new client/customer/profile; or when a user has no Ayrshare account, no API key, or an API key that is not set up yet. Trigger even if the user doesn't say "Ayrshare" by name — if they're calling tools named `mcp__ayrshare__*`, setting up an MCP server for social posting, or asking how to connect a client's social accounts through an AI assistant, this is the skill. Every other Ayrshare MCP skill (posts, comments, analytics, history, messages, media, profiles, generate, webhooks, errors) cross-links here for the shared auth and retry rules.
---

# Ayrshare MCP — Getting Started

The single home for the cross-cutting model behind the Ayrshare MCP server: the **auth model** (API key connection header, plus optional profile selection by either a `Profile-Key` connection header or a per-call `profileKey` tool argument), the **client onboarding sequence**, and the **retry-safety rules**. Every other Ayrshare MCP skill points here instead of repeating it. If you only read one Ayrshare MCP skill, read this one. Most failures are auth-layer mistakes, not tool bugs.

- Transport: **HTTP MCP server** at `https://api.ayrshare.com/mcp`, authenticated by the header `Authorization: Bearer <API key>`, filled from the plugin's `api_key` setting.
- Tools are named `mcp__ayrshare__<action>` (e.g. `mcp__ayrshare__create_post`).
- Full install instructions (Claude Code plugin, MCP-only, CI/headless) live in `references/install.md`. The summary below is enough to get running.

## Authentication — the #1 failure mode

Auth has two parts. The **API key** is always a per-connection header configured in the MCP client (the plugin's `.mcp.json`), never a tool argument. The **target profile** is optional and can be set two equivalent ways, either a `Profile-Key` connection header (applies to every call on the connection) or a per-call `profileKey` tool argument (selects the profile for that one call):

**`Authorization: Bearer <API key>` (required).** Your account-level Ayrshare Business API key. The MCP server sends it on every call as the HTTP Bearer token. It is entered once in the plugin's settings (`/plugin configure ayrshare@ayrshare`; `/ayrshare:setup` explains it), never passed as a per-call argument. The multi-profile features (Profiles) require a **Business plan**.

**Target profile (optional): `Profile-Key` header *or* `profileKey` argument.** Selects which client/customer profile a call acts on; a `profileKey` is returned by `mcp__ayrshare__create_profile`. Set it either as the `Profile-Key` connection header (the default for every call on that connection) or as an optional `profileKey` argument on the tool call (just that call). **When both are present, the `profileKey` argument wins**; with neither, calls act on the account's **primary** profile. The per-call argument lets an agent act as a client it learns at runtime (from chat, a database, or a CRM) without editing `.mcp.json` or restarting. It is accepted by the profile-scoped tools only; the account-level and no-social-account tools ignore it (see *Which tools accept `profileKey`* below). (`generate_jwt_social_linking_url` uses the same value to pick the sub-profile it mints a linking URL for, and **requires** it via either input.)

**The rule in one line:** the API key sets the account; the profile selection sets the identity. With only the API key set, every call acts on the primary/Business profile. To act as a client profile, either set that profile's `Profile-Key` header on the connection (applies to all calls) or pass its `profileKey` as a tool argument (per call, and it wins over the header). `list_profiles` and `create_profile` are account-level; the profile-scoped tools, including `generate_jwt_social_linking_url`, act on whichever profile the argument-or-header selects.

### Which tools accept the `profileKey` argument

Most tools are **profile-scoped** and accept the optional `profileKey` argument (equivalently, the `Profile-Key` header): the **post**, **history**, **analytics**, **comments**, **messages**, and **webhook** tools, plus `generate_jwt_social_linking_url`. Six tools do **not** take it:

- **Account-level** (operate on the Business account, not a sub-profile): `create_profile`, `list_profiles`.
- **Utility / AI tools** (no per-call sub-profile selection): `validate_media`, `explain_error`, `generate_post`, `recommend_hashtags`. (`recommend_hashtags` still reads the `Profile-Key` header to pick whose linked TikTok account it uses; it just does not accept the per-call argument.)

**One exception:** on `get_platform_history` and `get_social_network_analytics`, a `userId`/`userName` lookup (a specific X/Twitter user, not your linked account) must use the **API key only**. Supplying a `profileKey` (argument *or* `Profile-Key` header) together with `userId`/`userName` returns **Error 400**.

### Setting the profile: connection header vs per-call argument

**As a connection header (the default for every call).** With the **plugin**, the bundled `.mcp.json` already carries the header, filled from the plugin's optional `profile_key` setting (empty by default):
```jsonc
"headers": {
  "Authorization": "Bearer ${user_config.api_key}",
  "Profile-Key": "${user_config.profile_key}"   // empty by default → primary profile
}
```
So the user just fills in **Profile-Key** with `/plugin configure ayrshare@ayrshare`; every call on that connection then defaults to that profile. Leave it empty and the header goes out empty, which the server treats as "act on the primary profile." (For a non-plugin `claude mcp add` setup, add the `Profile-Key` header yourself alongside `Authorization`.)

**As a per-call argument (no restart).** Leave the header unset (or keep one as the default) and pass `profileKey` directly on a profile-scoped tool call to act as a specific client for that single call. The argument takes precedence over the header when both are set, so an agent can act as a client it discovers at runtime without touching `.mcp.json` or restarting:
```jsonc
// e.g. get this client's recent posts without changing the connection
{ "limit": 25, "profileKey": "<the client's profileKey>" }
```

**Security note: the header keeps the key out of band; the argument does not.** A `Profile-Key` connection header is sent by the transport and never enters the model context or the conversation transcript. A `profileKey` argument is, by design, part of the tool call the agent emits, so it appears in the conversation transcript and the payload sent to the model provider. Prefer the header for static, single-profile setups; use the per-call argument deliberately when you need runtime selection. The exposure is bounded: a `profileKey` is inert without the account API key (which is never an argument), so a key seen in a transcript is not usable on its own.

## The tool surface (27 tools, by domain)

Each domain has its own skill with full parameters and gotchas:

- **Posts** (`../post/SKILL.md`): `create_post`, `validate_post`, `get_post`, `update_post`, `retry_post`.
- **History** (`../history/SKILL.md`): `get_post_history`, `get_platform_history`.
- **Analytics** (`../analytics/SKILL.md`): `get_post_analytics`, `get_post_analytics_by_social_id`, `get_social_network_analytics`.
- **Comments** (`../comments/SKILL.md`): `get_comments`, `add_comment`, `reply_comment`.
- **Messages / DMs** (`../messages/SKILL.md`): `get_messages`, `send_message`, `get_auto_response`, `set_auto_response`.
- **Profiles** (`../profiles/SKILL.md`): `list_profiles`, `create_profile`, `generate_jwt_social_linking_url`.
- **Media validation** (`../media/SKILL.md`): `validate_media`.
- **Generate** (`../generate/SKILL.md`): `generate_post`, `recommend_hashtags`.
- **Webhooks** (`../webhooks/SKILL.md`): `register_webhook`, `unregister_webhook`, `list_webhooks`.
- **Errors** (`../errors/SKILL.md`): `explain_error`.

(There is no `get_user`, `delete_post`, `delete_comment`, `delete_profile`, or media upload/library/resize tool — if you reach for one of those, it does not exist.)

## API key requirement

The API key (the plugin's `api_key` setting) is the **account-level Business plan API key**, obtained from the Ayrshare dashboard ([app.ayrshare.com](https://app.ayrshare.com) → Settings → API Key). The MCP server sends it as the HTTP Bearer token on every call; it is not a per-call argument. A **Business plan is required** for the Profiles / multi-profile features (create profile, act as a client). If the key is missing, Claude Code does not load the server, so no Ayrshare tools appear; if it is wrong, every tool returns 401/403. See *Missing credentials* below.

## Installation (summary)

Full, copy-paste instructions are in `references/install.md`. The essentials:

**Claude Code plugin (recommended)** — register the marketplace once, install (this brings commands, agents, and skills), then enter the key in the plugin's settings:
```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin   # one time
claude plugin install ayrshare@ayrshare                  # default scope: user (all projects)
# or scope it:
claude plugin install ayrshare@ayrshare --scope local    # this project, not committed
claude plugin install ayrshare@ayrshare --scope project  # this project, committed with team
```
Then inside Claude Code, open the plugin's settings dialog and enter the key:
```text
/plugin configure ayrshare@ayrshare
```
If the user added Ayrshare from Claude's plugin directory on claude.ai, the plugin's id is `ayrshare@synced`, so the command is `/plugin configure ayrshare@synced` (needs Claude Code 2.1.273 or later; `claude plugin list` shows the id; the dialog is also under `/plugin` → **Installed** → **ayrshare** → **Configure options**, and in VS Code under `/plugins` → the gear icon on the **ayrshare** row). The value is masked and stored in the system's secure credential store. Never ask the user to paste the key into the chat. `claude plugin install` from a shell does not show the dialog, so it has to be opened from a session. `/ayrshare:setup` walks the user through the fields.

**MCP only (no plugin)** — raw tools, no commands/agents/skills:
```bash
claude mcp add ayrshare --transport http \
  https://api.ayrshare.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

**CI / headless:** install the plugin, then run `claude plugin configure ayrshare@ayrshare --values-stdin < values.json` (Claude Code 2.1.285+), where the CI system writes `values.json` from its secret store, so the key never sits on a command line (details in `references/install.md`).

**Upgrading from 0.2.2 or earlier:** those versions read the `AYRSHARE_API_KEY` environment variable. This version does not. Until the user enters the key with `/plugin configure ayrshare@ayrshare`, Claude Code does not load the server, so no `mcp__ayrshare__*` tools appear at all. The plugin needs Claude Code 2.1.119 or later.

> **Verification gotcha:** saving the settings reloads the plugin and reconnects the server. If a tool still returns 401/403 right after, run `/reload-plugins` or restart Claude Code, then run `mcp__ayrshare__get_post_history` (a plan-agnostic read — `list_profiles` needs a Business plan) to confirm the connection works.

## Canonical client onboarding sequence

Onboarding a new client means creating a profile under your Business account, getting that client's social accounts linked, then acting as that profile.

1. **`mcp__ayrshare__create_profile`** (account-level, API key) — pass a `title`; returns the `profileKey`. Capture it (store it securely — it is sensitive).
2. **Mint the linking URL with `mcp__ayrshare__generate_jwt_social_linking_url`,** passing the `profileKey` from step 1 as the `profileKey` argument. No header change and no restart are needed; the argument selects the target profile for that call (the `Profile-Key` header still works as an alternative). It needs **no private key or domain** (the server derives those from your authenticated account) and returns a hosted linking `url` (valid 5 minutes by default, or `expiresIn` on the Max Pack). Hand the `url` to the client; they open it in a browser to OAuth their accounts. (The Ayrshare dashboard's linking page and the REST `/profiles/generateJWT` endpoint are equivalent alternatives.) Requires a provisioned social-linking domain (Business/Enterprise). Full schema in `../profiles/SKILL.md`.
3. **Act as the new profile** for ongoing posting/analytics/comments/etc.: pass its `profileKey` as the tool argument on each call, or fill in the plugin's Profile-Key setting with `/plugin configure ayrshare@ayrshare` to make that profile the default for the whole connection. `create_profile`/`list_profiles` ignore both and stay account-level.
4. **Verify:** `mcp__ayrshare__list_profiles` shows the profile and its linked platforms; `mcp__ayrshare__get_post_history` (with the `profileKey` argument or the `Profile-Key` header set) confirms you are acting on the right profile.

Don't over-railroad this. A client may already have a `profileKey`, in which case skip to step 2.

## Retry safety (global — referenced by every group skill)

The server fails fast and surfaces errors. Your job is to **not paper over them**.

- **When a tool call fails, call `mcp__ayrshare__explain_error`** with the returned error `code` to translate the raw API error into plain language, then surface that explanation to the user. Do **not** silently retry.
- **Never auto-retry a write on a 4xx.** A 4xx means bad input, wrong key/permission, or a platform rule — a retry won't fix it and a re-post can duplicate. Surface the error (via `explain_error`) instead. This applies to all writes: `create_post`, `update_post`, `retry_post`, `add_comment`, `reply_comment`, `send_message`, `set_auto_response`, `create_profile`, and the webhook writes. (Separately, the plugin's confirm-before-acting prompt covers the live-social publish/send actions plus `create_profile`; the webhook writes are still writes (no auto-retry on a 4xx), but are reversible config and are not behind that prompt.)
- **`retry_post` is the ONLY correct way to re-attempt a failed post** — and only when the original error says it is retryable, and only once. Never re-run `create_post` to "try again" (that double-posts on platforms that already succeeded).
- **429 (rate limit):** at most **one** retry, after a short delay. No tight loops, no unbounded backoff.
- **Pre-flight validation:** `mcp__ayrshare__validate_post` checks content for platform-specific issues *before* publishing, and `mcp__ayrshare__validate_media` checks a media URL is reachable. Use them ahead of `create_post` — detail in the Posts and Media skills.
- **Onboarding step failure:** if `mcp__ayrshare__create_profile` fails (wrong plan, invalid key, etc.) — **STOP and explain** (via `explain_error`) rather than pushing forward.

## Missing credentials / free-trial guidance

When the user has **no Ayrshare account or API key**, when **the API key is not set or wrong** (an unset key means no `mcp__ayrshare__*` tools load at all), or when the API returns **401/403**, surface this signup link for a **28-day free trial of the Ayrshare Launch plan**:

```text
https://billing.ayrshare.com/b/9B6bJ15Oidr9fz615u1Nu0h?utm_source=claude
```

The `?utm_source=claude` query parameter MUST be preserved exactly — it is signup attribution. Do not alter, shorten, URL-encode differently, or drop it. Note that the *Launch* trial does not include Profiles; the Profiles / multi-profile features still require a **Business** plan.

## Gotchas

- **Losing track of which profile a call targets.** The `profileKey` tool argument wins over the `Profile-Key` connection header; with neither set, calls act on the primary profile. To act as a client for one call, pass `profileKey` as the argument; to make a whole connection default to that client, set the header. The account-level tools (`create_profile`, `list_profiles`) and the no-social-account tools (`validate_media`, `explain_error`, `generate_post`, `recommend_hashtags`) accept neither. On `get_platform_history` / `get_social_network_analytics`, a `userId`/`userName` lookup must use the API key only (a `profileKey` there returns Error 400).
- **Reaching for a tool that doesn't exist.** There is no `get_user`, `delete_post`, `delete_comment`, `delete_profile`, or media upload/list/resize tool. Deleting profiles is done in the dashboard / REST API; linking a client's accounts is done by the client opening a `generate_jwt_social_linking_url` URL in their browser (the MCP mints the link but does not run the OAuth).
- **Verifying before the server reconnects.** The Bearer token is sent from the plugin's saved settings. If a call still returns 401/403 just after saving them, run `/reload-plugins` or restart Claude Code, then verify with `mcp__ayrshare__get_post_history` (works on any plan).
- **Auto-retrying a failed write.** Never retry a write on a 4xx; call `explain_error` and surface it. To re-attempt a failed post use `retry_post` (once, only if retryable), never a second `create_post`. 429 gets exactly one retry.
- **Assuming a non-Business plan works for Profiles.** Profiles and multi-profile onboarding require a Business plan. A Launch/free-trial key fails these even though the key is valid for single-account posting.
- **Dropping `utm_source=claude`.** The trial link's query param is attribution; keep it byte-for-byte.
- **Assuming you must restart to switch clients.** You don't. Pass the target client's `profileKey` as a tool argument and that single call acts as them, with no header change and no restart. Change the Profile-Key setting (with `/plugin configure ayrshare@ayrshare`) only when you want every call on the connection to default to one client.
