# Ayrshare for Claude Code

The unified social media API for AI agents. Publish, schedule, and analyze across 14+ networks directly from Claude Code, with post history for brand voice and validation against each platform's rules before every post. Powered by the [Ayrshare](https://www.ayrshare.com) API.

## Install

```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin
claude plugin install ayrshare@ayrshare
```

Then, inside Claude Code, run `/plugin configure ayrshare@ayrshare` and enter your API key. Claude Code stores it in your system's secure credential store. `/ayrshare:setup` walks you through it. Scopes and other install paths are under [Installation](#installation) below.

Then use the plugin by **asking in plain English** (e.g. "show my recent Instagram posts", "post this to LinkedIn tomorrow at 9am"); the tools fire on intent, and the trigger-skills route them. `/ayrshare:setup` is the only slash command, so there is no `/ayrshare:post` or `/ayrshare:get_post_history` to type.

## Overview

The plugin lets an agent run the full social workflow without leaving Claude Code. Before publishing, the agent validates each post against the target network's rules and asks you to confirm, so avoidable rejections are caught before anything goes live. A single request publishes to Facebook, Instagram, LinkedIn, X, TikTok, YouTube, Pinterest, Reddit, Telegram, Threads, Bluesky, and others. Post history is available for matching a brand's voice, and analytics can be read back to inform the next post. Platform integrations are maintained by Ayrshare, so upstream API changes are handled outside your code. The API currently handles 30M+ calls per day.

## The loop

- **Learn**. Retrieve a profile's post history for brand-voice context and review how past posts performed.
- **Act**. Publish and schedule across 13+ networks in a single call.
- **Observe**. Read live analytics to inform the next post.
- **Stay safe**. Validate each post against platform rules, then confirm before publishing.

---
## Why use the API instead of a custom integration?

An LLM can generate an OAuth flow, but it does not cover Meta Tech Provider approval, X's pay-per-use API changes, or ongoing maintenance when a network changes its rules. The plugin handles:

- **Pre-publish validation**. `validate_post` dry-runs content against each network's rules (length, format, media requirements) before you publish, and `validate_media` checks that a media URL is reachable before you attach it in `mediaUrls`. X BYOK is supported for the 2026 mandate, and platform integrations are maintained so an upstream API change does not break your integration.
- **One call, 13+ networks**. A single request publishes to all connected platforms, each validated against its own rules.
- **Multi-tenant**. Post on behalf of many user profiles rather than a single brand.
- **History API**. Retrieve a profile's past posts for brand-voice context, and use historical performance data to inform future posts.

### Safety

By default the plugin validates a post and requests confirmation before publishing. The agent prepares the draft; the user approves it.

A `PreToolUse` hook (`hooks/hooks.json`) also makes Claude Code ask you before it runs any tool with a live side effect. If you decline, the tool does not run and no request reaches Ayrshare. The hook covers these tools:

| Tool | Why it asks |
|---|---|
| `create_post`, `update_post`, `retry_post` | Publishes, schedules, or changes a post on real social accounts. |
| `add_comment`, `reply_comment` | Posts a comment in public. |
| `send_message` | Sends a direct message. |
| `set_auto_response` | Turns the DM auto-responder on or off, or changes its reply. |
| `create_profile` | Creates a client profile, which uses profile quota and cannot be deleted through MCP. |
| `update_ad` | Pauses, resumes, deletes, or archives a boosted ad, which changes real ad spend. The same status is also set on the ad's ad set and campaign, so other ads there change too. |
| `boost_post` | Boosts a published post as a paid ad, which spends real money from the ad account. |
| `register_webhook`, `unregister_webhook` | Changes where Ayrshare sends your account's events, including message and comment content. |

The hook matches each tool under both names Claude Code can give it: `mcp__plugin_ayrshare_ayrshare__<tool>` when the server comes from this plugin, and `mcp__ayrshare__<tool>` when you registered the server yourself under the name `ayrshare`.

Versions before 0.2.2 never showed this prompt for plugin installs, because the hook only listed the `mcp__ayrshare__<tool>` names. To update, run `claude plugin marketplace update ayrshare`, then `claude plugin update ayrshare@ayrshare`, and restart Claude Code.

---

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- An Ayrshare account with an API key — get one at [app.ayrshare.com](https://app.ayrshare.com) under **Settings → API Key** or get a [28 day free trial here](https://billing.ayrshare.com/b/9B6bJ15Oidr9fz615u1Nu0h?utm_source=github)

---

## Installation

### Plugin installation scopes

The plugin scope controls where commands, agents, and skills are available. First register this repo as a marketplace (one time):

```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin
```

Then install the `ayrshare` plugin (`plugin@marketplace`) at the scope you want:

| Scope | Command | Where it's available |
|---|---|---|
| Global (default) | `claude plugin install ayrshare@ayrshare` | Every project on this machine |
| This project only | `claude plugin install ayrshare@ayrshare --scope local` | Current project, not committed to git |
| This project (shared) | `claude plugin install ayrshare@ayrshare --scope project` | Current project, committed to git with the team |

After installing, configure your API key inside Claude Code:

```text
/plugin configure ayrshare@ayrshare
```

The dialog asks for your **Ayrshare API key** (required) and three optional settings: a **Profile-Key** and your **X API Key** and **X API Secret** (see [Optional Configuration](#optional-configuration)). Each value is masked as you type it and kept in your system's secure credential store, not in a settings file or the chat. `claude plugin install` run from a shell does not show this dialog, so open it from a session afterwards. `/ayrshare:setup` explains each field.

#### Upgrading from 0.2.2 or earlier

Earlier versions read your key from the `AYRSHARE_API_KEY` environment variable (and the optional `AYRSHARE_PROFILE_KEY`, `X_TWITTER_OAUTH1_API_KEY` and `X_TWITTER_OAUTH1_API_SECRET`). Version 0.2.3 no longer reads them. After updating, run `/plugin configure ayrshare@ayrshare` once and enter the same values. You can then delete the old variables from the `env` block of your `settings.json` (or your shell profile).

---

### Detailed walkthroughs

#### Global: available in all your projects

```bash
# 1. Register the marketplace (one time)
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin

# 2. Install the plugin globally (user is the default scope)
claude plugin install ayrshare@ayrshare

# 3. Enter your API key. Run inside Claude Code:
/plugin configure ayrshare@ayrshare
```

Commands, agents, and skills are available in every project. No project files are modified.

---

#### This project only: local, not committed

```bash
# 1. Register the marketplace (one time)
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin

# 2. Install scoped to the current project (not committed to git)
claude plugin install ayrshare@ayrshare --scope local

# 3. Enter your API key. Run inside Claude Code:
/plugin configure ayrshare@ayrshare
```

Commands, agents, and skills only appear in this project.

---

#### This project: committed with the team

```bash
# 1. Register the marketplace (one time)
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin

# 2. Install scoped to the project (committed to git)
claude plugin install ayrshare@ayrshare --scope project

# 3. Each developer enters their own API key. Run inside Claude Code:
/plugin configure ayrshare@ayrshare
```

The plugin is committed to the repo so the whole team gets it automatically. The repo holds no key: each developer enters their own, and it stays in that developer's secure credential store.

---

### MCP only (no plugin)

If you only want the raw MCP tools without commands, agents, or skills:

```bash
claude mcp add ayrshare --transport http \
  https://api.ayrshare.com/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

Add `--scope local` to limit it to the current project. The key is stored permanently in Claude Code's config — run once and you are done.

---

### Installing without a dialog (CI / headless)

`claude plugin install` takes each setting as `--config KEY=VALUE`:

```bash
claude plugin marketplace add ayrshare/ayrshare-social-media-api-claude-plugin
claude plugin install ayrshare@ayrshare --config api_key=YOUR_API_KEY
```

The keys are `api_key`, `profile_key`, `x_api_key` and `x_api_secret`. A value typed on the command line can end up in shell history, so on a shared or CI machine fill it in from your CI system's secret store. On Claude Code 2.1.285 or later you can instead pipe a JSON object of the values into `claude plugin configure ayrshare@ayrshare --values-stdin`.

---

## Credentials by surface

The plugin has one API key setting (`api_key`, required) and three optional ones (`profile_key`, `x_api_key`, `x_api_secret`). Where you can enter them depends on the surface, and the fully supported surface today is **Claude Code**:

| Surface | How credentials are supplied | Profile-Key | X BYOK |
|---|---|---|---|
| **Claude Code** (CLI / IDE / desktop "Code" tab) | The plugin's settings dialog: `/plugin configure ayrshare@ayrshare` | `profile_key` setting, or per-call `profileKey` argument | `x_api_key` + `x_api_secret` settings |
| **Cowork / claude.ai desktop app** | **Not supported yet.** The skills install, but Cowork does not prompt for plugin settings, so the plugin's MCP server does not load there. Needs OAuth (see [Roadmap](#roadmap)). | not available yet | not available yet |
| **CI / headless** | `--config` on `claude plugin install`, or `claude plugin configure --values-stdin` | `profile_key` | `x_api_key` + `x_api_secret` |

## Cowork and the desktop app (not supported yet)

The plugin's skills install in Cowork and the claude.ai desktop app, but its API tools **cannot authenticate** there yet, so you cannot actually use them. This is a platform limitation, not a setting you can change today:

- Cowork does not prompt for a plugin's settings, and it skips an MCP server that needs a setting with no default. The API key has no default, so the plugin's server does not load in Cowork.
- Adding Ayrshare as a **claude.ai / org connector** loads the tools but offers no field for an `Authorization` header, so every call returns `403` (`code 102`).
- A Claude Code **`managed-mcp.json`** (an admin/enterprise managed-MCP config file; see the [managed MCP docs](https://code.claude.com/docs/en/managed-mcp.md)) does **not** apply to Cowork; it is a Claude Code feature. Deploying one will not help Cowork, and it takes exclusive, machine-wide control of MCP that suppresses this plugin's own server and your other MCP servers **in Claude Code**. Do not use it for Cowork.

The fix is per-user OAuth on the MCP endpoint (see [Roadmap](#roadmap)). Until it ships, use the plugin in **Claude Code** (CLI, IDE, or the desktop **Code** tab).

---

## Commands

| Command | Description |
|---|---|
| `/ayrshare:setup` | Configure or rotate your API key |

The former `/ayrshare:post`, `/ayrshare:analytics`, and `/ayrshare:profiles` commands were consolidated into the **social-manager** and **profile-manager** agents and the trigger-based skills below.

---

## Skills

Trigger-based skills activate automatically on intent (even when you don't name Ayrshare) and teach Claude how to drive the MCP tools correctly: the auth model (API key, plus optional profile selection by `Profile-Key` header or per-call `profileKey` argument), retry safety, and platform quirks. They cover 27 of the server's 33 tools. The six ads tools (`list_ad_accounts`, `list_ads`, `get_ad_history`, `search_ad_targeting`, `update_ad`, `boost_post`) do not have a skill yet. **Start with `getting-started`**; every group skill cross-links to it.

| Skill | Tools | Use when |
|---|---|---|
| `getting-started` | (cross-cutting) | Installing/configuring the plugin, any 401/403, "which credential do I use", onboarding a client, or a missing API key. Home of the auth model (API key, plus `Profile-Key` header or per-call `profileKey` argument), onboarding sequence, retry/`explain_error` rules, and the free-trial link. |
| `post` | `create_post`, `validate_post`, `get_post`, `update_post`, `retry_post` | Validating, posting, scheduling, editing, fetching, or retrying social content. |
| `history` | `get_post_history`, `get_platform_history` | Listing posts sent via Ayrshare, or native posts (incl. ones not made via Ayrshare) and finding native Social Post IDs. |
| `analytics` | `get_post_analytics`, `get_post_analytics_by_social_id`, `get_social_network_analytics` | Per-post analytics (by Ayrshare or native Social Post ID) or account/network analytics (followers, reach, impressions). |
| `comments` | `get_comments`, `add_comment`, `reply_comment` | Reading, adding, or replying to comments on a post. |
| `messages` | `get_messages`, `send_message`, `get_auto_response`, `set_auto_response` | Reading/sending direct messages (Facebook, Instagram, X) or configuring the DM auto-responder. |
| `profiles` | `list_profiles`, `create_profile`, `generate_jwt_social_linking_url` | Creating or listing client profiles (account-level), or minting a client's social-account linking URL for the target profile (set by the `profileKey` argument or `Profile-Key` header; Business/Enterprise plan). |
| `media` | `validate_media` | Checking a media URL is reachable before posting (media is referenced by URL; there is no upload/library/resize tool). |
| `generate` | `generate_post`, `recommend_hashtags` | Drafting AI post copy (never publishes) or suggesting hashtags for a keyword (TikTok-sourced; uses the targeted profile's linked TikTok account). |
| `webhooks` | `register_webhook`, `unregister_webhook`, `list_webhooks` | Subscribing to push notifications (e.g. when a scheduled post publishes) instead of polling. |
| `errors` | `explain_error` | Decoding an Ayrshare error code into a plain-English cause + fix. |
| `draft-in-brand-voice` | (workflow: `get_platform_history`/`get_post_history` → `get_post_analytics` → `generate_post` → `validate_post`) | Writing on-brand content by matching a profile's established voice from its post history; drafts only. |
| `plan-and-schedule-campaign` | (workflow: `validate_post` → `create_post` per post, with `scheduleDate`) | Planning and scheduling a multi-post, multi-platform campaign or content calendar, validating each post first. |

The last two are multi-step **workflow** skills: they orchestrate the tools above rather than adding new ones. Skills and agents refer to tools by their short `mcp__ayrshare__<action>` name (e.g. `mcp__ayrshare__create_post`). When the server comes from this plugin, Claude Code names the same tool `mcp__plugin_ayrshare_ayrshare__<action>`.

---

## Optional Configuration

The API key is the plugin's `api_key` setting, which the `ayrshare` server's `.mcp.json` puts into its `Authorization` header (`Bearer ${user_config.api_key}`). Profile scoping has two equivalent inputs: a `Profile-Key` connection header (the default for every call) **or** an optional `profileKey` tool argument on a profile-scoped tool call (the argument wins when both are set; a few utility/AI tools such as `generate_post` and `recommend_hashtags` are excluded, see `getting-started`). The per-call argument lets an agent act as a client it learns at runtime without changing settings or restarting.

The optional `Profile-Key` and X BYOK headers come from settings that default to empty, so you turn each one on by filling it in with `/plugin configure ayrshare@ayrshare`. Leave a setting empty and its header goes out empty, which the server treats as not provided. No `.mcp.json` editing, and nothing to redo after `claude plugin update`.

### Act as a specific client profile (`profile_key` setting or `profileKey` argument)

Fill in the **Profile-Key** setting to default the whole connection to one client; the bundled `Profile-Key: ${user_config.profile_key}` header carries it. With it empty, calls act on the account's primary profile. To act as a client for a single call instead, pass `profileKey` as a tool argument on a profile-scoped call (it takes precedence over the header, no restart needed). The `generate_jwt_social_linking_url` tool uses the same target-profile value (required for that tool, via either input). You do not pass a private key or domain (the server derives them from your authenticated account), but the account must still have a **provisioned social-linking domain (Business/Enterprise)**; without one the call returns a "No social-linking domain is provisioned for this account" error.

### Bring-your-own X/Twitter app (BYOK)

Posting to X/Twitter requires your own X Developer App (the [X BYO-key mandate](https://www.ayrshare.com/docs/apis/overview#x/twitter-byo-credentials), effective March 31, 2026). After linking X with your app's credentials, fill in **both** settings with `/plugin configure ayrshare@ayrshare` (values are never logged):

- **X API Key** (`x_api_key`): your X API Key (Consumer Key); sent as the `X-Twitter-OAuth1-Api-Key` header.
- **X API Secret** (`x_api_secret`): your X API Secret (Consumer Secret); sent as the `X-Twitter-OAuth1-Api-Secret` header.

These are the only X BYO headers Ayrshare uses: one key pair per Ayrshare account, sent on every X-targeting request (the same pair for all sub-profiles). Ayrshare does **not** use OAuth 2.0 client credentials here. Set both or neither: with neither set, an X/Twitter request returns error `419` (`x_credentials_required`); with only one set, it returns error `400`.

---

## Supported Platforms

Facebook, Instagram, LinkedIn, X (Twitter), TikTok, YouTube, Pinterest, Reddit, Telegram, Threads, Bluesky, Snapchat, and Google Business Profile — depending on your Ayrshare plan and connected profiles.

---

## Roadmap

- **Zero-config per-user auth (MCP OAuth 2.1): our top auth priority.** This is the only path that makes Cowork and the claude.ai desktop app first-class surfaces, so it is the next major auth investment for this plugin. We plan to add MCP-spec OAuth 2.1 (PKCE, dynamic client registration, authorization-server and protected-resource metadata) to `api.ayrshare.com/mcp`. Once shipped, an org-installed plugin authenticates per user on the first tool call ("Connect to Ayrshare", a dashboard login, a per-user token) on **every** surface, with no manual key step and no files. Claude Code already supports this flow via `/mcp` authenticate, so setup collapses to "install, then click Connect" everywhere. Until it lands, **Claude Code** is the supported surface (via the plugin's settings dialog), and **Cowork and the claude.ai desktop app are not supported**, because the plugin's bearer-header MCP server cannot authenticate there.

---

## Privacy

The plugin sends your requests, and the API key and optional settings above, only to the Ayrshare API at `https://api.ayrshare.com/mcp`. See the [Ayrshare privacy policy](https://www.ayrshare.com/privacy/).

---

## Resources

- [Ayrshare API Docs](https://docs.ayrshare.com)
- [Ayrshare Dashboard](https://app.ayrshare.com)
- [Claude Code Docs](https://docs.anthropic.com/claude-code)
